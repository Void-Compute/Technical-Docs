# Ghost Diagnostics and Trace Infrastructure — Technical Report

This report documents Ghost's trace infrastructure: the per-call logging design,
the log taxonomy, and the reason the trace log doubles as the canonical bug-report
format for the project's community testing program.

## 1. The Problem

A multi-layer interception stack (router → identity → translation → runtime)
produces failures that cross layer boundaries: a symptom in the application may
originate in the translation layer, the identity layer, the environment, or the
host's own assumptions. Debugging such stacks traditionally requires triaging
*which layer to blame* before any fix can start — the most expensive step.

Ghost solves this at design time rather than debug time: every layer logs its
decisions into **one chronological, structured log**, in a taxonomy that makes
layer attribution a pattern-matching exercise.

## 2. Solution Architecture

### 2.1 Design Rules

1. **Every decision logs** — identity responses, forwarding decisions, overrides,
   fallbacks: if the router chose a path, the choice and its inputs are in the log.
2. **Before/after pairing** — spoofed values log both the real and served values
   (`Real:x->Spoofed:y`), creating a permanent audit trail.
3. **Caller attribution** — entries record the calling module and return address,
   so behavior can be attributed to specific host components.
4. **One file, chronological** — all layers append to one trace stream; layer
   attribution comes from the entry prefix, not from separate files.

### 2.2 Log Taxonomy

| Prefix | Layer | Meaning |
|---|---|---|
| `[GHOST]` | Router | Driver-API call served by Ghost (identity/forward decision) |
| `[FWD_RAW]` | Router | Call forwarded to ZLUDA untranslated |
| `[ZLUDA_OK]` | Translation | ZLUDA call returned success (with duration) |
| `[ATTR]` | Identity | Device attribute response, with `Real:→Spoofed:` values |
| `[HOOK]` | Injection | Interceptor installation events |
| `[EXPORT_TBL]` | Router | Export-table query handling |
| `[CUDA-INTEROP]` | Bridge | External-memory/semaphore bridge events |
| `[VK]` | Graphics layer | Vulkan enumeration/creation interception |
| `[ERROR]` | Any | Failure with returning layer |

### 2.3 A Worked Example

A single application frame produces an interleaved trace:

```
[GHOST] cuInit -> 0                        (router: init served)
[ZLUDA_OK] cuDevicePrimaryCtxRetain …      (translation: context acquired)
[ATTR] cuDeviceGetAttribute (ID:75 Real:0->Spoofed:7)   (identity: override)
[ATTR] cuDeviceGetAttribute (ID:115 Real:0->Spoofed:0)  (identity: honest pass)
[CUDA-INTEROP] cuImportExternalMemory: HIP returned 0   (bridge: import OK)
```

Reading this sequence, a debugger knows: init succeeded, the context is real,
two attribute overrides applied (one capability honestly absent), and an
external memory import succeeded — four layers, one frame, zero ambiguity about
which layer served which answer.

## 3. The Trace Log as Bug-Report Format

Community beta testers span many GPUs, driver versions, and application
workloads. Traditional bug reports ("it crashed, I think") are unusable across
that variance. Ghost's answer: **the trace log is the report.**

Testers run with tracing always enabled and submit `ghost_trace.log` plus the
application log. Because every layer self-describes its decisions, the maintainer
can determine from the log alone:

- Which layer failed (router / translation / bridge / graphics)
- Which specific call or query triggered it
- What the real hardware values were at that moment

This converts community testing from a support burden into a data pipeline — and
it is the reason Ghost's beta program functions across a hardware matrix the
maintainer does not own.

## 4. Diagnostics — The Meta Layer

Because the trace describes the diagnostic system itself, the project's
diagnostic checklists are tables of *log patterns* rather than procedures.
Examples (full checklists in the domain reports):

| Pattern | Interpretation |
|---|---|
| Router entries absent entirely | Interception did not engage — deployment/resolution issue, not logic |
| `[ZLUDA_OK]` for init but `[ATTR]` missing after | Identity layer not engaged for that module |
| Bridge entries absent while CUDA calls continue | Host not attempting external-memory sharing (expected in some modes) |
| Repeated identical error from one caller | Deterministic host-side failure — layer is serving consistently |

## 5. Known Limitations

1. **Volume** — per-call tracing on kernel-heavy workloads produces large logs;
   rotating/critical-only modes exist but change the debugging contract.
2. **Hot-path overhead** — trace calls on per-frame paths are measurable on
   stressed systems; the tracer supports quiet operation for performance runs.
3. **Multi-process correlation** — traces are per-process; correlating a launcher
   and child requires the process-id field in each entry.
4. **Clock skew** — timestamps are per-process; cross-machine correlation relies
   on UTC wall-clock lines, not the monotonic timer.

## Summary

The trace infrastructure is not an accessory to Ghost — it is the reason a
multi-layer stack built by one developer can be operated by a community.
Design-time logging converts every future failure into a self-describing
artifact, and it is the component that scales the project beyond its author.
