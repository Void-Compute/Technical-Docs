# Ghost JIT Stub Compilation System — Technical Report

This report documents Ghost's just-in-time stub compilation subsystem: the
runtime generation of hardware-specific C++ sources and their compilation into
native DLLs via the host's MSVC toolchain. This mechanism is what allows Ghost's
interceptors to be *tailored to the detected GPU* at launch time rather than
shipped as static binaries.

## 1. The Problem

Interception stubs must embed hardware-specific data: device names, memory
configurations, driver versions, and capability answers. A static stub binary
cannot know at build time which GPU it will encounter. Three approaches exist:

1. **Static stubs with lookup tables** — rigid, bloats with every supported GPU.
2. **Configuration files read at runtime** — adds parse/config failure modes to
   the interception hot path.
3. **Runtime compilation** — generate a stub specialized to the detected
   hardware, compile with the host's native toolchain, deploy.

Ghost uses approach 3. The resulting stubs contain the profile data as compiled
constants: no runtime parsing, no lookup overhead, and a traceable build
artifact per hardware configuration.

## 2. Solution Architecture

### 2.1 Toolchain Location

At startup, the orchestrator locates MSVC via:

1. `vswhere.exe` (installed with Visual Studio) → latest C++ workload instance
2. Environment `VCINSTALLDIR` if already present
3. Direct `cl.exe` PATH probe as fallback

The located toolchain is initialized through `vcvars64.bat`, which is invoked
via a generated wrapper batch (`ghost_build.bat`-style) so each compilation runs
in a correctly configured environment without polluting the parent shell.

### 2.2 Compilation Pipeline

Per stub, the pipeline is:

```
1. Generate <stub>.cpp (profile constants embedded as literals)
2. Write source to the .ghost working tree (nv_spoof/)
3. Locate MSVC (vswhere → vcvars64.bat, with direct cl.exe fallback)
4. Compile: cl.exe /LD /MT /O2 <stub>.cpp /Fe:<stub>.dll /link /MACHINE:X64 …
5. Version-check: compare output hash/mtime against deployed copy
6. Deploy to resolution path if new or changed
```

Compilation flags are deliberate: `/MT` (static CRT — stubs must not impose CRT
version requirements on the host process), `/O2` (these stubs sit on query paths
used per-frame by some applications), `/LD` (DLL target).

### 2.3 Compiled Artifact Inventory

| Artifact | Generated From | Purpose |
|---|---|---|
| `nvml.dll` | NVML stub template + profile constants | Management-surface identity (device name, VRAM, driver version) |
| `ghost_cuda_interop.dll` | Interop bridge source | External-memory/semaphore bridging (separate report) |
| Ghost Vulkan layer | Layer source + profile | Graphics-side identity configuration |
| CUPTI stub | CUPTI template | Profiling-surface neutralization |
| GFSDK stubs | Aftermath template | Diagnostics-surface neutralization |

Each artifact is a *specialized instance*: two machines with different GPUs
produce different binaries, and the embedded constants are the audit trail of
what identity each machine presented.

## 3. Deployment Mechanics

Stubs are deployed into the application's DLL resolution path. Isaac Sim-family
applications load with `LOAD_LIBRARY_SEARCH_SYSTEM32`, which restricts resolution
to System32 — so system-surface stubs are deployed there, with original files
backed up (`*.original`) before overwrite and restorable via `clean`.

Deployment is version-aware: a stub is only overwritten if the generated source
or build differs from the deployed copy, keeping repeat launches fast.

## 4. Diagnostics

| Log Entry | Meaning |
|---|---|
| `[GHOST] JIT Compiling <stub> …` | Generation started for detected profile |
| `[GHOST] <stub> Compiled and Cached` | Build succeeded, deployed |
| `Failed to locate cl.exe` | Toolchain missing — run `doctor` |
| `[GHOST] <stub> up to date` | Version check passed, recompile skipped |

## 5. Known Limitations

1. **Toolchain dependency** — the MSVC Build Tools are a hard requirement for
   first launch; Ghost cannot self-bootstrap a compiler. `doctor` provides the
   install link.
2. **Antivirus friction** — JIT-compiled, freshly written DLLs are a classic
   heuristic-detection pattern. Ghost's binaries are unsigned; users may need to
   whitelist the `.ghost` tree.
3. **Rebuild determinism** — MSVC output embeds timestamps; byte-identical
   rebuilds are not guaranteed. The version check uses source-change detection
   rather than output-hash equality.
4. **Parallel launch races** — two concurrent first-launches could race on the
   compile cache; the orchestrator serializes compilation under a lock.

## Summary

The JIT compilation subsystem converts hardware detection directly into
specialized native interceptors, trading a one-time toolchain dependency for
per-machine stubs that carry zero runtime configuration burden. The same
pipeline produces every Ghost interceptor — one mechanism, one audit trail, one
cleanup path.
