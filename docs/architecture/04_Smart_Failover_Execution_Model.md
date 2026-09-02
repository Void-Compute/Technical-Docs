# Ghost Smart Failover Execution Model — Technical Report

This report documents Ghost's dual-backend execution model: how applications are
launched natively on ROCm first, monitored, and relaunched through the ZLUDA
translation layer on failure — and the Waiting Room interface that manages the
user experience during long initialization phases.

## 1. The Problem

AMD applications have two viable execution backends, with an inverted
performance/compatibility trade-off:

| Backend | Performance | Compatibility |
|---|---|---|
| ROCm native | Maximum (native HIP kernels) | Poor for CUDA-first applications — unmapped kernels, missing extensions, silent breakage |
| ZLUDA translation | Good (translation overhead) | High — the full CUDA driver surface is emulated |

The correct backend is application-dependent and only observable by running the
application. A static choice is therefore wrong for a general-purpose launcher:
the decision must be made *empirically, per launch*, with automatic recovery.

## 2. Solution Architecture

### 2.1 The Launch Sequence

```
1. Validate target (script/exe/bat), classify type
2. Environment: HIP/ROCm variables set, PATH ordered
3. Primary launch: native ROCm path
4. Monitor: log streaming + exit-code watch
5. On crash/incompatibility signal:
     → relaunch with ZLUDA translation layer injected
     → Waiting Room TUI engages for the (longer) translation init
6. On success: control hands to the application
```

### 2.2 Crash Classification

Ghost watches the child process for distinguishable failure classes:

- **Hard exit** (non-zero return, immediate) → ZLUDA relaunch
- **Log-signatured failure** — known incompatibility strings in the streamed
  application log (e.g. kernel-missing errors, backend init failures)
- **Hang detection** — no log progress within configured windows

Each class is trace-logged with the triggering evidence, so the failover
decision is auditable after the fact.

### 2.3 Process Management

The child process handle is stored in an atomic (`CHILD_PID`) so the Ctrl+C
handler can terminate the full process tree on user interrupt — including the
stub-deployed child, not just the launcher shell. Log streaming continues after
exit so final messages (success or crash dumps) are visible.

## 3. The Waiting Room Interface

Large AI applications (Stable Diffusion, LLM servers) spend minutes loading
models before producing a window. The Waiting Room converts this dead time into
a managed experience:

- **Live telemetry** — VRAM, temperature, GPU load, polled and displayed in the
  TUI
- **DOOM integration** — a fully functional DOOM port runs inside the shell
  (key: `D`), because initialization waiting time is real time
- **Background music** — toggleable stream (key: `M`)
- **Port watch** — Ghost polls for the application's local web port; the
  millisecond it opens, control hands over to the application. The handoff is
  the completion signal — no fixed timeouts.

The Waiting Room is itself an exercise in resource discipline: it runs on the
same GPU under load, so its telemetry path is intentionally lightweight (registry
and API polling, no rendering beyond text).

## 4. ZLUDA Trace Mode

Failover diagnostics depend on visibility into the translation layer. Ghost
forces ZLUDA's trace mode (`ZLUDA_DUMP_DIR`) on every launch:

- All CUDA/NVML calls through ZLUDA are dumped to `%TEMP%\zluda`
- The dump cross-references the Ghost trace log (which captures the same calls
  at the router level)
- Together, the two logs distinguish *router-level* decisions (Ghost's) from
  *translation-level* behavior (ZLUDA's) — a two-level diagnostic split that
  localizes failures to one layer or the other

## 5. Diagnostics

| Signal | Meaning |
|---|---|
| Primary launch succeeds | ROCm native path is compatible — done |
| Relaunch with ZLUDA | Native path failed; translation path engaged (logged with reason) |
| Waiting Room engaged | Application initializing; port watch active |
| Crash after ZLUDA relaunch | Translation-layer incompatibility — ghost_trace + ZLUDA dump required |

## 6. Known Limitations

1. **Failover is per-launch, not per-call** — a mid-session failure after native
   success does not hot-swap backends; the application must be relaunched.
2. **Signature lists require maintenance** — log-signature detection only knows
   failures it has seen; novel failures fall back to generic crash detection.
3. **Waiting Room port detection** — applications that open no local port get no
   automatic handoff; manual focus switch is required (documented behavior).
4. **GPU contention** — Waiting Room telemetry polls the GPU under application
   load; on stressed systems the polling itself is measurable overhead
   (configurable).

## Summary

The dual-backend execution model turns a per-application configuration problem
into a per-launch empirical decision, with automatic recovery and full
diagnostic capture. The Waiting Room converts the resulting initialization
latency into a managed experience instead of a frozen prompt — on the principle
that the user's time during model loading is as much a part of the software as
the model itself.
