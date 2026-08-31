---
name: go-performance-engineering
description: Evidence-driven Go performance engineering for CPU, memory, GC, allocation, compiler/SSA, concurrency, cache/layout, lock-free, unsafe, cgo, and runtime-boundary work. Use for performance investigations, optimization reviews, benchmark/profiling plans, and implementation changes that must be measured, justified, documented, and regression-guarded.
---

# Go Performance Engineering

Use this skill to diagnose or implement Go performance work.

The goal is not to collect clever tricks. Identify the real cost, make the smallest justified change, prove the result, and preserve maintainability.

## Non-negotiable rule

Never infer a bottleneck from source shape alone.

An allocation, mutex, interface, bounds check, copy, atomic, pointer, or GC cycle is a **candidate**, not proof of a performance problem.

Use this loop:

```text
Goal
→ Baseline
→ Classify
→ Measure
→ Cost-model hypothesis
→ Candidate change
→ Representative A/B
→ Maintainability gate
→ System validation
→ Regression guard
```

Read `references/methodology.md` for open-ended investigations.

---

## Reference router

Load only the references needed for the current bottleneck.

| Need | Reference |
|---|---|
| Investigation workflow, evidence, stop conditions | `references/methodology.md` |
| Benchmark/profiling design | `references/benchmarking-profiling.md` |
| Cache, TLB, locality, false sharing, AoS/SoA, branch, bandwidth | `references/cpu-memory.md` |
| Atomic, Mutex, RWMutex, channel, contention, lock-free, ABA | `references/concurrency.md` |
| Escape, inline, BCE, SSA, devirtualization, PGO | `references/compiler.md` |
| Allocation, GC, GOGC, GOMEMLIMIT, retention, Pool, scavenger | `references/memory-gc.md` |
| Unsafe, zero-copy, KeepAlive, Pinner, cgo, mmap, directives | `references/unsafe-runtime.md` |
| Comment/documentation/review contract | `references/optimization-review.md` |
| Technique risk/recommendation lookup | `references/technique-catalog.md` |
| Command lookup | `references/tool-reference.md` |
| Go-version-sensitive behavior | `references/version-notes.md` |
| Primary Go sources | `references/official-sources.md` |

---

## Evidence levels

```text
L0  Source inspection
    Theoretical candidate only.

L1  Profile / compiler / runtime diagnostics
    pprof, runtime/metrics, -m, BCE diagnostics, trace evidence.

L2  Reproducible microbenchmark
    Representative repeated A/B + benchstat.

L3  Component/service benchmark
    Realistic concurrency, lifecycle, and traffic shape.

L4  Production/canary
    Real traffic and operating constraints.

L5  Hardware evidence
    perf/PMU for cache, TLB, branch, bandwidth, coherence.
```

Raise the evidence threshold as risk rises.

Typical expectations:

- idiomatic preallocation: L1-L2;
- layout/sharding: L2-L3;
- PGO: representative L3/L4 profile;
- unsafe zero-copy: L2 + L3, preferably L4;
- custom lock-free: L2/L3 plus correctness evidence;
- runtime-private tricks: do not recommend for normal applications.

---

## Symptom routing

### CPU high

Start with CPU profiling. If runtime GC/allocation functions are hot, trace the application cause rather than treating runtime as the root problem.

### CPU low but throughput low

Prioritize blocking, serialization, IO, mutex/block profiles, and execution trace.

### Allocation / GC high

Separate:

```text
allocation churn
live heap
scannable heap
memory-limit pressure
GC assist
retention
```

Use allocs profile for churn; heap/inuse for retained live memory.

### High-core scaling collapse

Check:

```text
true sharing
false sharing
atomic RMW
mutex contention
memory bandwidth
NUMA/topology
```

Never infer atomic/lock-free superiority from single-thread results.

### Tail-latency spikes

Use execution trace or FlightRecorder when temporal causality matters. Small STW does not rule out GC assist, scheduler delay, blocking, or syscalls.

### Compiler hot path

Escalate only as needed:

```text
-m
→ BCE diagnostics
→ SSA
→ assembly
```

### CPU microarchitecture

Use perf/PMU only after Go-level evidence is insufficient. Generic cache-miss counts alone do not prove false sharing.

---

## Prefer source-level cost removal

Prefer:

```text
heap escape
→ fix lifetime/API/representation

allocation churn
→ preallocate / append-style API / reuse

true sharing
→ shard / batch / local accumulation / single writer

false sharing
→ layout / padding

pointer-heavy traversal
→ flatten / compact / index representation

compiler cannot prove safety
→ provide a safe proof before using unsafe

copy proven hot
→ zero-copy only after ownership/lifetime review
```

For severe contention, redesign sharing before repeatedly swapping synchronization primitives.

For GC pressure, identify churn vs live set vs pointer scan vs retention vs memory-limit pressure before tuning GOGC/GOMEMLIMIT.

---

## Maintainability gate - mandatory

Benchmark success is not enough.

Read `references/optimization-review.md` whenever a change:

- looks redundant or unusual;
- depends on compiler, GC, allocator, runtime, cache-line, or CPU behavior;
- intentionally copies data where a view looks simpler;
- relies on padding, unusual field order, index/offset representation, SoA, or hot/cold layout;
- introduces pool capacity limits, sharding, batching, CAS/backoff, or custom lock-free logic;
- changes ownership/lifetime semantics;
- uses `unsafe`, mmap, cgo, assembly, compiler directives, or runtime boundaries;
- could be "simplified" later and silently reintroduce the regression.

### Required comment content

For non-obvious performance invariants, document the relevant parts:

```text
WHY
Why is this unusual form intentional?

MECHANISM
What measured cost does it reduce?

PRESERVATION
What tempting simplification would reintroduce the problem?

SAFETY CONTRACT
For unsafe/FFI/lifetime code: what ownership/lifetime rules make it correct?
```

Do not add comments that merely restate obvious idiomatic code.

Bad:

```go
// Optimization.
// Faster.
// Avoid allocation.
```

Good comments preserve the invariant and let the next maintainer know why the code must not be casually simplified.

### Stronger rule for unsafe

Unsafe/FFI comments must explain correctness and lifetime, not merely performance.

If the ownership/lifetime contract cannot be explained clearly, do not introduce the optimization.

---

## Evidence preservation

When practical, preserve the reason for an optimization with:

- a representative benchmark;
- allocation benchmark/invariant;
- scaling benchmark;
- correctness regression test;
- race/stress/ABA tests for concurrency;
- an existing project performance note for architectural changes.

A comment may name an existing benchmark/test.

Never invent documentation links or benchmark names.

Broad representation/ownership changes should be documented at the nearest stable project-level design/performance document in addition to local comments.

---

## Benchmark requirements

Prefer `testing.B.Loop` for new benchmarks.

A benchmark must model the relevant production behavior:

- input-size distribution;
- branch distribution;
- hit/miss ratio;
- mutation/reset behavior;
- read/write ratio;
- concurrency topology;
- cache warm/cold behavior;
- object lifetime.

For synchronization changes, measure a scaling curve across multiple workers/GOMAXPROCS values.

Run repeated A/B samples and use benchstat.

Record:

- Go version;
- GOARCH;
- CPU;
- GOMAXPROCS;
- relevant flags/GOEXPERIMENT;
- commit.

Do not treat race/checkptr/sanitizer timings as production performance.

---

## Validation

Progressively validate:

```text
microbenchmark
→ component
→ service/end-to-end
→ production/canary when appropriate
```

Track guardrails, not only ns/op.

Examples:

- allocation: B/op, allocs/op, GC CPU, P99;
- contention: scaling, CPU efficiency, P99;
- layout: system throughput, optionally PMU;
- unsafe zero-copy: end-to-end gain plus lifetime/correctness tests.

If system-level benefit is negligible and complexity rises materially, prefer reverting.

---

## Unsafe/runtime escalation

Read `references/unsafe-runtime.md` before proposing unsafe/runtime-boundary work.

Default order:

```text
safe Go
→ compiler optimization
→ safe restructuring
→ public unsafe API
→ specialized FFI/assembly contract
```

Do not automatically recommend:

- `//go:nosplit` for speed;
- `//go:linkname` to runtime-private symbols;
- private `runtime.procPin`;
- manual noescape tricks;
- runtime tagged-pointer assumptions.

Runtime source explains cost models; it is not automatically a public performance API.

---

## Version-sensitive rule

Re-verify implementation-sensitive optimizations on the target Go toolchain:

- escape;
- inline;
- BCE;
- devirtualization;
- generic lowering;
- allocator paths/thresholds;
- GC behavior;
- cgo overhead;
- architecture-specific code generation.

Historical benchmark results are hypotheses unless they match the target toolchain closely enough.

Read `references/version-notes.md`.

---

## Stop conditions

Stop or reject the optimization when:

- no measured hotspot exists;
- theoretical maximum gain is too small;
- the benchmark cannot represent the real workload;
- microbenchmark gain does not propagate to component/system behavior;
- complexity/unsafe/maintenance risk exceeds expected benefit;
- a required guardrail regresses;
- the change depends on undocumented runtime/compiler behavior without an explicit low-level maintenance commitment.

A valid result is:

> Do not optimize this path.

---

## Definition of done

A significant performance change is complete only when:

1. objective and baseline are known;
2. bottleneck has evidence;
3. a cost model explains the change;
4. A/B results are reproducible;
5. gain matters at the appropriate system level;
6. guardrails remain acceptable;
7. non-obvious code passes the maintainability gate;
8. unsafe/FFI code documents its safety/lifetime contract;
9. appropriate regression protection exists.

Optimize measured system behavior, not cleverness.
