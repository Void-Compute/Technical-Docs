# CUDA-Vulkan Interop Implementation Plan

## Problem Statement

When Omniverse renders a frame in Vulkan, it shares that render buffer with CUDA for physics/compute via `cuImportExternalMemory`. The data flow is:

```
Vulkan (AMD driver) → vkGetMemoryWin32HandleKHR → Win32 HANDLE → cuImportExternalMemory (ZLUDA) → hipImportExternalMemory (HIP) → AMD compute
```

**The gap:** ZLUDA's `cuImportExternalMemory` is **auto-generated as an unimplemented stub** that returns `CUDA_ERROR_NOT_SUPPORTED`. When Omniverse's `carb.cudainterop.plugin.dll` calls it, it will fail with `"Failed to import external memory in CUDA"`.

## Research Findings

### ZLUDA Source Analysis (E:\ghost_project\nvidia source\ZLUDA-7-preview.3)

| Component | Status |
|-----------|--------|
| `cuImportExternalMemory` | ❌ **Unimplemented** — declared in `cuda_macros/src/cuda.rs:10963`, dispatch table in `process_table.rs:4388`, but no implementation body anywhere in `zluda/src/impl/`. Falls through to `mod.rs:26` → `CUerror::NOT_SUPPORTED` |
| `cuImportExternalSemaphore` | ❌ **Unimplemented** — same pattern |
| `cuExternalMemoryGetMappedBuffer` | ❌ **Unimplemented** |
| `cuSignalExternalSemaphoresAsync` | ❌ **Unimplemented** |
| `cuWaitExternalSemaphoresAsync` | ❌ **Unimplemented** |
| `cuDestroyExternalMemory` | ❌ **Unimplemented** |
| `cuDestroyExternalSemaphore` | ❌ **Unimplemented** |
| HIP FFI bindings (`hipImportExternalMemory` etc.) | ✅ **Present** in `ext/hip_runtime-sys/src/lib.rs:6580` — links to `amdhip64_7.dll` |

### AMD Driver Capabilities (Verified via vulkaninfo)

| Extension | Supported |
|-----------|-----------|
| `VK_KHR_external_memory` | ✅ rev 1 |
| `VK_KHR_external_memory_win32` | ✅ rev 1 |
| `VK_KHR_external_semaphore` | ✅ rev 1 |
| `VK_KHR_external_semaphore_win32` | ✅ rev 1 |
| `VK_KHR_timeline_semaphore` | ✅ rev 2 |
| `VK_KHR_dedicated_allocation` | ✅ rev 3 |
| `VK_EXT_external_memory_host` | ✅ rev 1 |

### HIP External Memory Status (Web Research)

- `hipImportExternalMemory` **exists in the API** and links against `amdhip64_7.dll`
- `hipExternalMemoryHandleTypeOpaqueWin32` is defined (value matches CUDA's)
- On **Windows**, the API is evolving but functional for basic opaque Win32 handles
- External semaphores are **not supported on Linux** but **may work on Windows** (WDDM)
- **DCC (Delta Color Compression)** and tiling are the main risk — AMD's Vulkan driver may produce compressed surfaces that HIP cannot read directly

### Ghost Vulkan Layer Status (Already Implemented)

| Component | Status |
|-----------|--------|
| NV→KHR extension injection/filtering | ✅ `g_InjectedExtensions[]` includes `VK_NV_external_memory*` |
| `vkAllocateMemory` NV→KHR translation | ✅ `Hook_vkAllocateMemory` rewrites `VkExportMemoryAllocateInfoNV` → `VkExportMemoryAllocateInfo` |
| `vkGetMemoryWin32HandleNV` → KHR shim | ✅ `Hook_vkGetMemoryWin32HandleNV` delegates to `vkGetMemoryWin32HandleKHR` |
| NV pNext chain stripping in `vkCreateDevice` | ✅ Strips `VkDeviceDiagnosticsConfigCreateInfoNV` etc. |
| NV stub safety net | ✅ `nv_nop_query_stub` for unrecognized NV functions |

> [!IMPORTANT]
> The Vulkan side is already handled by our ghost_vulkan_layer. The Win32 HANDLE will be created correctly by the AMD Vulkan driver via the KHR path. The missing piece is entirely on the **CUDA side** — ZLUDA needs to translate those handles into HIP.

## Proposed Changes

### Phase 1: CUDA External Memory/Semaphore Implementation

We need to add the actual implementation for 7 CUDA functions inside ZLUDA. Since we cannot recompile ZLUDA itself (it requires the full ROCm toolchain), we will implement these as **interceptors inside `void_shim.dll`** or as a **standalone `ghost_cuda_interop.dll`** that hooks the ZLUDA dispatch table at runtime.

---

#### [NEW] ghost_cuda_interop.cpp

A new C++ source file (compiled alongside `void_shim.dll` or as a separate DLL) that implements the 7 missing CUDA external memory/semaphore functions by directly calling the HIP equivalents from `amdhip64_7.dll`.

**Functions to implement:**

1. **`cuImportExternalMemory`**
   - Translate `CUDA_EXTERNAL_MEMORY_HANDLE_DESC` → `hipExternalMemoryHandleDesc`
   - Map `CU_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_WIN32` (2) → `hipExternalMemoryHandleTypeOpaqueWin32` (2)
   - Map `CU_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_WIN32_KMT` (3) → `hipExternalMemoryHandleTypeOpaqueWin32Kmt` (3)
   - Copy `handle.win32.handle`, `handle.win32.name`, `size`, `flags`
   - Call `hipImportExternalMemory`
   - Translate `hipError_t` → `CUresult`

2. **`cuExternalMemoryGetMappedBuffer`**
   - Translate `CUDA_EXTERNAL_MEMORY_BUFFER_DESC` → `hipExternalMemoryBufferDesc`
   - Copy `offset`, `size`, `flags`
   - Call `hipExternalMemoryGetMappedBuffer`
   - Return device pointer

3. **`cuExternalMemoryGetMappedMipmappedArray`**
   - Translate `CUDA_EXTERNAL_MEMORY_MIPMAPPED_ARRAY_DESC` → `hipExternalMemoryMipmappedArrayDesc`
   - Call `hipExternalMemoryGetMappedMipmappedArray`

4. **`cuDestroyExternalMemory`**
   - Direct passthrough to `hipDestroyExternalMemory`

5. **`cuImportExternalSemaphore`**
   - Translate `CUDA_EXTERNAL_SEMAPHORE_HANDLE_DESC` → `hipExternalSemaphoreHandleDesc`
   - Map handle types (opaque Win32 → HIP equivalent)
   - Call `hipImportExternalSemaphore`

6. **`cuSignalExternalSemaphoresAsync` / `cuWaitExternalSemaphoresAsync`**
   - Translate semaphore params and stream handle
   - Call `hipSignalExternalSemaphoresAsync` / `hipWaitExternalSemaphoresAsync`

7. **`cuDestroyExternalSemaphore`**
   - Direct passthrough to `hipDestroyExternalSemaphore`

---

#### [MODIFY] [main.rs](file:///E:/ghost_project/src/main.rs)

The Rust orchestrator needs to:
1. Generate `ghost_cuda_interop.cpp` at build time (same pattern as `void_shim.cpp` and `ghost_vulkan_layer.cpp`)
2. Compile it with MSVC linking against `amdhip64_7.dll`
3. Inject the interop hooks into the ZLUDA dispatch using a **dual-layer strategy:**
   - **Option A (Primary):** Patching ZLUDA's `nvcuda.dll` dispatch table in-memory after it loads, replacing the 7 stub pointers with our implementations. Walk the export table of ZLUDA's `nvcuda.dll`, locate the internal dispatch entries for the 7 external memory/semaphore functions, and overwrite them with pointers to our `ghost_cuda_interop.dll` implementations.
   - **Option B (Fallback):** If Option A fails (dispatch table is obfuscated or relocated), hook `cuGetProcAddress` instead. When Omniverse calls `cuGetProcAddress("cuImportExternalMemory")`, our hook intercepts it and returns a pointer to our implementation instead of ZLUDA's stub. This acts as a safety net since `carb.cudainterop.plugin.dll` resolves CUDA functions dynamically.
   - **Boot sequence:** Option A is attempted first during DLL injection. If the dispatch table patch fails (detected by a verification call), Option B is activated automatically as fallback.

### Phase 2: Tiling/DCC Safety

#### [MODIFY] [ghost_vulkan_layer.cpp](file:///E:/ghost_project/target/release/.ghost/nv_spoof/ghost_vulkan_layer.cpp)

Add safety measures to prevent DCC/tiling corruption:

1. **In `Hook_vkAllocateMemory`:** When the pNext chain contains `VkExportMemoryAllocateInfo`, also inject `VkMemoryDedicatedAllocateInfoKHR` if not already present. Dedicated allocations disable DCC compression on AMD, ensuring clean linear memory for cross-API sharing.

2. **In `Hook_vkCreateImage`:** When the image has `VK_EXTERNAL_MEMORY_IMAGE_CREATE_INFO` in pNext, force `tiling = VK_IMAGE_TILING_LINEAR` if the original was `VK_IMAGE_TILING_OPTIMAL`. This prevents AMD's hardware tiling from producing swizzled data that HIP cannot interpret. Log a warning when this override occurs. **Decision: We start with the conservative LINEAR approach first.** If this causes allocation failures for large textures or unacceptable performance, we will fall back to keeping OPTIMAL tiling and instead inserting `VK_IMAGE_LAYOUT_GENERAL` transitions before handle export.

3. **Add `Hook_vkGetPhysicalDeviceExternalBufferProperties`:** Intercept this function to report that the AMD driver supports external buffer sharing with opaque Win32 handles, even if the real driver response is conservative.

### Phase 3: Semaphore Synchronization

#### [MODIFY] ghost_vulkan_layer.cpp

1. **In `Hook_vkCreateSemaphore`:** When the pNext chain contains `VkExportSemaphoreCreateInfo` with `VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_WIN32_BIT`, ensure the AMD driver receives it correctly (it should, since `VK_KHR_external_semaphore_win32` is supported).

2. **Add `Hook_vkGetSemaphoreWin32HandleKHR`:** Trace-only hook to log semaphore handle exports for debugging.

3. **Add `Hook_vkImportSemaphoreWin32HandleKHR`:** Trace-only hook.

## Resolved Design Decisions

> [!NOTE]
> **Dispatch patching: Dual-layer strategy (A + B).** Option A (in-memory dispatch table patching) is the primary method. Option B (hooking `cuGetProcAddress`) is deployed as an automatic fallback if A fails. Both are implemented; the boot sequence tries A first, verifies it worked, and falls back to B if needed.

> [!NOTE]
> **Tiling: LINEAR first.** We start with `VK_IMAGE_TILING_LINEAR` for all externally shared images. If this causes allocation failures or performance issues, we fall back to OPTIMAL with `VK_IMAGE_LAYOUT_GENERAL` transitions.

> [!WARNING]
> **HIP external semaphore stability on Windows:** AMD's documentation marks external semaphores as "not supported on Linux" but is ambiguous about Windows. If `hipImportExternalSemaphore` fails at runtime on Windows, the fallback is already planned: use `hipStreamSynchronize` / `vkQueueWaitIdle` as a brute-force synchronization alternative (trading performance for correctness). The `ghost_cuda_interop.cpp` implementation will detect the failure and automatically activate the brute-force path.

## Verification Plan

### Automated Tests
1. `cargo build --release` — verify compilation
2. Launch Isaac Sim with `GHOST_TRACE=1` — capture full trace log
3. Check `ghost_trace.log` for:
   - `cuImportExternalMemory` calls (should show NV→HIP translation)
   - `hipImportExternalMemory` return codes
   - Any `ACCESS_VIOLATION` crashes in the interop path

### Manual Verification
1. Monitor for `"Failed to import external memory in CUDA"` in Omniverse logs
2. If rendering starts, check for visual corruption (checkerboard = DCC issue, scrambled = tiling issue)
3. Check if physics simulation runs (Warp kernels accessing shared Vulkan buffers)
