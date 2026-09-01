# Cache Hierarchy

## Overview

Modern CPUs use multiple levels of storage:

```text
Registers
 ↓
L1 Cache
 ↓
L2 Cache
 ↓
Last Level Cache
 ↓
DRAM
```

The cost difference between these levels makes memory access patterns critical.

## Cache Lines

CPUs transfer memory in cache-line sized units.

A single field access may load neighboring data.

This creates opportunities:

- spatial locality;
- sequential access;
- compact layouts.

## Temporal Locality

Recently accessed data is likely to be reused.

Keeping hot data small improves cache reuse.

## Cache Coherence

On multi-core systems, cores must maintain a consistent view of memory.

Frequent writes to shared cache lines create coherence traffic.

This is the foundation behind:

- true sharing;
- false sharing.

## Engineering Implication

Performance is often determined by:

> How data is accessed, not only what algorithm is used.
