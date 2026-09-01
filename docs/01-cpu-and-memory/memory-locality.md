# Memory Locality

## Contiguous Data

A slice of values:

```go
[]T
```

stores objects together.

Benefits:

- better cache locality;
- fewer pointer dereferences;
- better prefetch behavior.

## Pointer Chasing

Pointer-heavy structures:

```text
Node
 ↓
Next
 ↓
Next
```

create dependent memory accesses.

The CPU cannot easily predict future addresses.

Costs:

- cache misses;
- TLB pressure;
- reduced memory-level parallelism.

## []T vs []*T

Neither is universally faster.

Consider:

- object size;
- mutation;
- identity;
- traversal pattern;
- GC impact.

## Working Set

The amount of frequently accessed data determines whether caches can help.

Reducing hot working sets can improve performance without changing algorithms.
