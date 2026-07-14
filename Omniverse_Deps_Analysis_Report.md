# Omniverse Dependency Deep Dive Report
Written for 143 DLLs in `E:\ghost_project\isaac-sim-standalone-6.0.0-windows-x86_64\extscache\omni.gpu_foundation-0.0.0+6312fa25.wx64.r.cp312\bin\deps`.

## 1. Proprietary NVIDIA SDKs (High Crash Risk)
These libraries are not part of the standard open-source framework and are likely to contain hardcoded NVIDIA driver checks, making them candidates for NOP-stubbing, custom implemenations or if possible ZLUDA forwarding

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
  - `!ComputeEtbl::ClCuEtbl`
  - `$__internal_0_$__cuda_sm20_div_u64`
  - `.?AUIContextState@Cuda@NV@@`
  - `.?AV<lambda_2>@?1??OptimizeBackingStore@ContextState@Cuda@NV@@UEAA_NPEAUContextStateOptimization@34@@Z@`
  - `.?AV<lambda_3>@?1??OptimizeBackingStore@ContextState@Cuda@NV@@UEAA_NPEAUContextStateOptimization@34@@Z@`

### nvperf_grfx_host.dll
- **Tech Stack**: cu, cuda, d3d12, nvapi, nvperf, optix, raytracing, vulkan
- **Notable Strings**:
  - `# of samples discarded by ZCULL, failing depth test (non-exclusively)`
  - `# of samples discarded by ZCULL, failing depth-bounds test (non-exclusively)`
  - `# of samples discarded by ZCULL, failing near/far clip plane test (non-exclusively)`
  - `# of samples discarded by ZCULL, failing stencil test (non-exclusively)`
  - `# of samples output by ZCULL, that failed depth-test but require further processing`
  - `Failed to execute bootstrap`
  - `Failed to get cuGetExportTable`
  - `Failed to get optixQueryFunctionTable`
  - `[CudaProfiler.cpp::BeginSessionOnCuContext()] pCuContext->profiler.submitter.ccuProf.CcuProfInitAll() failed!`
  - `[CudaProfiler.cpp::BeginSessionOnCuContext()] pCuContext->profiler.submitter.ccuProf.ClearInaccessibleCcuProfs() failed!`
  - ` // Cuda compilation tools, release 10.0, V10.0.19 `
  - `# of 8-bit floating-point thread instructions executed where all predicates were true`
  - `# of DADD thread instructions executed where all predicates were true`
  - `# of DADD, DMUL and DFMA thread instructions executed where all predicates were true`
  - `# of DFMA thread instructions executed where all predicates were true`

### nvtt30205.dll
- **Tech Stack**: cu, cuda
- **Notable Strings**:
  - `-45_GLOBAL__N__bd4bfeae_12_nvdxt_gpu_cu_69dd9d7316error_bc1_i`
  - `-45_GLOBAL__N__bd4bfeae_12_nvdxt_gpu_cu_69dd9d7316error_bc1_i!`
  - `-45_GLOBAL__N__bd4bfeae_12_nvdxt_gpu_cu_69dd9d7316error_bc1_iH`
  - `.?AVscheduler_resource_allocation_error@Concurrency@@`
  - `.?AVscheduler_worker_creation_error@Concurrency@@`
  - `745_GLOBAL__N__bd4bfeae_12_nvdxt_gpu_cu_69dd9d7316error_bc1_indiceILb0E`
  - `CUDA Error from %s (%s line %d): %s (%s)`
  - `CUDA Error from %s: %s (%s)`
  - `CUDA error`
  - `a kernel launch error has occurred due to cluster misconfiguration`
  - `        <requestedExecutionLevel level='asInvoker' uiAccess='false' />`
  - `    </security>`
  - `    <security>`
  - `%*0CU`
  - `'c_34__bd4bfeae_12_nvdxt_gpu_cu_69dd9d73__ZN43_INTERNAL1`

### slang.dll
- **Tech Stack**: cu, cuda, d3d12, nvapi, optix, raytracing, vulkan
- **Notable Strings**:
  - `            // Calculate a relative relative error`
  - `            // SubgroupMemory triggers vulkan validation layer error; `
  - `        [__requiresNVAPI]`
  - `    // Also note that you *can* include NVAPI headers in your Slang source, and directly use NVAPI functions`
  - `    // Directly using NVAPI functions does *not* add the #include on the output`
  - `    // Finally note you can *mix* NVAPI direct calls, and use of NVAPI intrinsics below. This doesn't cause`
  - `    // NOTE! To use this feature on HLSL based targets the path to 'nvHLSLExtns.h' from the NvAPI SDK must`
  - `    // NvAPI support on DX`
  - `    // any clashes, as Slang will emit any NVAPI function it parsed (say via a include in Slang source) with`
  - `    //! SLANG_FAIL is the generic failure code - meaning a serious error occurred and the call`
  - `                            // Document seems to have a typo. `lod` must be `sample`.`
  - `                            // The document seems to have a typo. `lod` must mean `sample`.`
  - `                        /**/ $CurrentTime`
  - `                        /**/ $CurrentTime;`
  - `                        __intrinsic_asm "$c$0.sample($1, $2, gradientcube($3, $4))$z";`

### cudart64_12.dll.dll
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

### GFSDK_Aftermath_Lib.x64.dll
- **Tech Stack**: aftermath, cu, vulkan
- **Notable Strings**:
  - `GFSDK_Aftermath_DX11_Initialize`
  - `GFSDK_Aftermath_DX12_CreateContextHandle`
  - `GFSDK_Aftermath_DX12_Initialize`
  - `GFSDK_Aftermath_DisableGpuCrashDumps`
  - `GFSDK_Aftermath_EnableGpuCrashDumps`
  - `GFSDK_Aftermath_GetCrashDumpStatus`
  - `GFSDK_Aftermath_GetData`
  - `GFSDK_Aftermath_GetShaderDebugInfoIdentifier`
  - `GFSDK_Aftermath_GetShaderHash`
  - `GFSDK_Aftermath_GetShaderHashForShaderInfo`
  - `GetCurrentProcess`
  - `GetCurrentProcessId`
  - `GetCurrentThreadId`

### libgstnvcodecs.dll
- **Tech Stack**: cu, cuda
- **Notable Strings**:
  - `CUDA call failed: %s, %s`
  - `CuCtxPushCurrent failed: codec %s, device %i, error code %i`
  - `CuGetErrorName`
  - `CuGetErrorString`
  - `Failed to create CUDA context`
  - `Failed to create CUDA context for cuda device %d`
  - `Failed to create CUDA context with device-id %d`
  - `Failed to create cuda context, ret: 0x%x`
  - `Failed to cuInit`
  - `Failed to free CUDA device memory, ret %d`
  - `        <requestedExecutionLevel level='asInvoker' uiAccess='false' />`
  - `    </security>`
  - `    <security>`
  - `A) with cuda-device-id %d on context(%p`
  - `C:\Users\local-harikrishnan\Downloads\rel-38\3rdparty\gst\gst-v4l2\nvcodec\gstcudabufferpool.c`

### libnvbufsurface.dll
- **Tech Stack**: cu, cuda
- **Notable Strings**:
  - `%s: cuExternalMemoryGetMappedBuffer failed result %d`
  - `%s: cuImportExternalMemory failed result %d`
  - `%s: failed to allocate memory for cudaBuffer`
  - `Cuda failure: status=%d`
  - `NvBufSurfaceCopy: cudaArrayGetPlane failed for destination`
  - `NvBufSurfaceCopy: cudaArrayGetPlane failed for source`
  - `nvbufsurface: Error(%d) in releasing cuda memory`
  - `nvbufsurface: NvBufSurfaceGetCudaFd failed`
  - `nvbufsurface: cuCtxGetCurrent failed: %d, line: %d`
  - `nvbufsurface: failed to allocate memory for cudaBuffer`
  - `        <requestedExecutionLevel level='asInvoker' uiAccess='false' />`
  - `    </security>`
  - `    <security>`
  - `%s: Unable to allocate NvBufSurfaceCudaBuffer memory`
  - `%s: cudaIpcGetMemHandle err: %d`

### libnvbufsurftransform.dll
- **Tech Stack**: cu, cuda
- **Notable Strings**:
  - `%s(%i) : getLastCudaError() CUDA error : %s : (%d) %s.`
  - `??$check@W4cudaError@@@@YA_NW4cudaError@@QEBD1H@Z`
  - `?__getLastCudaError@@YAXPEBD0H@Z`
  - `?getCuPtrAndParams@NvBufResStorage@@QEAA?AW4NvBufSurfTransform_Error@@PEAUNvBufSurface@@IIPEAU_NvBufResParams@@W4NvBufSurfTransform_Inter@@_N@Z`
  - `CUDA error at %s:%d code=%d(%s) "%s" `
  - `Cuda failure: status=%d in %s at line %d`
  - `Failed to get last cuda error %s`
  - `NPP_CUDA_KERNEL_EXECUTION_ERROR`
  - `Recevied NvBufSurfTransformError_Execution_Error`
  - `cuArrayGetPlane failed`
  - `        <requestedExecutionLevel level='asInvoker' uiAccess='false' />`
  - `    </security>`
  - `    <security>`
  - `$__internal_0_$__cuda_sm3x_div_rn_noftz_f32_slowpath`
  - `$__internal_10_$__cuda_sm3x_div_rn_noftz_f32_slowpath`

### nppc64_12.dll
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

### nppidei64_12.dll
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
  - `%6]cu`
  - `.CRT$XCU`
  - `.global .align 8 .b8 __cudart_i2opi_d[144] = {8, 93, 141, 31, 177, 95, 25`
  - `.note.nv.cuver`
  - `.visible .entry _Z12setup_philoxIdLi1EEvP24curandStateP`

### nppig64_12.dll
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
  - ` _Cubic`
  - `$_Cubic`
  - `$_Cubic;`
  - `$_CubicJ`
  - `$__internal_0_$__cuda_sm3x_div_rn_noftz_f32_slowpath`

## 2. Carbonite Engine Plugins
These are the core Omniverse modules. They typically implement C++ interfaces and can be bypassed or replaced via `extension.toml`.

### carb.blockcompression.plugin.dll
- **Tech Stack**: carb, cu, cuda
- **Exposed/Used Interfaces**: carb::assert::IAssert, carb::blockcompression::BlockCompression, carb::cudainterop::CudaInterop, carb::dictionary::IDictionary, carb::dictionary::v1::IDictionary ...
- **Notable Strings**:
  - `nvtt: Cuda error`
  - `        <requestedExecutionLevel level='asInvoker' uiAccess='false' />`
  - `    </security>`
  - `    <security>`
  - `.CRT$XCU`

### carb.cudainterop.plugin.dll
- **Tech Stack**: carb, cu, cuda
- **Exposed/Used Interfaces**: carb::IFailureInjector, carb::assert::IAssert, carb::cudainterop::CudaInterop, carb::cudainterop::allocateDeviceMemory, carb::cudainterop::allocateDeviceMemoryAsync ...
- **Notable Strings**:
  - `CUDA error %d: %s - %s)`
  - `CUDA error at %s:%d code=%d(%s) "%s" `
  - `CUPTI call failed with: %s `
  - `CUPTI failed. There can't be multiple instances of MemoryActivityGlobal.`
  - `CUPTI: buffersCallbackCompleted failed. Result: %s `

### carb.failureinjector.plugin.dll
- **Tech Stack**: carb, cu
- **Exposed/Used Interfaces**: carb::IFailureInjector, carb::assert::IAssert, carb::detail::rstring::Internals::Internals, carb::detail::rstring::Internals::allocChunk, carb::detail::rstring::Internals::init ...
- **Notable Strings**:
  - `C:\g\309888382\rendering\_build\windows-x86_64\release\plugins\gpu.foundation\carb.failureinjector.plugin.pdb`
  - `C:\g\309888382\rendering\source\plugins\carb.failureinjector\FailureInjector.cpp`
  - `Ignoring unsupported carb.failureinjector rule type %u`
  - `carb.failureinjector`
  - `carb.failureinjector.plugin.dll`

### carb.glinterop.plugin.dll
- **Tech Stack**: carb, cu
- **Exposed/Used Interfaces**: carb::assert::IAssert, carb::dictionary::IDictionary, carb::dictionary::v1::IDictionary, carb::extras::getLibraryFilenameByHandle, carb::glinterop::ExternalSemaphore ...
- **Notable Strings**:
  - `        <requestedExecutionLevel level='asInvoker' uiAccess='false' />`
  - `    </security>`
  - `    <security>`
  - `.CRT$XCU`
  - `5CUxL`

### carb.graphics-direct3d.plugin.dll
- **Tech Stack**: aftermath, carb, cu, d3d12, dlss, ngx, nvapi, nvperf, raytracing, vulkan
- **Exposed/Used Interfaces**: carb::Buffer, carb::IFailureInjector, carb::assert::IAssert, carb::detail::rstring::Internals::Internals, carb::detail::rstring::Internals::allocChunk ...
- **Notable Strings**:
  - `/renderer/debug/aftermath/verboseLogging/enabled`
  - `Acquiring the optional carb::memorytracking2::IGpuMemoryTracker interface failed in carb.graphics.`
  - `Aftermath D3D12 failed to initialize.`
  - `Aftermath Error 0x`
  - `Aftermath context failed to initialize.`

### carb.graphics-vulkan.plugin.dll
- **Tech Stack**: aftermath, carb, cu, d3d12, dlss, ngx, nvapi, nvperf, raytracing, vulkan
- **Exposed/Used Interfaces**: carb::Buffer, carb::IFailureInjector, carb::assert::IAssert, carb::detail::rstring::Internals::Internals, carb::detail::rstring::Internals::allocChunk ...
- **Notable Strings**:
  - `/renderer/debug/aftermath/verboseLogging/enabled`
  - `Acquiring the optional carb::memorytracking2::IGpuMemoryTracker interface failed in carb.graphics.`
  - `Aftermath Error 0x`
  - `Aftermath is incompatible with D3D API interception, such as PIX or Nsight Graphics.`
  - `Aftermath unsupported driver version - A newer driver version is required for the current features.`

### carb.graphics_optional.plugin.dll
- **Tech Stack**: carb, cu, dlss
- **Exposed/Used Interfaces**: carb::Framework, carb::assert::IAssert, carb::dictionary::IDictionary, carb::dictionary::v1::IDictionary, carb::extras::getLibraryFilenameByHandle ...
- **Notable Strings**:
  - `Failed to get carb::Framework`
  - `        <requestedExecutionLevel level='asInvoker' uiAccess='false' />`
  - `    </security>`
  - `    <security>`
  - `.CRT$XCU`

### carb.imguiregistry.plugin.dll
- **Tech Stack**: carb, cu
- **Exposed/Used Interfaces**: carb::assert::IAssert, carb::dictionary::IDictionary, carb::dictionary::v1::IDictionary, carb::filesystem::IFileSystem, carb::filesystem::v1::IFileSystem ...
- **Notable Strings**:
  - `ImguiDebugWindow enabled, but failed to find carb.imguiregistry.plugin.`
  - `carbFailure`
  - `carbFailureMessage`
  - `        <requestedExecutionLevel level='asInvoker' uiAccess='false' />`
  - `    </security>`

### carb.memorytracking.plugin.dll
- **Tech Stack**: carb, cu, cuda
- **Exposed/Used Interfaces**: carb::assert::IAssert, carb::detail::rstring::Internals::Internals, carb::detail::rstring::Internals::allocChunk, carb::detail::rstring::Internals::init, carb::dictionary::IDictionary ...
- **Notable Strings**:
  - `        <requestedExecutionLevel level='asInvoker' uiAccess='false' />`
  - `    </security>`
  - `    <security>`
  - `.CRT$XCU`
  - `/plugins/carb.memorytracking.plugin/enabled`

### carb.profiler-gpu.plugin.dll
- **Tech Stack**: carb, cu, cuda, d3d12, vulkan
- **Exposed/Used Interfaces**: carb::ErrorApi, carb::assert::IAssert, carb::cudainterop::CudaInterop, carb::detail::getCarbErrorApiFunc, carb::dictionary ...
- **Notable Strings**:
  - `C:\g\309888382\rendering\_build\target-deps\carb_sdk_plugins\include\carb\Error.h`
  - `Could not find `carbGetErrorApi` function at runtime -- #define CARB_REQUIRE_LINKED 1 before including this file`
  - `carbGetErrorApi`
  - `const struct carb::ErrorApi *__cdecl carb::ErrorApi::instance::<lambda_5c2a58d68438e8a814de5530c779f483>::operator ()(void) const`
  - `const void *(__cdecl *__cdecl carb::detail::getCarbErrorApiFunc(void))(struct carb::Version *)`

### carb.shadercompiler-slang.plugin.dll
- **Tech Stack**: carb, cu, vulkan
- **Exposed/Used Interfaces**: carb::assert::IAssert, carb::dictionary::IDictionary, carb::dictionary::v1::IDictionary, carb::extras::getLibraryFilenameByHandle, carb::l10n::IL10n ...
- **Notable Strings**:
  - `        <requestedExecutionLevel level='asInvoker' uiAccess='false' />`
  - `    </security>`
  - `    <security>`
  - `.CRT$XCU`
  - `C:\g\309888382\rendering\_build\target-deps\carb_sdk_plugins\include\carb\InterfaceUtils.h`

### carb.vg-nanovg.plugin.dll
- **Tech Stack**: carb, cu
- **Exposed/Used Interfaces**: carb::assert::IAssert, carb::dictionary::IDictionary, carb::dictionary::v1::IDictionary, carb::graphics::Graphics, carb::l10n::IL10n ...
- **Notable Strings**:
  - `        <requestedExecutionLevel level='asInvoker' uiAccess='false' />`
  - `    </security>`
  - `    <security>`
  - `.CRT$XCU`
  - `C:\g\309888382\rendering\_build\target-deps\carb_sdk_plugins\include\carb\InterfaceUtils.h`

### gpu.foundation.plugin.dll
- **Tech Stack**: aftermath, carb, cu, cuda, d3d12, ngx, nvapi, raytracing, vulkan
- **Exposed/Used Interfaces**: carb::ErrorApi, carb::Format)., carb::Framework, carb::IFailureInjector, carb::assert::IAssert ...
- **Notable Strings**:
  - `%s failed to allocate array (failed to match CUDA format-descriptor to carb::Format).`
  - `%s failed to allocate array (unable to access CUDA-interop interface).`
  - `%s failed to allocate array (unable to match CUDA device id '%u' to a valid graphics device).`
  - `%s failed to allocate buffer (unable to match CUDA device id '%u' to a valid graphics device).`
  - `%s failed to allocate texture from resource manager (CUDA interop failed, no valid CUDA array).`

### omni.perfmonitor.plugin.dll
- **Tech Stack**: carb, cu, d3d12, raytracing
- **Exposed/Used Interfaces**: carb::assert::IAssert, carb::dictionary::IDictionary, carb::dictionary::v1::IDictionary, carb::filesystem::IFileSystem, carb::l10n::IL10n ...
- **Notable Strings**:
  - `        <requestedExecutionLevel level='asInvoker' uiAccess='false' />`
  - `    </security>`
  - `    <security>`
  - `.?AV?$CachedInterface@UIDictionary@v1@dictionary@carb@@$0A@@detail@carb@@`
  - `.?AV?$CachedInterface@VISettings@v2@settings@carb@@$0A@@detail@carb@@`

### omni.streamingstatus.plugin.dll
- **Tech Stack**: carb, cu
- **Exposed/Used Interfaces**: carb::assert::IAssert, carb::detail::getCarbReallocate, carb::detail::rstring::Internals::Internals, carb::detail::rstring::Internals::allocChunk, carb::detail::rstring::Internals::init ...
- **Notable Strings**:
  - `ImguiDebugWindow enabled, but failed to find carb.imguiregistry.plugin.`
  - `carbFailure`
  - `carbFailureMessage`
  - `        <requestedExecutionLevel level='asInvoker' uiAccess='false' />`
  - `    </security>`

## 3. Generic 3rd-Party / Utility Libraries
Standard open-source or utility libraries (e.g. zlib, Assimp, standard Vulkan loaders). Generally safe.

`driver_shader_cache_wrapper.dll`, `dxcompiler.dll`, `dxil.dll`, `gio-2.0-0.dll`, `girepository-2.0-0.dll`, `glew32.dll`, `glib-2.0-0.dll`, `gmodule-2.0-0.dll`, `gobject-2.0-0.dll`, `gstapp-1.0-0.dll`, `gstbase-1.0-0.dll`, `gstcheck-1.0-0.dll`, `gstcontroller-1.0-0.dll`, `gstnet-1.0-0.dll`, `gstreamer-1.0-0.dll`, `gthread-2.0-0.dll`, `slang-glslang.dll`, `VkLayer_khronos_validation.dll`, `WinPixEventRuntime.dll`, `gstadder.dll`, `gstalaw.dll`, `gstallocators-1.0-0.dll`, `gstalpha.dll`, `gstalphacolor.dll`, `gstapetag.dll`, `gstapp-1.0-0.dll`, `gstapp.dll`, `gstaudio-1.0-0.dll`, `gstaudioconvert.dll`, `gstaudiofx.dll`, `gstaudiomixer.dll`, `gstaudioparsers.dll`, `gstaudiorate.dll`, `gstaudioresample.dll`, `gstaudiotestsrc.dll`, `gstauparse.dll`, `gstautodetect.dll`, `gstavi.dll`, `gstbasedebug.dll`, `gstcodecparsers-1.0-0.dll`, `gstcompositor.dll`, `gstcoreelements.dll`, `gstcoretracers.dll`, `gstcutter.dll`, `gstdebug.dll`, `gstdeinterlace.dll`, `gstdirectsound.dll`, `gstdsd.dll`, `gstdtmf.dll`, `gsteffectv.dll`, `gstencoding.dll`, `gstequalizer.dll`, `gstfft-1.0-0.dll`, `gstflv.dll`, `gstflxdec.dll`, `gstgoom.dll`, `gstgoom2k1.dll`, `gstid3demux.dll`, `gstimagefreeze.dll`, `gstinterleave.dll`, `gstisomp4.dll`, `gstjack.dll`, `gstlevel.dll`, `gstmonoscope.dll`, `gstmulaw.dll`, `gstmultipart.dll`, `gstnavigationtest.dll`, `gstoverlaycomposition.dll`, `gstpbtypes.dll`, `gstpbutils-1.0-0.dll`, `gstplayback.dll`, `gstrawparse.dll`, `gstreplaygain.dll`, `gstriff-1.0-0.dll`, `gstrtp-1.0-0.dll`, `gstrtp.dll`, `gstrtpmanager.dll`, `gstrtsp-1.0-0.dll`, `gstrtsp.dll`, `gstsdp-1.0-0.dll`, `gstshapewipe.dll`, `gstsmpte.dll`, `gstsoup.dll`, `gstspectrum.dll`, `gstsubparse.dll`, `gsttag-1.0-0.dll`, `gsttypefindfunctions.dll`, `gstudp.dll`, `gstvideo-1.0-0.dll`, `gstvideobox.dll`, `gstvideoconvertscale.dll`, `gstvideocrop.dll`, `gstvideofilter.dll`, `gstvideomixer.dll`, `gstvideoparsersbad.dll`, `gstvideorate.dll`, `gstvideotestsrc.dll`, `gstvolume.dll`, `gstwaveform.dll`, `gstwavenc.dll`, `gstwavparse.dll`, `gstxingmux.dll`, `gsty4menc.dll`, `libgstnvdsseimeta.dll`, `libgstnvvideoconvert.dll`, `libnvbuf_fdmap.dll`, `libnvdsbufferpool.dll`, `libnvdsgst_customhelper.dll`, `libnvds_meta.dll`, `nvdsgst_helper.dll`, `nvdsgst_meta.dll`, `nvds_winport.dll`, `pthreadVC3.dll`, `pthreadVCE3.dll`, `pthreadVSE3.dll`
