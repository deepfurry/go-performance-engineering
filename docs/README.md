# Go Performance Engineering Handbook

This handbook explains Go performance engineering from first principles.

It is intentionally different from the Agent Skill in this repository:

- `docs/` explains mechanisms, cost models, trade-offs, and engineering reasoning;
- `skill/` turns those ideas into an operational workflow for AI agents;
- `proofs/` contains small experiments that reproduce specific performance behaviors.

The handbook is written for engineers who want to understand **why** a Go program behaves the way it does, not only which command or optimization technique to use.

## Scope

The handbook studies performance across several layers:

```text
Application design
        ↓
Algorithms and data structures
        ↓
Go compiler
        ↓
Go runtime
        ↓
Operating system
        ↓
CPU and memory hardware
```

Performance problems often cross these boundaries.

For example:

- excessive allocation may appear as GC CPU;
- shared counters may appear as atomic hot spots but originate from cache coherence;
- interface abstraction may appear as call overhead but the larger loss may be missed inlining;
- a tiny slice may retain a very large backing array;
- a lock-free algorithm may consume more CPU than a mutex under contention.

The goal of this handbook is to provide the cost models needed to reason about those effects.

## Reading Order

### 00 — Foundations

Start here before studying individual techniques.

- [Performance Engineering](./00-foundations/performance-engineering.md)
- [Cost Models](./00-foundations/cost-models.md)
- [Optimization Principles](./00-foundations/optimization-principles.md)

These chapters define the vocabulary used throughout the rest of the handbook.

### 01 — CPU and Memory

- [Cache Hierarchy](./01-cpu-and-memory/cache-hierarchy.md)
- [Memory Locality](./01-cpu-and-memory/memory-locality.md)
- [Branch Prediction](./01-cpu-and-memory/branch-prediction.md)
- [TLB and Pages](./01-cpu-and-memory/tlb-and-pages.md)
- [Data Layout](./01-cpu-and-memory/data-layout.md)

These chapters explain how Go data structures map onto real hardware.

### 02 — Concurrency

- [Synchronization Model](./02-concurrency/synchronization-model.md)
- [Mutex](./02-concurrency/mutex.md)
- [Atomic Operations](./02-concurrency/atomic.md)
- [RWMutex](./02-concurrency/rwmutex.md)
- [Channels](./02-concurrency/channels.md)
- [Lock-Free Algorithms](./02-concurrency/lock-free.md)
- [The ABA Problem](./02-concurrency/aba.md)
- [Ownership-Oriented Design](./02-concurrency/ownership-design.md)

These chapters explain why concurrency performance is fundamentally about shared state, cache coherence, waiting, retry, and ownership.

## How to Read Performance Claims

A useful performance claim should connect:

```text
Observation
    ↓
Mechanism
    ↓
Cost Model
    ↓
Evidence
    ↓
Conditions
    ↓
Trade-off
```

Statements such as:

> "Atomics are faster than mutexes."

or:

> "Zero-copy is always better."

are not useful without workload conditions.

A performance result should answer:

- What cost changed?
- Under what workload?
- What new cost was introduced?
- How was the effect measured?
- Does the result still hold on the target Go version and hardware?

## Relationship to Proofs

The handbook explains mechanisms.

The `proofs/` tree should demonstrate those mechanisms with minimal reproducible experiments.

For example:

```text
docs/03-compiler/bounds-check-elimination.md
        ↓ explains

proofs/compiler/bounds-check-elimination/
        ↓ demonstrates
```

A proof is not expected to show a universal speedup. Its job is to demonstrate that the claimed behavior can be reproduced under a documented environment.

## Relationship to the Agent Skill

The Skill should not duplicate the handbook.

The handbook answers:

> Why does this behavior exist?

The Skill answers:

> When should an agent investigate this behavior, and what evidence is required before changing code?

Keeping those roles separate allows the Skill to remain compact while preserving detailed technical knowledge in this handbook.

## Version Sensitivity

Compiler and runtime behavior changes over time.

Implementation-sensitive claims involving:

- inlining;
- escape analysis;
- bounds-check elimination;
- allocator paths;
- GC internals;
- runtime synchronization;
- architecture-specific code generation;

must be revalidated on the target Go toolchain.

Hardware-sensitive claims must also be interpreted in the context of the target CPU, operating system, and workload.

## Principle

The central principle of this handbook is simple:

> Optimize measured system behavior, not code cleverness.
