# Go Performance Engineering

Evidence-driven Go performance engineering with documentation, Agent Skill, and reproducible proofs.

## Overview

This repository focuses on understanding and improving Go performance through evidence.

It is not a collection of isolated optimization tricks.

The project covers:

- compiler and SSA optimization;
- CPU cache and memory layout;
- concurrency and synchronization;
- allocation and garbage collection;
- unsafe and runtime boundaries;
- benchmarking and profiling methodology.

## Repository Structure

```text
.
├── docs/
│   Complete performance engineering documentation
│
├── skill/
│   Agent Skill package
│
└── proofs/
    Minimal reproducible experiments
```

## Documentation

The `docs/` directory contains the complete knowledge base.

Start here:

- [Go Performance Engineering Handbook](./docs/README.md)

## Agent Skill

The `skill/` directory contains the AI-oriented performance workflow.

It provides:

- diagnostic routing;
- evidence requirements;
- optimization risk rules;
- maintainability checks;
- validation workflow.

Start here:

- [SKILL.md](./skill/SKILL.md)

## Proofs

The `proofs/` directory contains minimal experiments for specific performance behaviors.

A proof demonstrates:

> This mechanism exists and can be reproduced under documented conditions.

It does not claim:

> This optimization is always faster.

See:

- [Proof Guidelines](./proofs/README.md)

## Philosophy

```text
Measure
  ↓
Understand the cost model
  ↓
Make the smallest justified change
  ↓
Benchmark
  ↓
Validate system impact
  ↓
Preserve the reason for the optimization
```

## License

Apache License 2.0