# Ghost Environment Orchestration — Technical Report

This report documents the environment orchestration subsystem: the environment
variables, Python import hooks, registry configuration, and cleanup machinery
that place the host application inside a correctly configured runtime before its
first instruction executes.

## 1. The Problem

Three runtimes must be tuned before a CUDA-first application initializes
correctly on AMD hardware:

1. **The HIP/ROCm runtime** — must accept the detected GPU generation
   (unsupported RDNA revisions require mapping onto compatible targets).
2. **The Python/CUDA ecosystem** — frameworks probe `torch.cuda.is_available()`
   and refuse to initialize on non-NVIDIA answers, before any driver query runs.
3. **The Windows device-identity surface** — registry-resident device data is
   consulted by management tools and some installers.

Each tuning must be applied *before* application start, scoped *exactly* to
Ghost's own keys, and *reversibly* — the boundary between "configured" and
"modified the user's system" must be explicit and cleanable.

## 2. Solution Architecture

### 2.1 HIP/ROCm Environment Layer

Set per detected GPU generation (Report 02 profiles):

| Variable | Purpose |
|---|---|
| `HSA_OVERRIDE_GFX_VERSION` | Maps unsupported RDNA revisions onto compatible ROCm targets (e.g. 11.0.0 / 10.3.0 / 10.1.0 / 9.0.x per series) |
| `HIP_VISIBLE_DEVICES` / `ROCR_VISIBLE_DEVICES` | Device ordinal selection |
| `HSA_ENABLE_SDMA` | SDMA engine behavior per generation quirks |

### 2.2 CUDA-Side Environment Layer

| Variable | Purpose |
|---|---|
| `CUDA_VERSION` / `NVIDIA_VISIBLE_DEVICES` | Satisfies environment probes in container-style frameworks |
| `GHOST_ZLUDA_DLL` / `GHOST_ZLUDA_DIR` | Pins the ZLUDA binary location for components that resolve it |
| `ZLUDA_DUMP_DIR` | Forces ZLUDA trace mode — every CUDA call through the translation layer is dumped |

### 2.3 Vulkan Environment Layer

| Variable | Purpose |
|---|---|
| `MESA_VK_IGNORE_CONFORMANCE_WARNING` | Unblocks stacks that refuse non-conforming drivers |
| `DISABLE_LAYER_AMD_SWITCHABLE_GRAPHICS_1` | Prevents AMD switchable-graphics layer from rerouting enumeration |
| `VK_LOADER_DEBUG` | Loader diagnostics for trace capture |

These are gated by render-mode: the graphics-identity configuration is applied
or skipped per launch mode (see the mode-split architecture in the planning
materials).

### 2.4 Python Import Hooks

A `sitecustomize.py` layer is injected onto the Python path before the host
framework initializes. Its function is narrow: ensure `torch.cuda.is_available()`
reflects the translated stack's reality for frameworks that probe *before*
driver initialization. This is not a behavioral override of PyTorch — it is a
correct answer delivered at the one moment the standard driver query is not yet
positioned to answer.

### 2.5 The Registry Guard

Ghost manages exactly one registry subtree (`SOFTWARE\NVIDIA Corporation` keys)
to present device identity to management surfaces. The guard implements:

- **Scoped writes** — only keys within the managed subtree are touched
- **Creation tracking** — every created key/value is journaled at deploy time
- **Automatic restore** — `clean` removes journaled keys and restores the
  pre-Ghost view
- **No cross-contamination** — keys existing before Ghost are never modified;
  the guard refuses to touch keys it did not journal

## 3. Cleanup and Restore

The `clean` command is the reversibility contract, and its completeness is a
design requirement, not a convenience:

```
1. Remove all journaled registry keys (guard)
2. Restore backed-up original DLLs (*.original suffixes)
3. Purge stubs from all deployment targets (System32, app folders)
4. Purge the .ghost working tree (stubs, caches, generated sources)
5. Report anything non-restorable (should be nothing)
```

The restore path exists because system-level interception without a clean exit
is malware behavior. Ghost's distinction from malware is not the interception —
it is the journal, the scope, and the exit.

## 4. Diagnostics

| Signal | Meaning |
|---|---|
| Env trace block at launch | All variables Ghost set, with values — the full environment contract |
| `Registry Guard: created key …` | Journal entry at deploy time |
| `clean: removed N journaled keys` | Restore executed and counted |
| `sitecustomize: torch probe served` | Import hook engaged for the host framework |

## 5. Known Limitations

1. **Environment inheritance** — variables set in the launcher shell propagate to
   child processes; non-Ghost shells launched from the same console inherit the
   tuned environment (by design, but worth knowing).
2. **Registry scope** — the guard manages its subtree only; applications that
   write their own device keys elsewhere are outside its journal.
3. **Import hook timing** — frameworks that probe CUDA through native code
   before Python initializes bypass the hook; the driver-level identity surface
   (Report 02) covers those cases instead.
4. **Concurrent Ghost instances** — two Ghost-managed processes with different
   profiles must not share a registry-journal scope; the launcher serializes
   deployment to prevent journal divergence.

## Summary

Environment orchestration is the quiet layer: no interception, no binary
patching — just the correct runtime answers delivered at the correct moment,
with every change journaled and every journaled change reversible. It is the
layer that makes the rest of Ghost removable, and removability is what
distinguishes an interoperability environment from a modification.
