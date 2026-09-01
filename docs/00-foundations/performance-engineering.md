# Performance Engineering

## What is Performance Engineering?

Performance engineering is the discipline of understanding system behavior, identifying dominant costs, and improving measurable outcomes under real constraints.

It is different from collecting optimization tricks.

An optimization is meaningful only when:

- the problem is measured;
- the cost model is understood;
- the change improves a relevant metric;
- the trade-offs are acceptable.

## Performance Is a System Property

A Go program is affected by multiple layers:

```text
Application Design
        ↓
Algorithms and Data Structures
        ↓
Go Compiler
        ↓
Go Runtime
        ↓
Operating System
        ↓
CPU and Memory Hardware
```

A problem observed at one layer may originate from another.

Example:

A CPU profile showing `runtime.scanobject` does not necessarily mean the runtime is slow. It may indicate excessive application allocation or pointer-heavy data structures.

## Optimization vs Engineering

Optimization asks:

> How can this code become faster?

Performance engineering asks:

> What is the dominant cost, and what is the best trade-off?

Examples:

Zero-copy:

Benefits:
- less copying;
- less allocation;
- less memory bandwidth.

Costs:
- ownership complexity;
- lifetime constraints;
- retention risks.

Lock-free:

Benefits:
- less blocking.

Costs:
- algorithm complexity;
- ABA;
- correctness verification;
- contention behavior.

## Core Principle

The best optimization is often removing unnecessary work.

Examples:

- inlining removes calls;
- escape analysis removes heap allocation;
- BCE removes bounds checks;
- dead-code elimination removes unused computation.

## Evidence Driven Development

A performance claim should follow:

```text
Observation
 ↓
Hypothesis
 ↓
Measurement
 ↓
Change
 ↓
Validation
```

Without evidence, a performance improvement is only a guess.
