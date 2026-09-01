# Go Performance Engineering Handbook

A detailed knowledge base for understanding and improving Go performance through cost models, compiler/runtime behavior, and reproducible evidence.

This documentation is intentionally separate from the Agent Skill:

- `docs/` explains the concepts and mechanisms.
- `skill/` defines the operational workflow for AI agents.
- `proofs/` contains minimal reproducible experiments.

## Reading Path

### Foundations

Start with:

- [Performance Engineering](./00-foundations/performance-engineering.md)
- [Cost Models](./00-foundations/cost-models.md)
- [Optimization Principles](./00-foundations/optimization-principles.md)

### CPU and Memory

- [Cache Hierarchy](./01-cpu-and-memory/cache-hierarchy.md)
- [Memory Locality](./01-cpu-and-memory/memory-locality.md)
- [Branch Prediction](./01-cpu-and-memory/branch-prediction.md)
- [TLB and Pages](./01-cpu-and-memory/tlb-and-pages.md)
- [Data Layout](./01-cpu-and-memory/data-layout.md)

## Documentation Philosophy

Performance is a system property.

A performance conclusion should connect:

```text
Claim
 ↓
Mechanism
 ↓
Cost Model
 ↓
Evidence
 ↓
Conditions
 ↓
Recommendation
```

The goal is not to collect tricks, but to understand why a change affects system behavior.