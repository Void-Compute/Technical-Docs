# Omniverse Dependency Deep Dive Report

Written for 143 DLLs in `E:\ghost_project\isaac-sim-standalone-6.0.0-windows-x86_64\extscache\omni.gpu_foundation-0.0.0+6312fa25.wx64.r.cp312\bin\deps`.

## 1. Proprietary NVIDIA SDKs (High Crash Risk)

These libraries are not part of the standard open-source framework and are likely to contain hardcoded NVIDIA driver checks, making them candidates for NOP-stubbing, custom implementations or, if possible, replacement via `extension.toml` redirects.

### cudart64_12.dll
- **Tech Stack**: cu, cuda
- **Notable Strings**:
  - `a kernel launch error has occurred due to cluster misconfiguration`
  - `an async error has occured in external entity outside of CUDA`
  - `cuGetErrorName`
  - `cuGetErrorString`
  - `cudaDeviceSynchronize failed because caller's grid depth exceeds cudaLimitDevRuntimeSyncDepth`
  - `cudaErrorAddressOfConstant`
  - `cudaErrorAlreadyAcquired`
  - `cudaErrorAlreadyMapped`
  - `cudaErrorApiFailureBase`
  - `cudaErrorArrayIsMapped`
  - `.CRT$XCU`
  - `API call is not supported in the installed CUDA driver`
  - `CUDA driver is a stub library`
  - `CUDA driver version is insufficient for CUDA runtime version`
  - `CUDA-capable device(s) is/are busy or unavailable`

### cupti64_2025.1.0.dll
- **Tech Stack**: cu, cuda, d3d12, optix, vulkan
- **Notable Strings**:
  - `Assertion failed, no current source file`
  - `CUPTI Error: (%d:%d)`
  - `CUPTI_ERROR_API_NOT_IMPLEMENTED`
  - `CUPTI_ERROR_CANT_OPEN_FILE`
  - `CUPTI_ERROR_CDP_TRACING_NOT_SUPPORTED`
  - `CUPTI_ERROR_CMP_DEVICE_NOT_SUPPORTED`
  - `CUPTI_ERROR_CONFIDENTIAL_COMPUTING_NOT_SUPPORTED`
  - `CUPTI_ERROR_CUDA_COMPILER_NOT_COMPATIBLE`
  - `CUPTI_ERROR_DISABLED`
  - `CUPTI_ERROR_FROM_DBMS`

### nvperf_grfx_host.dll
- **Tech Stack**: cu, cuda, d3d12, nvapi, nvperf, optix, raytracing, vulkan
- **Notable Strings**:
  - `# of samples discarded by ZCULL, failing depth test (non-exclusively)`
  - `Failed to execute bootstrap`
  - `Failed to get cuGetExportTable`
  - `Failed to get optixQueryFunctionTable`
  - `[CudaProfiler.cpp::BeginSessionOnCuContext()] pCuContext->profiler.submitter.ccuProf.CcuProfInitAll() failed!`
  - `[CudaProfiler.cpp::BeginSessionOnCuContext()] pCuContext->profiler.submitter.ccuProf.ClearInaccessibleCcuProfs() failed!`

### nvtt30205.dll
- **Tech Stack**: cu, cuda
- **Notable Strings**:
  - `CUDA Error from %s (%s line %d): %s (%s)`
  - `CUDA Error from %s: %s (%s)`
  - `CUDA error`
  - `a kernel launch error has occurred due to cluster misconfiguration`

### slang.dll
- **Tech Stack**: cu, cuda, d3d12, nvapi, optix, raytracing, vulkan
- **Notable Strings**:
  - `[__requiresNVAPI]`
  - `// Also note that you *can* include NVAPI headers in your Slang source, and directly use NVAPI functions`
  - `// NvAPI support on DX`
  - `//! SLANG_FAIL is the generic failure code — meaning a serious error occurred and the call failed`

### GFSDK_Aftermath_Lib.x64.dll
- **Tech Stack**: aftermath, cu, vulkan
- **Notable Strings**:
  - `GFSDK_Aftermath_DX11_Initialize`
  - `GFSDK_Aftermath_DX12_CreateContextHandle`
  - `GFSDK_Aftermath_DX12_Initialize`
  - `GFSDK_Aftermath_DisableGpuCrashDumps`
  - `GFSDK_Aftermath_EnableGpuCrashDumps`

### libgstnvcodecs.dll
- **Tech Stack**: cu, cuda
- **Notable Strings**:
  - `CUDA call failed: %s, %s`
  - `CuCtxPushCurrent failed: codec %s, device %i, error code %i`
  - `Failed to create CUDA context`
  - `Failed to create CUDA context for cuda device %d`
  - `Failed to cuInit`

### libnvbufsurface.dll
- **Tech Stack**: cu, cuda
- **Notable Strings**:
  - `%s: cuExternalMemoryGetMappedBuffer failed result %d`
  - `%s: cuImportExternalMemory failed result %d`
  - `Cuda failure: status=%d`
  - `NvBufSurfaceCopy: cudaArrayGetPlane failed for destination`

### libnvbufsurftransform.dll
- **Tech Stack**: cu, cuda
- **Notable Strings**:
  - `%s(%i) : getLastCudaError() CUDA error : %s : (%d) %s.`
  - `CUDA error at %s:%d code=%d(%s) "%s" `
  - `Cuda failure: status=%d in %s at line %d`
  - `Failed to get last cuda error %s`

### nppc64_12.dll
- **Tech Stack**: cu, cuda
- **Notable Strings**: (CUDA-related error messages)

### nppidei64_12.dll
- **Tech Stack**: cu, cuda
- **Notable Strings**: (CUDA-related error messages)

### nppig64_12.dll
- **Tech Stack**: cu, cuda
- **Notable Strings**: (CUDA-related error messages)

## 2. Carbonite Engine Plugins

These are the core Omniverse modules. They typically implement C++ interfaces and can be bypassed or replaced via `extension.toml`.

### carb.blockcompression.plugin.dll
- **Tech Stack**: carb, cu, cuda
- **Notable Strings**:
  - `nvtt: Cuda error`

### carb.cudainterop.plugin.dll
- **Tech Stack**: carb, cu, cuda
- **Notable Strings**:
  - `CUDA error %d: %s — %s)`
  - `CUDA error at %s:%d code=%d(%s) "%s" `
  - `CUPTI call failed with: %s `
  - `CUPTI failed. There can't be multiple instances of MemoryActivityGlobal.`

### carb.graphics-direct3d.plugin.dll
- **Tech Stack**: aftermath, carb, cu, d3d12, dlss, ngx, nvapi, nvperf, raytracing, vulkan
- **Notable Strings**:
  - `Aftermath D3D12 failed to initialize.`
  - `Aftermath Error 0x`

### carb.graphics-vulkan.plugin.dll
- **Tech Stack**: aftermath, carb, cu, d3d12, dlss, ngx, nvapi, nvperf, raytracing, vulkan
- **Notable Strings**:
  - `Aftermath is incompatible with D3D API interception, such as PIX or Nsight Graphics.`
  - `Aftermath unsupported driver version — A newer driver version is required for the current features.`

## 3. Generic 3rd-Party / Utility Libraries

Standard open-source or utility libraries (e.g. zlib, Assimp, standard Vulkan loaders). Generally safe.

Examples include: `driver_shader_cache_wrapper.dll`, `dxcompiler.dll`, `dxil.dll`, `gio-2.0-0.dll`, `glew32.dll`, `glib-2.0-0.dll`, `gmodule-2.0-0.dll`, `gobject-2.0-0.dll`, and many others.
