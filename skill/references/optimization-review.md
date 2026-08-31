# Optimization Review Contract

Use this reference when reviewing or implementing a performance change that makes the code less obvious, relies on implementation behavior, changes ownership/lifetime, or could be accidentally "simplified" later.

## 1. Purpose

Performance code has a special maintainability failure mode:

> The optimized version can look worse than the slower version.

Examples:

```go
_ = b[7]
```

looks redundant.

```go
_ cpu.CacheLinePad
```

looks wasteful.

```go
out := append([]byte(nil), input[:n]...)
```

looks like an unnecessary allocation and copy.

```go
type node struct {
    left  uint32
    right uint32
}
```

may look less natural than `*node`.

A future maintainer can reasonably "clean up" these patterns and silently reintroduce a measured regression.

Therefore non-obvious performance invariants must be made recoverable from the code and its tests/benchmarks.

---

## 2. Maintainability is a hard gate

The performance workflow is:

```text
measure
↓
candidate change
↓
A/B confirms gain
↓
MAINTAINABILITY GATE
↓
component/system validation
↓
merge
```

Benchmark success alone does not satisfy the gate.

Ask:

1. Does the new code look redundant, strange, or over-engineered?
2. Does it depend on compiler/runtime/GC/CPU behavior?
3. Could a normal refactor remove the optimization unintentionally?
4. Does it alter ownership, aliasing, or lifetime?
5. Does it introduce a representation invariant?
6. Does it depend on a magic threshold, padding, shard count, or capacity cap?
7. Does correctness require special concurrency or memory assumptions?
8. Is there an existing benchmark or regression test that can preserve the reason?

If any answer is yes, document the invariant.

---

## 3. Do not comment every optimization

Do not create noise around ordinary idiomatic code.

Usually no special performance comment is needed for:

```go
items := make([]Item, 0, len(src))
```

or a clearly structured critical section:

```go
value := compute(input)

mu.Lock()
cache[key] = value
mu.Unlock()
```

The code already communicates intent.

Avoid comments like:

```go
// Preallocate for performance.
items := make([]Item, 0, n)
```

when nothing non-obvious is being preserved.

A comment is required when it protects knowledge that is **not recoverable from the local syntax**.

---

## 4. Required content of a performance comment

A non-obvious performance comment should answer the minimum relevant set:

### WHY

Why is the unusual form intentional?

### MECHANISM

Which cost is reduced?

Examples:

- redundant bounds checks;
- backing-array retention;
- GC scan work;
- cache-line bouncing;
- allocation churn;
- atomic contention;
- pointer chasing;
- copy/memory bandwidth.

### PRESERVATION

What apparently simpler change would bring the problem back?

### SAFETY CONTRACT

Required for unsafe, FFI, mmap, lock-free, lifetime-sensitive code:

- who owns the storage;
- how long it must remain valid;
- whether it may mutate;
- what synchronization protects it;
- whether a native caller may retain a pointer.

### EVIDENCE

When an existing benchmark/test exists and is stable, mention it by name or keep it nearby in the package.

Do not invent evidence references.

---

## 5. Comment quality

### Bad

```go
// Optimization.
_ = b[7]
```

```go
// Faster.
_ cpu.CacheLinePad
```

```go
// Avoid allocation.
return unsafe.String(...)
```

These comments do not preserve the rationale or safety contract.

### Good

```go
// Keep this bounds check: it proves len(b) >= 8 to the compiler,
// allowing the indexed reads below to eliminate redundant checks.
_ = b[7]
```

### Good - intentional copy

```go
// Copy the small result instead of retaining the entire input buffer.
// Returning input[:n] here can keep a multi-megabyte backing array alive.
out := append([]byte(nil), input[:n]...)
```

### Good - pool retention

```go
// Do not retain unusually large buffers in the pool. A single oversized
// request would otherwise increase the pool's retained memory footprint.
if cap(buf) <= maxPooledBuffer {
    pool.Put(buf[:0])
}
```

### Good - pointer-free representation

```go
// Child references are indices intentionally. Keeping node storage
// pointer-free reduces GC scanning and pointer chasing during bulk traversal.
type node struct {
    left  uint32
    right uint32
}
```

### Good - false-sharing padding

```go
type shard struct {
    counter atomic.Uint64

    // Keep independently written shard counters on separate cache lines.
    // Removing this padding can reintroduce false sharing under contention.
    _ cpu.CacheLinePad
}
```

---

## 6. When "Do not simplify" language is appropriate

Use explicit preservation wording only when a likely cleanup would be harmful.

Good:

```go
// Intentionally copy here. Do not replace this with input[:n]:
// doing so would retain the full backing array.
out := append([]byte(nil), input[:n]...)
```

Good:

```go
// Keep this access even though its value is unused. It supplies one
// bounds proof for the indexed reads below.
_ = b[7]
```

Do not use `DO NOT REMOVE` around ordinary idiomatic optimizations.

Excessive warning comments become noise and stop being trusted.

---

## 7. Representation changes need stronger documentation

The following often need a type-level or block-level comment, not only a line comment:

- pointer → index/offset;
- AoS → SoA;
- hot/cold split;
- sharded structures;
- unusual field ordering;
- cache-line padding;
- tagged/versioned state;
- custom object/slab arena.

Explain the invariant at the level where the representation is defined.

Example:

```go
// node uses integer child indices instead of pointers. The storage is scanned
// in bulk, so the compact pointer-free representation reduces GC scan work and
// improves locality. Index 0 is reserved as "no child".
type node struct {
    left  uint32
    right uint32
}
```

The comment should also document semantic conventions such as sentinel values.

---

## 8. Configuration tuning also needs rationale

Changes such as:

```text
GOGC
GOMEMLIMIT
pool maximum capacity
shard count
batch size
```

may not live next to algorithmic code, but still create performance policy.

Record:

- why the value exists;
- what workload was used;
- what operational limit it protects;
- whether it is a hard requirement or an initial tuning value.

Do not write false precision.

Bad:

```go
const shards = 64 // fastest
```

unless this is truly an invariant.

Better:

```go
// shardCount reduces write contention for the current high-QPS workload.
// Re-evaluate with the scaling benchmark when core count or traffic mix changes.
const shardCount = 64
```

If the exact number is intended to be configurable, prefer configuration over pretending it is a universal constant.

---

## 9. Benchmarks as living evidence

A benchmark can preserve *why* an optimization exists.

Examples:

```text
BenchmarkDecodeHeader
BenchmarkCounterScaling
BenchmarkTreeTraversal
BenchmarkBufferPool
```

A good performance comment may mention an existing benchmark by name:

```go
// Keep counters cache-line separated; BenchmarkCounterScaling covers the
// intended write-heavy workload.
```

Only mention benchmarks that actually exist.

Benchmarks should be representative enough that a future maintainer can compare:

```text
optimized representation
vs
simplified alternative
```

without reconstructing the original investigation from scratch.

---

## 10. Regression tests vs performance benchmarks

Some performance changes introduce correctness invariants.

Examples:

### Intentional copy

The important correctness property may be:

```text
result does not alias caller-owned mutable input
```

A normal test can protect that.

### Pointer/index representation

Tests should protect:

- sentinel semantics;
- stable lookup;
- bounds;
- mutation invariants.

### Lock-free

Use:

- race testing;
- stress tests;
- ABA-specific scenarios where relevant;
- invariant/linearizability-oriented testing.

A benchmark alone cannot prove correctness.

### Unsafe zero-copy

Tests should deliberately exercise:

- mutation;
- reuse;
- lifetime;
- empty values;
- boundary sizes;
- architecture differences where relevant.

---

## 11. Unsafe comment standard

Unsafe optimization comments have a higher bar.

A comment saying:

```go
// Zero-copy conversion.
```

is insufficient.

The local documentation must explain why the operation is safe.

Example:

```go
// bytesToString returns a zero-copy view of b.
// The backing bytes must remain immutable and must not be reused while
// the returned string is reachable.
func bytesToString(b []byte) string {
    if len(b) == 0 {
        return ""
    }
    return unsafe.String(unsafe.SliceData(b), len(b))
}
```

Review questions:

1. Who owns `b`?
2. Can another goroutine mutate it?
3. Can it return to a pool?
4. Can the string outlive the buffer's logical owner?
5. Can a tiny string retain a huge buffer?
6. Is the end-to-end benefit large enough to justify this contract?

If these cannot be answered clearly, do not use zero-copy.

---

## 12. mmap / FFI lifetime comments

For mmap/native memory, local comments should make lifetime ordering explicit where not obvious.

Conceptually:

```text
mapping created
↓
views may exist
↓
all views cease use
↓
unmap
```

For cgo:

```text
Go pointer given to C
↓
pinning/handle contract
↓
C stops retaining it
↓
unpin/delete handle
```

Do not depend on a maintainer remembering implicit lifetime rules.

---

## 13. Compiler trick comments

Examples requiring comments:

- `_ = b[N]` for BCE;
- deliberately shaped hot helper for devirtualization/inlining when otherwise surprising;
- unusual no-allocation API form;
- assembly declaration with `//go:noescape`.

Do not over-document normal code just because the compiler optimizes it.

For implementation-sensitive tricks, mention that the benefit must be reverified on Go upgrades when appropriate.

---

## 14. CPU/layout comment requirements

Comment when layout is intentionally non-obvious:

- cache-line padding;
- field separation;
- hot/cold split;
- compact fixed-width representation;
- false-sharing prevention;
- SoA layout.

The comment should name the physical cost:

```text
cache-line ownership
working set
pointer chasing
bandwidth
GC scan
```

instead of saying only "cache optimization".

---

## 15. Concurrency comment requirements

For sharding, batching, or single-writer architecture, document:

- ownership;
- which operations may run concurrently;
- how aggregation occurs;
- whether ordering is important;
- why a single global atomic/mutex was rejected if that is not obvious.

For custom lock-free code, documentation requirements are substantially higher:

- linearization point;
- progress guarantee;
- ABA handling;
- reclamation/reuse assumptions;
- retry/backoff behavior;
- correctness tests.

Custom lock-free code without this reasoning should not pass review.

---

## 16. Pool comments

For `sync.Pool`, document non-obvious retention rules.

Example:

```go
const maxPooledBuffer = 64 << 10

// Buffers above this threshold are intentionally dropped instead of pooled.
// Rare oversized requests must not raise steady-state retained memory.
```

If the threshold was empirically tuned, preserve the benchmark/workload context in a project performance note or benchmark.

Do not imply a universal Go runtime threshold.

---

## 17. Project-level performance notes

Create or update a project-level performance note when the optimization:

- changes a broad data model;
- changes ownership architecture;
- affects multiple packages;
- introduces a subsystem-wide memory limit/tuning policy;
- adds a custom lock-free structure;
- introduces significant unsafe/FFI assumptions;
- requires substantial benchmark context.

Local comments should stay concise and point to existing project documentation only when that document really exists.

Do not create a large design document for a one-line BCE proof.

---

## 18. Keep comments current

An obsolete performance comment is worse than no comment.

When changing the code:

1. re-run the benchmark/test that justified the invariant;
2. update or remove the comment if the mechanism no longer applies;
3. re-check version-sensitive compiler/runtime assumptions.

Example:

If a newer compiler eliminates the same bounds checks without `_ = b[N]`, remove the workaround **and** its comment after verification.

The goal is to preserve valid knowledge, not fossilize hacks forever.

---

## 19. Review checklist

Before accepting a non-obvious performance change:

### Evidence

- [ ] Is there a measured bottleneck?
- [ ] Is the candidate change linked to a clear cost model?
- [ ] Is A/B reproducible?
- [ ] Is the effect meaningful at the relevant system level?

### Maintainability

- [ ] Is unusual code explained adjacent to the invariant?
- [ ] Does the comment explain why, not merely what?
- [ ] Does it warn against a likely harmful simplification when needed?
- [ ] Are thresholds/config values documented without false universality?

### Correctness

- [ ] Does representation/ownership behavior have tests?
- [ ] For concurrency, are invariants and ownership clear?
- [ ] For unsafe/FFI, is the lifetime/safety contract explicit?
- [ ] Are race/checkptr/stress tools used where appropriate?

### Regression

- [ ] Is there a benchmark or measurable invariant that can catch regression?
- [ ] Are benchmark names/comments real and maintained?
- [ ] Will a future Go-version upgrade trigger revalidation if implementation-sensitive?

---

## 20. Definition of done for optimized code

A non-obvious optimization is done only when:

```text
measured benefit
+
correctness
+
clear invariant
+
appropriate local explanation
+
representative regression evidence
+
acceptable maintenance cost
```

Performance is not an excuse to make the code inexplicable.

The next maintainer should be able to answer:

> Why is this code intentionally unusual, what problem does it prevent, and how can I verify whether it is still needed?
