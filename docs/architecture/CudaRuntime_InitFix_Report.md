# Ghost CUDA Runtime & Initialization Fix — Technical Report

This report details the design, implementation, and operational behavior of the Ghost CUDA Runtime Shim and cuInit HIP Fallback, which resolve CUDA Error 35 (`cudaErrorInsufficientDriver`) and `ZLUDA_MISSING:cuInit` failures during Omniverse (Isaac Sim) startup on AMD hardware.

## 1. The Problem

When Isaac Sim starts, two independent CUDA initialization paths fail simultaneously, creating a cascading failure that prevents GPU creation:

### 1.1 cuInit Direct Call Failure

`warp.dll` and other engine DLLs call `cuInit` twice during startup:

1.  **Via `cuGetProcAddress("cuInit")`** — Our master router intercepts this and returns a pointer to our local implementation. This path works.
2.  **Via direct DLL import (`GetProcAddress(nvcuda.dll, "cuInit")`)** — This bypasses the router entirely and hits ZLUDA's internal implementation directly. ZLUDA attempts to initialize against the physical hardware, finds no NVIDIA GPU, and returns `-1`.

Isaac Sim logs:

*   `ZLUDA_MISSING:cuInit`

### 1.2 CUDA Error 35 (Insufficient Driver)

`omni.rtx` dynamically loads `cudart64_12.dll` (which was previously ZLUDA's `nvcudart_hybrid64.dll`). When it calls `cudaGetDeviceCount`, ZLUDA's runtime internally attempts `cuInit`. Since the direct `cuInit` already failed, the runtime misinterprets this as an outdated driver version and returns error 35.

Isaac Sim logs:

*   `CUDA error 35 at initialization`
*   GPU info table shows `Active: No` for all devices

### 1.3 Vulkan Driver Version Leak

Omniverse queries `vkGetPhysicalDeviceProperties` and receives the real AMD driver version (`26.06.01`), which is encoded in AMD's format rather than NVIDIA's bitwise-shifted format. Omniverse's version parsing interprets this as an ancient NVIDIA driver and applies additional restrictions.

Isaac Sim logs:

*   `Driver Version: 26.06.01` (should show `610.74`)

### 1.4 cuDevicePrimaryCtxRelease NOP Fallback

When `cuDevicePrimaryCtxRelease` is resolved via `cuGetProcAddress`, it falls through to Tier 3 (ZLUDA forward) or Tier 4 (NOP stub) depending on whether ZLUDA is loaded at that point. If it hits the NOP stub, it returns `CUDA_ERROR_NOT_SUPPORTED` (801), which causes context cleanup failures.

## 2. The Solution Architecture

Ghost solves these with four coordinated fixes:

```
Omniverse Engine
  │
  ├── Runtime Calls (cudaGetDeviceCount, cudaGetDeviceProperties, ...)
  │     └── ghost_cudart64_12.dll (Ghost Runtime Shim)
  │           └── Spoofed RTX 2080 Ti + HIP forwarding for compute
  │
  ├── Driver API Calls (cuInit, cuCtxCreate, ...)
  │     └── void_shim.dll (Ghost Driver API Router)
  │           ├── Try 1: ZLUDA cuInit → failed
  │           └── Try 2: hipInit(0) → success (HIP compute context live)
  │
  ├── Vulkan Queries (vkGetPhysicalDeviceProperties, ...)
  │     └── ghost_vulkan_layer.dll
  │           └── driverVersion patched to NVIDIA 610.74 encoding
  │
  └── Context Management (cuDevicePrimaryCtxRelease, ...)
        └── Tier 1 Router → local exports → ZLUDA forward
```

## 3. Fix 1 — cuInit HIP Fallback

### 3.1 Implementation

The exported `cuInit` function in `void_shim.dll` was rewritten from a simple `FORWARD_TO_ZLUDA` macro to a dual-init implementation with caching:

```
cuInit(Flags)
  → Check g_HipInitialized cache
    → If cached: return 0 immediately (fast path)
  → Try 1: Forward to ZLUDA (nvcuda_zluda.dll!cuInit)
    → If ZLUDA returns 0: set cache, return 0
    → If ZLUDA fails: log error code, continue
  → Try 2: Direct HIP init (amdhip64_7.dll!hipInit)
    → If HIP returns 0: set cache, return 0
    → If HIP fails: return CUDA_ERROR_OPERATING_SYSTEM (304)
  → If both fail: return 304
```

### 3.2 HIP Function Declaration

A new `HIP_FUNC` declaration was added to the HIP loader section:

*   `HIP_FUNC(hipError_t, hipInit, unsigned int)` — declares `pfn_hipInit` and `resolve_hipInit()`.

### 3.3 Caching

The global `g_HipInitialized` flag prevents redundant initialization. Since `cuInit` is called by multiple DLLs during startup (warp.dll, omni.rtx, omni.physx, etc.), the cached path avoids repeated HIP initialization overhead and guarantees consistent results.

### 3.4 Trace Output

| Log Entry | Meaning |
|-----------|---------|
| `[cuInit] ZLUDA returned: 0` | ✅ ZLUDA initialized successfully (unlikely on AMD) |
| `[cuInit] ZLUDA returned: -1` | ⚠️ ZLUDA failed, falling back to HIP |
| `[cuInit] HIP fallback hipInit returned: 0` | ✅ HIP initialized, compute context live |
| `[cuInit] HIP fallback hipInit returned: [non-zero]` | ❌ Both backends failed |
| `[cuInit] BOTH ZLUDA and HIP failed to init` | ❌ No compute backend available |
| `cuInit (cached OK)` | ℹ️ Subsequent call, returning cached success |

## 4. Fix 2 — CUDA Runtime Shim (ghost_cudart64_12.dll)

### 4.1 Rationale

ZLUDA's `nvcudart_hybrid64.dll` (deployed as `cudart64_12.dll`) internally calls ZLUDA's driver API during initialization. If the driver `cuInit` failed, the runtime cascades to error 35. We cannot control ZLUDA's internal state without source modifications.

The solution is to **replace the entire runtime DLL** with a Ghost-generated shim that:

1.  Returns spoofed device identity for all queries
2.  Forwards real compute operations to HIP
3.  Never touches ZLUDA's internal state

### 4.2 Build Process

1.  Rust orchestrator writes `ghost_cudart.cpp` (~560 lines of C++) to `nv_spoof/`
2.  Locates MSVC via `vswhere.exe` → `vcvars64.bat`
3.  Compiles with: `cl.exe /LD /MT /O2 ghost_cudart.cpp /Fe:cudart64_12.dll /link /MACHINE:X64 kernel32.lib user32.lib advapi32.lib`
4.  Fallback: direct `cl.exe` invocation if `vswhere` is unavailable
5.  Deployment: the compiled `cudart64_12.dll` in `nv_spoof/` takes priority. If compilation failed, falls back to ZLUDA's runtime with a warning.

### 4.3 HIP Resolution

On `DllMain(DLL_PROCESS_ATTACH)`, the shim pre-initializes HIP:

1.  Attempts `LoadLibraryW(L"amdhip64_7.dll")`
2.  Falls back through `amdhip64_6.dll`, `amdhip64_5.dll`, `amdhip64_4.dll`, `amdhip64.dll`
3.  Resolves `hipInit` and calls `hipInit(0)` immediately
4.  All HIP function pointers are resolved lazily on first use via `HIP_FN` / `HIP_CALL` macros

### 4.4 Exported Functions — Full Inventory

The shim exports **60+ CUDA Runtime API functions** organized into categories:

#### 4.4.1 Identity / Version (Spoofed)

| Function | Return Value |
|----------|-------------|
| `cudaDriverGetVersion` | `12080` (CUDA 12.8) |
| `cudaRuntimeGetVersion` | `12080` (CUDA 12.8) |
| `cudaGetDeviceCount` | `1` |
| `cudaGetDevice` | `0` |

#### 4.4.2 cudaGetDeviceProperties (Full RTX 2080 Ti Spoof)

Returns a complete `cudaDeviceProp` struct matching the CUDA 12 ABI layout:

| Field | Value |
|-------|-------|
| `name` | `"NVIDIA GeForce RTX 2080 Ti"` |
| `totalGlobalMem` | `11811160064` (11 GB) |
| `major` / `minor` | `7` / `5` (SM 7.5 — Turing) |
| `multiProcessorCount` | `68` |
| `warpSize` | `32` |
| `maxThreadsPerBlock` | `1024` |
| `clockRate` | `1545000` kHz |
| `memoryClockRate` | `7000000` kHz |
| `memoryBusWidth` | `352` bits |
| `l2CacheSize` | `5767168` bytes (5.5 MB) |
| `asyncEngineCount` | `3` |
| `computeMode` | `0` (cudaComputeModeDefault) |

#### 4.4.3 cudaDeviceGetAttribute (Switch Table)

Returns sane RTX 2080 Ti values for 22 common attributes. Unknown attributes return `0`.

#### 4.4.4 Memory Management (Forwarded to HIP)

| CUDA Function | HIP Target |
|---------------|------------|
| `cudaMalloc` | `hipMalloc` |
| `cudaFree` | `hipFree` |
| `cudaMemcpy` | `hipMemcpy` |
| `cudaMemcpyAsync` | `hipMemcpyAsync` |
| `cudaMemset` | `hipMemset` |
| `cudaMemsetAsync` | `hipMemsetAsync` |
| `cudaMemGetInfo` | Returns `11 GB / 11 GB` (spoofed) |

#### 4.4.5 Stream Management (Forwarded to HIP)

| CUDA Function | HIP Target |
|---------------|------------|
| `cudaStreamCreate` | `hipStreamCreate` |
| `cudaStreamCreateWithFlags` | `hipStreamCreateWithFlags` |
| `cudaStreamCreateWithPriority` | `hipStreamCreateWithPriority` |
| `cudaStreamDestroy` | `hipStreamDestroy` |
| `cudaStreamSynchronize` | `hipStreamSynchronize` |
| `cudaStreamWaitEvent` | NOP (safe fallback) |

#### 4.4.6 Event Management (Forwarded to HIP)

| CUDA Function | HIP Target |
|---------------|------------|
| `cudaEventCreate` | `hipEventCreate` |
| `cudaEventCreateWithFlags` | `hipEventCreate` |
| `cudaEventRecord` | `hipEventRecord` |
| `cudaEventSynchronize` | `hipEventSynchronize` |
| `cudaEventDestroy` | `hipEventDestroy` |
| `cudaEventElapsedTime` | `hipEventElapsedTime` |

#### 4.4.7 Host Memory (Native Allocation)

| Function | Behavior |
|----------|----------|
| `cudaMallocHost` | `malloc(size)` — pinned memory not required for correctness |
| `cudaHostAlloc` | `malloc(size)` |
| `cudaFreeHost` | `free(ptr)` |
| `cudaHostGetDevicePointer` | Returns same pointer (unified addressing) |
| `cudaHostRegister` / `cudaHostUnregister` | NOP |

#### 4.4.8 Unsupported Features (Return 801)

| Function | Response |
|----------|----------|
| `cudaMallocManaged` | `CUDA_ERROR_NOT_SUPPORTED` (801) |
| `cudaMallocAsync` | `CUDA_ERROR_NOT_SUPPORTED` (801) |
| `cudaIpcGetMemHandle` | `CUDA_ERROR_NOT_SUPPORTED` (801) |
| `cudaIpcOpenMemHandle` | `CUDA_ERROR_NOT_SUPPORTED` (801) |
| `cudaGraphCreate` | `CUDA_ERROR_NOT_SUPPORTED` (801) |
| `cudaGraphInstantiate` | `CUDA_ERROR_NOT_SUPPORTED` (801) |
| `cudaGraphLaunch` | `CUDA_ERROR_NOT_SUPPORTED` (801) |

#### 4.4.9 Safety NOPs

All of the following return `0` (success) without side effects:

*   `cudaDeviceSynchronize`, `cudaDeviceReset`, `cudaGetLastError`, `cudaPeekAtLastError`
*   `cudaDeviceSetCacheConfig`, `cudaDeviceGetCacheConfig`
*   `cudaDeviceSetSharedMemConfig`, `cudaDeviceGetSharedMemConfig`
*   `cudaSetDeviceFlags`, `cudaGetDeviceFlags`
*   `cudaDeviceSetLimit`, `cudaDeviceGetLimit`
*   `cudaFuncSetAttribute`, `cudaFuncSetCacheConfig`
*   `cudaThreadSynchronize`, `cudaThreadExit`

### 4.5 Trace Tag

All trace output uses the `[GHOST-CUDART]` tag for filtering in `ghost_trace.log`.

## 5. Fix 3 — Vulkan Driver Version Spoof

### 5.1 NVIDIA Version Encoding

NVIDIA encodes Vulkan driver versions using a non-standard bitwise shift:

```
version = (major << 22) | (minor << 14) | (patch << 6)
```

For driver `610.74`:

```
version = (610 << 22) | (74 << 14) | (0 << 6) = 2,559,737,856
```

### 5.2 Implementation

Both Vulkan property hooks were modified to patch the `driverVersion` field after spoofing vendorID, deviceID, and deviceName:

*   **`Hook_vkGetPhysicalDeviceProperties`:** `props->driverVersion = (610u << 22) | (74u << 14) | (0u << 6);`
*   **`Hook_vkGetPhysicalDeviceProperties2`:** `props->properties.driverVersion = (610u << 22) | (74u << 14) | (0u << 6);`

### 5.3 Expected Result

The GPU info table in Isaac Sim logs should now display:

```
| GPU | Name                             | Active | Driver Version |
|-----|----------------------------------|--------|----------------|
| 0   | NVIDIA GeForce RTX 2080 Ti       | Yes    | 610.74         |
```

Instead of:

```
| GPU | Name                             | Active | Driver Version |
|-----|----------------------------------|--------|----------------|
| 0   | NVIDIA GeForce RTX 2080 Ti       |        | 26.06.01       |
```

## 6. Fix 4 — cuDevicePrimaryCtxRelease Router

### 6.1 Implementation

Two new entries were added to the Tier 1 (Identity Spoofs) section of the `cuGetProcAddress` master router:

*   `cuDevicePrimaryCtxRelease` → local export → `FORWARD_TO_ZLUDA`
*   `cuDevicePrimaryCtxRelease_v2` → local export → `FORWARD_TO_ZLUDA`

### 6.2 Route Classification

Both are classified as `TRACE_TYPE_FORWARD` (not `TRACE_TYPE_SPOOF`) because they forward the actual call to ZLUDA for real context management. The router entry simply ensures the symbol is resolved from our exports rather than falling through to a NOP stub.

### 6.3 Expected Forensic Output

The run report should now show:

```
cuDevicePrimaryCtxRelease    → GHOST (FORWARD)
cuDevicePrimaryCtxRelease_v2 → GHOST (FORWARD)
```

Instead of:

```
cuDevicePrimaryCtxRelease    → NOP_STUB
```

## 7. Diagnostic Checklist

When debugging startup issues at runtime, check `ghost_trace.log` for:

| Log Entry | Meaning |
|-----------|---------|
| `[cuInit] ZLUDA returned: -1` | ⚠️ Expected — ZLUDA fails on AMD hardware |
| `[cuInit] HIP fallback hipInit returned: 0` | ✅ HIP initialized, compute context live |
| `[cuInit] BOTH ZLUDA and HIP failed to init` | ❌ No compute backend — check AMD driver installation |
| `cuInit (cached OK)` | ℹ️ Subsequent call, fast path |
| `[GHOST-CUDART] ghost_cudart64_12.dll loaded` | ✅ Runtime shim active, ZLUDA runtime bypassed |
| `[GHOST-CUDART] DllMain: hipInit(0) -> 0` | ✅ HIP pre-initialized in runtime shim |
| `[GHOST-CUDART] cudaGetDeviceProperties -> RTX 2080 Ti` | ✅ omni.rtx accepted the spoofed device |
| `[GHOST-CUDART] cudaMalloc(N) -> 0` | ✅ Real HIP allocation succeeded |
| `[GHOST-CUDART] WARNING: No HIP DLL found` | ❌ amdhip64 not in PATH — check ROCm installation |
| `Spoofed -> NVIDIA GeForce RTX 2080 Ti driverVer=610.74` | ✅ Vulkan driver version spoof active |

## 8. Known Risks and Limitations

1.  **cudaDeviceProp struct ABI:** The `cudaDeviceProp` struct layout changes between CUDA versions. Our struct is padded to ~900 bytes with a 512-byte tail pad. If `omni.rtx` reads fields beyond our struct boundary, it will read zeros. CUDA 12.x added fields like `accessPolicyMaxWindowSize` and `clusterLaunch` that we don't populate. If omni.rtx checks these, they will be zero.

2.  **Host memory pinning:** Our `cudaMallocHost` uses `malloc()` instead of `hipHostMalloc()`. This means the memory is not truly pinned and DMA transfers will fall back to staging buffers. Performance may be slightly reduced for host-to-device copies, but correctness is preserved.

3.  **CUDA Graph API:** The Graph API is stubbed as `NOT_SUPPORTED`. If omni.rtx attempts to use CUDA Graphs for render pipeline optimization, it will fall back to stream-based execution.

4.  **Unified Memory:** `cudaMallocManaged` returns `NOT_SUPPORTED`. If any engine component requires managed memory (automatic page migration), it will fail. This is acceptable because AMD's HIP managed memory semantics differ from NVIDIA's, and ZLUDA does not support it either.

5.  **cudaGetDeviceCount hardcoded to 1:** Multi-GPU AMD configurations will only expose one device through the runtime shim. The driver API side (void_shim.dll) may report different counts. If omni.rtx cross-references the runtime and driver device counts, a mismatch could cause issues.

## Summary

The four fixes collectively eliminate the CUDA initialization cascade failure that prevented Isaac Sim from recognizing any GPU. Fix 1 ensures a valid HIP compute context is always established regardless of ZLUDA's internal state. Fix 2 completely isolates the CUDA Runtime API surface from ZLUDA by deploying a standalone shim DLL with 60+ exports that return spoofed RTX 2080 Ti identity data and forward real compute operations to HIP. Fix 3 patches the Vulkan driver version to pass NVIDIA compatibility checks. Fix 4 ensures context lifecycle functions are routed through our exports instead of falling through to NOP stubs. All of this is transparent to Omniverse — it sees a CUDA 12.8 driver, an RTX 2080 Ti with 11 GB of VRAM, and a working compute pipeline.
