# Ghost Developer Tools — Technical Report

This report documents the operator tools exposed by the Ghost shell: `doctor`,
`benchmark`, `translate`, `install-deps`, and `clean`. These commands exist
because a single-operator interoperability project must make diagnosis,
measurement, source migration, and reversal self-service — every hour spent
manually debugging a user's environment is an hour the project cannot afford.

## 1. `doctor` — Environment Diagnosis

A read-only sweep of every prerequisite and runtime surface Ghost depends on:

| Check | Method | Failure Guidance |
|---|---|---|
| MSVC toolchain (`cl.exe`) | `vswhere` + PATH probe | Links the Visual Studio Build Tools installer (C++ workload) |
| HIP SDK presence | Component path/version probe | Points to the AMD installer; checks version against the profile matrix |
| GPU driver surface | Driver version query per Report 02 | Generation-specific guidance |
| ZLUDA presence | Hash-consistent cache probe | Offers `install-deps` |
| Registry state | Ghost-managed key journal check | Offers `clean` if drift detected |
| Python surface | Import hook reachability | Hook placement guidance |

`doctor` performs no modifications — it is the triage tool, and its output is
designed to be pasted into a Discord support request verbatim.

## 2. `benchmark` — Real Throughput Measurement

Measures the GPU's actual FP16/FP32 throughput through the *active* compute
backend (ROCm native or ZLUDA), rather than reporting marketing numbers:

- Kernel selection is deliberately simple and bandwidth/ALU-bound
- Results are per-precision, per-backend — which makes the ROCm-vs-ZLUDA
  performance delta *quantifiable* (input to the smart-failover design, Report 04)
- Used by the community to sanity-check driver/ROCm updates: a benchmark delta
  is a regression signal independent of any application

## 3. `translate` — CUDA-to-HIP Source Migration

Wraps HIPIFY (Perl-based) to convert CUDA C++/Python sources to native HIP:

```
translate <folder>
  → locates toolchain (Strawberry Perl + HIPIFY scripts, auto-managed)
  → walks the source tree, converts CUDA constructs to HIP equivalents
  → reports unmapped constructs per file
```

This is the escape hatch from translation entirely: sources migrated to HIP run
natively on ROCm at full speed, no interception involved. For projects whose
sources are available, `translate` is the permanent solution; the runtime
translation layer exists precisely for the cases where sources are not
available.

## 4. `install-deps` — Deterministic Provisioning

Re-runs the full dependency acquisition pass (ZLUDA, HIP SDK components,
toolchain checks, compatibility stub generation) with hash/version checking.
Exists because dependency state drifts — driver updates, ROCm upgrades, disk
cleanups — and re-provisioning must be one command, not an archaeology project.

## 5. `clean` — The Reversibility Contract

Full removal of every Ghost modification:

1. Registry keys from the guard journal (Report 05)
2. Original DLLs restored from `*.original` backups
3. Stubs purged from all deployment targets (System32, application folders)
4. The `.ghost` working tree removed (stubs, caches, generated sources)

`clean` is the layer that makes everything else ethical: system-level
interception with a complete, tested, single-command exit. The purge is scope-checked (Ghost-journaled targets only) and reports anything
non-restorable instead of failing silently.

## 6. Design Principles Across the Toolset

| Principle | Implementation |
|---|---|
| Self-service over support | Every tool outputs paste-ready diagnostics |
| Measurement over marketing | `benchmark` reports measured, per-backend, per-precision numbers |
| Migration over interception | `translate` offers the permanent (native HIP) path whenever sources exist |
| Reversibility over persistence | `clean` is tested as a feature, not an afterthought |
| Idempotence | Every tool is safe to re-run; provisioning is hash-checked |

## Summary

The toolset exists because interoperability software fails in user-specific ways,
and the only scalable support model is one where users can diagnose, measure,
migrate, and reverse — without the developer in the loop. Every tool encodes a
support conversation that happened once and was never repeated.
