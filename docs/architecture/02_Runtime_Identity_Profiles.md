# Ghost Runtime Identity Profiles — Technical Report

This report documents the runtime identity subsystem: how the Ghost Environment
presents a stable, coherent CUDA device identity to host applications running on
AMD RDNA/Vega hardware, and how per-series attribute mapping keeps capability
queries consistent across GPU generations.

## 1. The Problem

CUDA applications rarely stop at `cudaGetDeviceCount`. Before accepting a device,
they typically probe a battery of identity and capability surfaces:

- `cuDeviceGetName` — marketing name used in UI, logs, and feature gates
- `cuDeviceGetAttribute` — 100+ capability attributes (shared-memory limits,
  cluster support, memory pools, cooperative launch, …)
- `cuDeviceGetUuid` / `cuDeviceGetLuid` — stable identifiers used for
  cross-API device matching (graphics↔compute association)
- `cuDriverGetVersion` / `cuDeviceGetAttribute(CUDA_DRIVER_VERSION)` —
  feature-tier gating

On a translated device (ZLUDA → HIP), these queries return values reflecting the
*real* AMD hardware. This is correct information, but it forms an identity that
some host applications treat as invalid: unknown vendor strings, missing
capability flags, or attribute combinations that no NVIDIA driver would ever
report. The result is not a crash — it is a silent feature downgrade or an
outright device refusal.

## 2. Solution Architecture

The identity subsystem operates at one control point — the `cuGetProcAddress`
router inside `void_shim.dll` (deployed as `nvcuda.dll`) — and resolves identity
queries at **priority 1 (Identity responses)**, ahead of any translation-layer
forwarding. Applications therefore receive profile-correct answers regardless of
the underlying translation state.

### 2.1 Compatibility Profiles

Each supported AMD GPU series maps to a **compatibility profile**: a
three-part tuple of (driver mask version, device name class, attribute override
set), selected at startup from the detected hardware:

| AMD Host Series | Mask Version | Compatibility Profile |
|:---|:---|:---|
| RX 9000 Series | 11.0.0 | GeForce RTX 5090 class |
| RX 7000 Series | 11.0.0 | GeForce RTX 4090 class |
| RX 6000 Series | 10.3.0 | GeForce RTX 3090 Ti class |
| RX 5000 Series | 10.1.0 | GeForce RTX 2080 Ti class |
| Radeon VII / MI50 | 9.0.6 | Tesla V100 class |
| Vega 64 / Vega 56 | 9.0.0 | Tesla P100 class |

Profile selection is driven by PCI device identification at startup; new cards
are one mapping-table entry.

### 2.2 The Attribute Mapping Rules

Attribute spoofing follows three rules, ordered by safety:

1. **Pass through** when the real value is architecturally valid and
   host-application compatible (e.g. warp size 32, max threads per block).
2. **Map to profile value** when the real value is AMD-specific but the profile
   defines a coherent NVIDIA-class equivalent (shared-memory limits, L2 size).
3. **Report truthfully and let the host fall back** when spoofing would promise
   behavior the hardware cannot deliver (e.g. memory-pool support attribute 115:
   reported honestly as 0; NVIDIA Warp then logs "mempool not supported" and
   uses its own allocator — graceful degradation by design).

Rule 3 is load-bearing: an honestly reported missing capability triggers clean
fallback paths in mature applications, whereas a falsely advertised capability
produces corruption deep inside optimized kernels.

### 2.3 Compute-Capability Selection

The profile's reported compute capability (e.g. `sm_75`) doubles as the JIT
target hint for runtime compilers (NVRTC, Warp). Older ISA targets are
deliberately chosen: they are the best-supported translation targets and produce
kernels that run on the widest hardware range.

## 3. Cross-API Coherence

Device identity is presented consistently across API surfaces:

- **CUDA API** — name/attributes/UUID via the router's identity priority.
- **Vulkan/DXGI enumeration** — governed by the graphics-side configuration,
  which is independently switchable (the CUDA compute path does not read Vulkan
  enumeration results).
- **NVML/NVSMI surface** — the JIT-compiled `nvml.dll` reports the profile name
  and memory configuration to management-tool queries.

This separation allows each API surface to present the identity appropriate to
its consumers without cross-contamination.

## 4. Diagnostics

Every identity response is traceable. In `ghost_trace.log`:

| Trace Entry | Meaning |
|---|---|
| `cuDeviceGetName -> 0` | Profile name served |
| `[ATTR] cuDeviceGetAttribute (ID:n Real:x->Spoofed:y)` | Attribute override applied (real value logged for audit) |
| `[ATTR] (ID:n Real:x->Spoofed:x)` | Pass-through per rule 1 |
| `cuDeviceGetUuid -> 0` | Stable UUID served |
| `cuDriverGetVersion -> 0` | Driver version surface served |

The `Real:→Spoofed:` pairing in attribute traces is the audit trail: every
deviation from hardware truth is recorded with both values.

## 5. Known Limitations

1. **Profile coarseness** — per-series profiles apply one attribute set to all
   cards in a series; SKUs with unusual memory configurations inherit the series
   profile's memory values.
2. **New attribute surface** — CUDA runtime updates add capability attributes
   faster than profiles are extended; unmapped attributes follow rule 3
   (pass-through) by default.
3. **UUID stability** — served UUIDs are deterministic per boot configuration,
   not persistent across driver reinstalls; applications that persist UUIDs
   across Ghost reinstalls will observe identity changes.
4. **Cross-API identity mixing** — because graphics and compute identity layers
   are independently switchable, applications correlating Vulkan and CUDA device
   identities (via LUID/UUID matching) must be configured consistently. Ghost's
   launcher applies a coherent set by default.

## Summary

The identity subsystem converts the "unknown device" problem into a
configuration problem: each AMD GPU series presents a coherent, internally
consistent NVIDIA-class identity whose every deviation from hardware truth is
trace-logged and auditable, and whose unspoofable capabilities degrade host
applications gracefully instead of catastrophically.
