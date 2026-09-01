# Cost Models

Performance work requires understanding what kind of cost is being reduced.

## CPU Cost Model

CPU execution cost includes:

- instructions;
- cycles;
- pipeline stalls;
- cache misses;
- branch misprediction;
- memory latency.

A shorter source program is not automatically faster.

## Memory Cost Model

Memory costs include:

- allocation;
- initialization;
- bandwidth;
- cache locality;
- retention;
- fragmentation.

A smaller logical object may still create worse access patterns.

## Garbage Collection Cost Model

Go GC cost is influenced by:

- allocation rate;
- live heap size;
- pointer density;
- object graph complexity;
- scan work;
- write barriers;
- GC assist.

Reducing allocations is only one possible strategy.

## Concurrency Cost Model

Concurrency costs include:

- lock contention;
- atomic cache-line ownership;
- scheduler interaction;
- goroutine blocking;
- synchronization frequency.

Changing primitives does not remove the underlying sharing problem.

## Compiler Cost Model

Compiler optimizations depend on what the compiler can prove.

Examples:

```text
unknown lifetime
        ↓
heap allocation

unknown bounds
        ↓
runtime check

unknown receiver
        ↓
indirect call
```

Many optimizations are about making invariants visible.
