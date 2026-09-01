# Optimization Principles

## 1. Measure Before Optimize

Source patterns are not bottlenecks.

Allocation, mutexes, interfaces, copies, and atomics are only candidates.

## 2. Remove Work Before Making Work Faster

The highest-value optimizations often eliminate work:

- stack allocation instead of heap allocation;
- bounds-check elimination;
- dead-code elimination;
- constant propagation;
- inlining.

## 3. Optimization Moves Costs

Optimization rarely removes all costs.

It usually moves them.

Examples:

```text
copy cost
↓
ownership complexity
```

or:

```text
blocking cost
↓
algorithm complexity
```

## 4. Maintainability Is Part of Performance

Non-obvious optimizations create future maintenance risk.

Important invariants should be preserved through:

- comments;
- benchmarks;
- tests;
- design notes.

## 5. Prefer Mechanisms Over Tricks

A good optimization explains:

- what mechanism changes;
- why it helps;
- when it applies;
- when it does not.
