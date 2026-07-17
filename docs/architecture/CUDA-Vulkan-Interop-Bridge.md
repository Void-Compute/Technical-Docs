# Ghost CUDA-Vulkan Interop Bridge — Technical Report

This report details the design, implementation, and operational behavior of the Ghost CUDA-Vulkan Interop Bridge, which enables Omniverse (Isaac Sim) to share GPU render buffers between the AMD Vulkan graphics pipeline and the HIP compute pipeline via the CUDA external memory/semaphore API surface.

## 1. The Problem

Omniverse renders frames in Vulkan and shares render buffers with CUDA for physics/compute (Warp kernels, PhysX). The data flow on a real NVIDIA system is:

```
Vulkan (NVIDIA driver) → vkGetMemoryWin32HandleKHR → Win32 HANDLE → cuImportExternalMemory (CUDA) → GPU compute
```

On our AMD system with ZLUDA, this flow breaks at the CUDA side. ZLUDA's `nvcuda.dll` declares all 7 external memory/semaphore functions in its symbol dispatch table (`process_table.rs:4388`), but **none of them have implementation bodies**. They are auto-generated stubs that fall through to `mod.rs:26` and return `CUDA_ERROR_NOT_SUPPORTED` (801).

When `carb.cudainterop.plugin.dll` calls `cuImportExternalMemory`, ZLUDA returns 801, and Omniverse logs:

*   `"Failed to import external memory in CUDA"`
*   `"Failed to import external semaphore in CUDA"`

Physics simulation, Warp kernels, and any compute that depends on shared render buffers will fail silently or crash.

## 2. The Solution Architecture

Ghost solves this with a **dual-layer interop bridge** that intercepts CUDA calls before they reach ZLUDA and redirects them to the equivalent HIP functions in AMD's `amdhip64_7.dll`.

### 2.1 Option A — void_shim.dll (Primary Path)

`void_shim.dll` is deployed as `nvcuda.dll` and contains the `cuGetProcAddress` master router. When Omniverse requests any of the 8 interop functions via `cuGetProcAddress("cuImportExternalMemory")`, the router returns a pointer to our local implementation **before** ZLUDA's dispatch table is ever consulted.

The router priority chain is:

```
cuGetProcAddress("cuImportExternalMemory")
  → 0. Self-reference check
  → 1. Identity spoofs (cuDeviceGetName, etc.)
  → 2. WELD OVERRIDES ← Our interop lives here
  → 3. ZLUDA forward (skipped for interop)
  → 4. NOP fallback
```

Because the interop functions are in category 2, they are always resolved locally and never forwarded to ZLUDA.

### 2.2 Option B — ghost_cuda_interop.dll (Fallback Path)

A standalone DLL compiled separately from `void_shim.dll`. It contains identical implementations of all 8 bridge functions plus a lookup export:

*   `Ghost_GetInteropFunction(const char* name)` — returns a function pointer for any of the 8 interop symbols.

This DLL exists as a fallback for environments where `void_shim.dll` is not loaded as `nvcuda.dll` (e.g., if a future deployment strategy changes the DLL injection order). It dynamically loads `amdhip64_7.dll` on first call and resolves all HIP functions via `GetProcAddress`.

### 2.3 The Complete Data Flow (After Ghost)

```
Vulkan (AMD driver)
  → VK_KHR_external_memory_win32 (AMD natively supports this)
  → vkGetMemoryWin32HandleKHR
  → Win32 HANDLE (opaque, valid on AMD)
  → cuImportExternalMemory (Ghost void_shim.dll)
  → hipImportExternalMemory (amdhip64_7.dll)
  → AMD compute pipeline
```

## 3. Bridged Functions — Full Inventory

All 8 CUDA functions are implemented as direct translations to HIP. The CUDA and HIP descriptor structs have **identical memory layouts by design** (AMD intentionally mirrors NVIDIA's struct definitions), so no field-by-field marshaling is needed — we cast and forward.

### 3.1 cuImportExternalMemory

*   **CUDA signature:** `CUresult cuImportExternalMemory(CUexternalMemory* extMem_out, const CUDA_EXTERNAL_MEMORY_HANDLE_DESC* memHandleDesc)`
*   **HIP target:** `hipImportExternalMemory`
*   **Handle type mapping:**
    *   `CU_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_WIN32` (2) → `hipExternalMemoryHandleTypeOpaqueWin32` (2) — values are identical
    *   `CU_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_WIN32_KMT` (3) → `hipExternalMemoryHandleTypeOpaqueWin32Kmt` (3) — values are identical
*   **Fields forwarded:** `handle.win32.handle`, `handle.win32.name`, `size`, `flags`
*   **Trace output:** `cuImportExternalMemory: HIP returned %d, extMem=%p`

This is the most critical function. If it fails, no render buffers can be shared with compute.

### 3.2 cuExternalMemoryGetMappedBuffer

*   **CUDA signature:** `CUresult cuExternalMemoryGetMappedBuffer(CUdeviceptr* devPtr, CUexternalMemory extMem, const CUDA_EXTERNAL_MEMORY_BUFFER_DESC* bufferDesc)`
*   **HIP target:** `hipExternalMemoryGetMappedBuffer`
*   **Fields forwarded:** `offset`, `size`, `flags`
*   **Note:** HIP returns a `void*`, which we cast to `CUdeviceptr` (`unsigned long long`). The pointer value is a GPU virtual address valid on the AMD device.
*   **Trace output:** `cuExternalMemoryGetMappedBuffer: HIP returned %d, devPtr=%p`

### 3.3 cuExternalMemoryGetMappedMipmappedArray

*   **CUDA signature:** `CUresult cuExternalMemoryGetMappedMipmappedArray(CUmipmappedArray* mipmap, CUexternalMemory extMem, const CUDA_EXTERNAL_MEMORY_MIPMAPPED_ARRAY_DESC* mipmapDesc)`
*   **HIP target:** `hipExternalMemoryGetMappedMipmappedArray`
*   **Note:** This function was **completely missing** from the previous void_shim implementation. It has been added new. The `CUDA_ARRAY3D_DESCRIPTOR` embedded in the desc struct is 56 bytes and passed through as an opaque blob.
*   **Trace output:** `cuExternalMemoryGetMappedMipmappedArray: HIP returned %d`

### 3.4 cuDestroyExternalMemory

*   **CUDA signature:** `CUresult cuDestroyExternalMemory(CUexternalMemory extMem)`
*   **HIP target:** `hipDestroyExternalMemory`
*   **Note:** Direct passthrough, no struct translation needed.

### 3.5 cuImportExternalSemaphore

*   **CUDA signature:** `CUresult cuImportExternalSemaphore(CUexternalSemaphore* extSem_out, const CUDA_EXTERNAL_SEMAPHORE_HANDLE_DESC* semHandleDesc)`
*   **HIP target:** `hipImportExternalSemaphore`
*   **Handle type mapping:**
    *   `CU_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_WIN32` (2) → identical HIP value
    *   `CU_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_WIN32_KMT` (3) → identical HIP value
    *   `CU_EXTERNAL_SEMAPHORE_HANDLE_TYPE_TIMELINE_SEMAPHORE_WIN32` (10) → identical HIP value
*   **Fields forwarded:** `handle.win32.handle`, `handle.win32.name`, `flags`
*   **Brute-force fallback:** If `hipImportExternalSemaphore` returns a non-zero error, the system:
    1.  Logs `"WARNING: hipImportExternalSemaphore FAILED - activating brute-force sync (hipStreamSynchronize) fallback!"`
    2.  Sets the global `g_SemaphoreBruteForce` flag to 1
    3.  Writes a dummy handle (`0xDEADBEEF`) to `extSem_out`
    4.  Returns `CUDA_SUCCESS` (0) — the caller thinks the import succeeded
    5.  All subsequent signal/wait calls will use `hipStreamSynchronize` instead
*   **Trace output:** `cuImportExternalSemaphore: HIP returned %d, extSem=%p`

### 3.6 cuSignalExternalSemaphoresAsync

*   **CUDA signature:** `CUresult cuSignalExternalSemaphoresAsync(const CUexternalSemaphore* extSemArray, const CUDA_EXTERNAL_SEMAPHORE_SIGNAL_PARAMS* paramsArray, unsigned int numExtSems, CUstream stream)`
*   **HIP target:** `hipSignalExternalSemaphoresAsync`
*   **Brute-force path:** When `g_SemaphoreBruteForce == 1`, calls `hipStreamSynchronize(stream)` instead. This is a full pipeline flush — it blocks until all prior GPU work on the stream completes, effectively acting as a fence. Performance is degraded but correctness is guaranteed.
*   **Trace output:** `cuSignalExternalSemaphoresAsync: count=%u stream=%p bruteForce=%d`

### 3.7 cuWaitExternalSemaphoresAsync

*   **CUDA signature:** `CUresult cuWaitExternalSemaphoresAsync(const CUexternalSemaphore* extSemArray, const CUDA_EXTERNAL_SEMAPHORE_WAIT_PARAMS* paramsArray, unsigned int numExtSems, CUstream stream)`
*   **HIP target:** `hipWaitExternalSemaphoresAsync`
*   **Brute-force path:** Same as signal — `hipStreamSynchronize(stream)`. The stream blocks until all prior work completes, then proceeds. This means Vulkan and HIP are fully serialized (no overlap), but no data races occur.
*   **Trace output:** `cuWaitExternalSemaphoresAsync: count=%u stream=%p bruteForce=%d`

### 3.8 cuDestroyExternalSemaphore

*   **CUDA signature:** `CUresult cuDestroyExternalSemaphore(CUexternalSemaphore extSem)`
*   **HIP target:** `hipDestroyExternalSemaphore`
*   **Brute-force path:** When `g_SemaphoreBruteForce == 1`, the handle is the dummy `0xDEADBEEF`, so we return `CUDA_SUCCESS` without calling HIP.

## 4. Vulkan-Side DCC/Tiling Safety (Phase 2)

Even if the CUDA-HIP bridge works perfectly, AMD's Vulkan driver can produce memory layouts that HIP cannot read. Two hardware features cause this:

### 4.1 Delta Color Compression (DCC)

AMD GPUs apply DCC transparently to render targets to reduce bandwidth. DCC-compressed surfaces have a non-standard memory layout that is invisible to HIP — HIP sees scrambled data.

**Mitigation:** In `Hook_vkAllocateMemory`, when the pNext chain contains `VkExportMemoryAllocateInfo` (meaning this memory will be shared with CUDA), we inject a `VkMemoryDedicatedAllocateInfo` struct if one is not already present. Dedicated allocations **disable DCC** on AMD's Vulkan driver.

*   **Detection:** Two-pass scan of the pNext chain — first pass detects `hasExport` and `hasDedicated` flags, then we inject only if `hasExport && !hasDedicated`.
*   **Struct injected:** `VkMemoryDedicatedAllocateInfo { sType=1000127001, pNext=NULL, image=0, buffer=0 }`
*   **Trace output:** `"vkAllocateMemory: Injecting VkMemoryDedicatedAllocateInfo to disable DCC for export"`

### 4.2 Hardware Tiling / Swizzle

AMD GPUs use proprietary tile swizzle patterns (`DISPLAY_MICRO_TILING`, `THIN_MICRO_TILING`, etc.) for optimal rendering. These patterns rearrange pixels in memory for cache efficiency. HIP's external memory import expects linear (row-major) data.

**Mitigation:** In `Hook_vkCreateImage`, when the image has `VkExternalMemoryImageCreateInfo` in its pNext chain, we override `tiling` from `VK_IMAGE_TILING_OPTIMAL` (0) to `VK_IMAGE_TILING_LINEAR` (1).

*   **Detection:** Walk pNext chain looking for `sType == 1000072001` (`VK_STRUCTURE_TYPE_EXTERNAL_MEMORY_IMAGE_CREATE_INFO`).
*   **Override:** `imageInfo->tiling = 1` (mutates the struct in-place before forwarding to the real driver).
*   **Trace output:** `"vkCreateImage: OVERRIDE tiling OPTIMAL->LINEAR for external memory image (DCC safety)"`
*   **Fallback strategy:** If LINEAR tiling causes allocation failures (some AMD GPUs reject LINEAR for certain formats/sizes), the fallback is to keep OPTIMAL and insert `VK_IMAGE_LAYOUT_GENERAL` transitions before handle export.

### 4.3 External Buffer Capability Patching

`Hook_vkGetPhysicalDeviceExternalBufferProperties` intercepts Omniverse's capability query for external buffer sharing support. AMD's driver may report conservative capabilities that cause Omniverse to skip interop entirely.

**Mitigation:** After calling the real driver, we patch the returned `VkExternalMemoryProperties.externalMemoryFeatures` to include:

*   `VK_EXTERNAL_MEMORY_FEATURE_EXPORTABLE_BIT` (0x1)
*   `VK_EXTERNAL_MEMORY_FEATURE_IMPORTABLE_BIT` (0x2)

*   **Detection:** Compare original and patched values.
*   **Trace output:** `"vkGetPhysicalDeviceExternalBufferProperties: PATCHED features 0x%X -> 0x%X (added EXPORTABLE|IMPORTABLE)"`
*   **Registered for both:** `vkGetPhysicalDeviceExternalBufferProperties` (core 1.1) and `vkGetPhysicalDeviceExternalBufferPropertiesKHR` (extension).

## 5. Vulkan-Side Semaphore Synchronization (Phase 3)

Omniverse uses Vulkan semaphores shared with CUDA to synchronize render-compute ordering. The Vulkan side of this is handled by three trace hooks:

### 5.1 Hook_vkCreateSemaphore

Scans the pNext chain for `VkExportSemaphoreCreateInfo` (`sType == 1000077000`). When found, logs the `handleTypes` bitmask to identify which semaphore sharing mode Omniverse is requesting.

*   **Expected value:** `handleTypes = 0x2` (`VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_WIN32_BIT`)
*   **Trace output:** `"vkCreateSemaphore: EXPORT semaphore detected! handleTypes=0x%X (OpaqueWin32=0x2)"`
*   **Purpose:** Diagnostic only — the real driver handles creation. This tells us whether Omniverse is actually attempting cross-API semaphore sharing.

### 5.2 Hook_vkGetSemaphoreWin32HandleKHR

Trace-only hook that logs every semaphore handle export. Resolves the real function from the driver via `g_NextGetDeviceProcAddr` and forwards the call.

*   **Trace output:** `"vkGetSemaphoreWin32HandleKHR: returned %d, handle=%p"`
*   **Purpose:** Captures the Win32 HANDLE value that will later be passed to `cuImportExternalSemaphore` on the CUDA side. Cross-referencing these values in `ghost_trace.log` confirms the handle chain is intact.

### 5.3 Hook_vkImportSemaphoreWin32HandleKHR

Trace-only hook that logs every semaphore handle import. Same pattern — resolve real function, forward, log result.

*   **Trace output:** `"vkImportSemaphoreWin32HandleKHR: returned %d"`

## 6. ghost_cuda_interop.dll — Standalone Bridge

The standalone DLL (`ghost_cuda_interop.dll`) is generated and compiled by the Rust orchestrator at build time, following the same pattern as `void_shim.dll` and `ghost_vulkan_layer.dll`.

### 6.1 Build Process

1.  Rust orchestrator writes `ghost_cuda_interop.cpp` to `nv_spoof/`
2.  Locates MSVC via `vswhere.exe` → `vcvars64.bat`
3.  Compiles with: `cl.exe /LD /MT /O2 ghost_cuda_interop.cpp /Fe:ghost_cuda_interop.dll /link /MACHINE:X64 kernel32.lib user32.lib advapi32.lib`
4.  Fallback: direct `cl.exe` invocation if `vswhere` is unavailable

### 6.2 HIP Resolution

On first call to any interop function, `ensure_hip()` is called:

1.  Attempts `LoadLibraryA("amdhip64_7.dll")`
2.  Falls back to `LoadLibraryA("amdhip64.dll")`
3.  Resolves all 9 function pointers via `GetProcAddress`:
    *   `hipImportExternalMemory`
    *   `hipExternalMemoryGetMappedBuffer`
    *   `hipExternalMemoryGetMappedMipmappedArray`
    *   `hipDestroyExternalMemory`
    *   `hipImportExternalSemaphore`
    *   `hipSignalExternalSemaphoresAsync`
    *   `hipWaitExternalSemaphoresAsync`
    *   `hipDestroyExternalSemaphore`
    *   `hipStreamSynchronize` (for brute-force fallback)
4.  Logs all resolved pointers for debugging: `"HIP functions resolved: mem=%p buf=%p mip=%p ..."`

### 6.3 Exports

*   `Ghost_cuImportExternalMemory` through `Ghost_cuDestroyExternalSemaphore` — all 8 bridge functions
*   `Ghost_GetInteropFunction(const char* name)` — lookup by CUDA symbol name, returns function pointer or NULL

### 6.4 Deployment

`ghost_cuda_interop.dll` is deployed to `C:\Windows\System32` alongside all other Ghost stubs (`nvcuda.dll`, `nvml.dll`, `nvapi64.dll`, etc.). This is required because Isaac Sim uses `LOAD_LIBRARY_SEARCH_SYSTEM32` which restricts DLL loading to System32 only.

## 7. Error Translation

All HIP error codes are translated to CUDA error codes via `hip_to_cuda()`:

| HIP Return | CUDA Return | Meaning |
|------------|-------------|---------|
| `hipSuccess` (0) | `CUDA_SUCCESS` (0) | Operation succeeded |
| Any non-zero | `CUDA_ERROR_OPERATING_SYSTEM` (304) | Generic failure |

The simplistic translation is intentional — Omniverse checks for `!= CUDA_SUCCESS` and logs its own error messages. The specific error code is not parsed by the caller.

## 8. The Semaphore Brute-Force Fallback

AMD's HIP documentation states that external semaphores are "not supported on Linux" but is ambiguous about Windows (WDDM). If `hipImportExternalSemaphore` fails at runtime, the interop bridge activates the brute-force synchronization path automatically.

### 8.1 How It Works

```
Normal path:           Vulkan signal → Win32 HANDLE → cuSignalExternalSemaphoresAsync → hipSignalExternalSemaphoresAsync
                       cuWaitExternalSemaphoresAsync → hipWaitExternalSemaphoresAsync → Vulkan wait

Brute-force path:     Vulkan signal → Win32 HANDLE → cuSignalExternalSemaphoresAsync → hipStreamSynchronize(stream)
                       cuWaitExternalSemaphoresAsync → hipStreamSynchronize(stream) → Vulkan wait
```

### 8.2 Performance Impact

The brute-force path serializes the Vulkan render and HIP compute pipelines. On a normal system, signal/wait allows overlap — the GPU can render frame N+1 while computing on frame N. With brute-force sync, each stage must complete before the next begins. This effectively halves GPU utilization for interop workloads.

### 8.3 Detection in Logs

When brute-force activates, `ghost_trace.log` will contain:

```
[CUDA-INTEROP] WARNING: hipImportExternalSemaphore FAILED - activating brute-force sync (hipStreamSynchronize) fallback!
[CUDA-INTEROP] cuSignalExternalSemaphoresAsync[BRUTE_FORCE]
[CUDA-INTEROP] cuWaitExternalSemaphoresAsync[BRUTE_FORCE]
```

## 9. Diagnostic Checklist

When debugging interop issues at runtime, check `ghost_trace.log` for:

| Log Entry | Meaning |
|-----------|---------|
| `cuImportExternalMemory: HIP returned 0` | ✅ Memory import succeeded |
| `cuImportExternalMemory: HIP returned [non-zero]` | ❌ AMD driver rejected the handle — check handle type and size |
| `cuExternalMemoryGetMappedBuffer: devPtr=%p` | ✅ Buffer mapping succeeded, valid GPU pointer |
| `hipImportExternalSemaphore FAILED` | ⚠️ Brute-force fallback active — performance degraded but functional |
| `vkAllocateMemory: Injecting VkMemoryDedicatedAllocateInfo` | ℹ️ DCC disabled for this allocation |
| `vkCreateImage: OVERRIDE tiling OPTIMAL->LINEAR` | ℹ️ Tiling forced to linear for cross-API safety |
| `vkGetPhysicalDeviceExternalBufferProperties: PATCHED features` | ℹ️ Capability flags augmented for Omniverse compatibility |
| `vkCreateSemaphore: EXPORT semaphore detected` | ℹ️ Omniverse is setting up cross-API semaphore sharing |
| `vkGetSemaphoreWin32HandleKHR: handle=%p` | ℹ️ Win32 handle exported — cross-reference with cuImportExternalSemaphore |

## 10. Known Risks and Limitations

1.  **Struct layout assumption:** We assume CUDA and HIP external memory/semaphore descriptor structs have identical binary layouts. This is true for HIP 6.x/7.x but could break if AMD changes the struct layout in a future ROCm release.

2.  **LINEAR tiling rejection:** Some AMD GPUs may reject `VK_IMAGE_TILING_LINEAR` for certain image formats (compressed formats like BC/ASTC, or unusual sample counts). If `vkCreateImage` returns `VK_ERROR_FORMAT_NOT_SUPPORTED` after our tiling override, the fallback is to revert to OPTIMAL with layout transitions.

3.  **DCC on dedicated allocations:** Injecting `VkMemoryDedicatedAllocateInfo` with `image=0` and `buffer=0` means we are requesting a dedicated allocation without binding it to a specific resource. Some drivers may ignore this or still apply DCC. If corruption persists, we may need to capture the specific `VkImage`/`VkBuffer` handle and pass it to the dedicated allocation.

4.  **Timeline semaphores:** If Omniverse uses `VK_SEMAPHORE_TYPE_TIMELINE` for external sharing, the brute-force fallback still works (it ignores the semaphore entirely and syncs the stream), but the timeline value tracking is lost. Omniverse may log warnings about timeline values being out of order.

5.  **Multi-device (P2P):** The current implementation assumes a single GPU. If the system has multiple AMD GPUs and Omniverse attempts P2P memory sharing, `hipImportExternalMemory` may fail because the Win32 handle belongs to device 0 but is being imported on device 1.

## Summary

The Ghost CUDA-Vulkan Interop Bridge implements functionality currently unimplemented in the preview version of ZLUDA. - 7 unimplemented CUDA external memory/semaphore functions — by routing them directly to HIP via `amdhip64_7.dll`. The Vulkan layer adds three safety measures (DCC disable, LINEAR tiling, capability patching) to ensure the shared memory is in a format that both Vulkan and HIP can read. The semaphore brute-force fallback guarantees correctness even if AMD's HIP doesn't support external semaphores on Windows. All of this is transparent to Omniverse — it calls standard CUDA functions and receives standard CUDA responses.
