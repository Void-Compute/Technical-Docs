# Ghost System Architecture — Technical Report

The AMD Ghost Environment is a runtime interoperability layer that enables
CUDA-first AI and rendering applications to execute on AMD RDNA/Vega hardware. This
report describes the layered component model, the deployment architecture, and the
design principles that govern every Ghost subsystem. Component-level detail is
provided in the companion reports (02–08).

## 1. The Problem

Modern AI and simulation frameworks (PyTorch, Warp, Isaac Sim, PhysX) are written
against the NVIDIA CUDA ecosystem: they link `nvcuda.dll`, query device identity,
require vendor-specific libraries (`nvml.dll`, `nvapi64.dll`), and probe runtime
attributes before accepting a device. AMD provides capable hardware and a mature
HIP runtime, but no drop-in translation surface for the full CUDA driver API, and
the applications themselves frequently gate their device selection on vendor
identity checks rather than on actual capability.

The gap is therefore threefold:

1. **API translation** — CUDA driver/runtime calls must execute as HIP calls.
2. **Identity coherence** — every subsystem that probes device identity must
   receive a consistent answer.
3. **Environment correctness** — runtimes (ROCm/HIP, Vulkan, Python) must be
   tuned per GPU generation before the application initializes.

## 2. Solution Architecture

Ghost is organized as five cooperating layers. Deployment is additive: components
are installed beside or over the application's resolution path and are fully
removable via the `clean` command.

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 5 — Host application (Isaac Sim, PyTorch, Warp, …)   │
├─────────────────────────────────────────────────────────────┤
│  Layer 4 — Execution shell                                   │
│    run_ai launcher · smart failover · Waiting Room TUI       │
├─────────────────────────────────────────────────────────────┤
│  Layer 3 — Identity & compatibility surface                  │
│    void_shim.dll (as nvcuda.dll) · nvml stub · nvapi stub    │
│    device identity profiles · attribute mapping              │
├─────────────────────────────────────────────────────────────┤
│  Layer 2 — Translation                                       │
│    ZLUDA (CUDA → HIP) · HIP SDK · HSA overrides              │
├─────────────────────────────────────────────────────────────┤
│  Layer 1 — AMD platform                                      │
│    ROCm/HIP runtime · AMD Vulkan/GL drivers · RDNA/Vega HW   │
└─────────────────────────────────────────────────────────────┘
```

### 2.1 The Orchestrator (`ghost_amd.exe`)

A single Rust binary (`main.rs`, ~8.5k lines) provides:

- **Dependency management** — acquisition and cache management of ZLUDA, HIP SDK
  components, Strawberry Perl and HIPIFY, with hash-consistent local layout
  (`.ghost/` directory tree).
- **JIT compilation subsystem** — runtime generation and MSVC compilation of
  hardware-specific stub DLLs (Report 03).
- **Identity subsystem** — compatibility identity profiles mapped per GPU series
  (Report 02).
- **Execution shell** — `(ghost amd)>` prompt hosting application launch, smart
  failover, and the Waiting Room interface (Report 04).
- **Diagnostics** — per-call trace infrastructure and log taxonomy (Report 06).
- **Lifecycle management** — `clean`/`purge` for full reversibility (Report 05).

### 2.2 Deployment Model

All interceptors are deployed into the application's DLL resolution path
(`System32` for components loaded under `LOAD_LIBRARY_SEARCH_SYSTEM32`, the
application directory otherwise). Deployment is idempotent: existing files are
version-checked, backed up when originals exist, and restorable. No Ghost
component persists after `clean`.

### 2.3 The Router (void_shim.dll)

The central control point is `void_shim.dll`, deployed as `nvcuda.dll`. It hosts
the `cuGetProcAddress` master router, through which every CUDA symbol resolution
in the host application passes. The router priority chain:

```
0. Self-reference check (recursive resolution guard)
1. Identity responses (cuDeviceGetName, capability attributes, …)
2. WELD overrides (locally implemented CUDA functions — e.g. the
   external-memory/semaphore bridge, documented separately)
3. Forward to ZLUDA (the general CUDA surface)
4. NOP fallback (return CUDA_SUCCESS for benign no-op queries)
```

This structure decouples the three concerns: identity answering, capability
bridging, and translation are independent priorities in one resolution path, and
each can evolve without affecting the others.

## 3. Design Principles

| Principle | Implementation |
|---|---|
| **Additive deployment** | All changes are new files or scoped registry keys; `clean` restores the pre-Ghost state |
| **Per-component kill switches** | Every subsystem can be disabled via environment or launcher flags without rebuilds |
| **Trace-first diagnostics** | Every interception, spoof, and forwarding decision emits structured trace output; the trace log is the canonical bug-report format |
| **Honest capability reporting** | Attributes that cannot be safely spoofed are reported truthfully, letting host applications take their own fallback paths (e.g. Warp reporting "mempool not supported" and continuing) |
| **Host transparency** | Host applications observe standard CUDA semantics and standard responses; no host modification is required |
| **Determinism over cleverness** | Where ambiguity exists (unmapped attributes, unknown queries), Ghost prefers stable, documented answers over dynamic behavior |

## 4. Known Limitations

1. **Single-GPU model** — the identity subsystem assumes one logical device;
   multi-GPU hosts are supported by device-ordinal mapping, not per-device
   profiles.
2. **Driver-generation coverage** — identity profiles are maintained per GPU
   series (Vega through RDNA3); unmapped cards fall back to the closest series
   profile.
3. **Windows-first** — the Windows deployment path is the hardened one; the Linux
   build is present but not at feature parity.

## Summary

Ghost is a layered interoperability environment in which translation (ZLUDA),
identity (profiles and the router), and orchestration (the shell) are independent,
individually replaceable components. The layered model is the reason the system
survives host-application updates: each layer adapts independently, and the
interface contracts between layers are few and explicit.
