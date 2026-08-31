# 08. Technique Catalog

> 本索引用于快速定位技巧。它不是“看到模式就自动套招”的清单。

## 1. Recommendation Levels

- **Production Safe**
- **Conditional**
- **Library-Level**
- **Runtime Internal**
- **Historical**

---

## 2. Catalog

| Technique | Category | Typical Trigger | Main Benefit | Risk | Verification | Recommendation |
|---|---|---|---|---|---|---|
| Preallocate slice/map | Memory | growth/realloc hotspot | alloc/copy ↓ | low | benchmem | Production Safe |
| Append-style API | Compiler/Memory | temporary result allocations | heap churn ↓ | low | allocs + benchmark | Production Safe |
| Fix escape | Compiler | hot heap allocations | heap/GC ↓ | low | `-m` + bench | Production Safe |
| BCE guard | Compiler | bounds-heavy parser | checks ↓ | low | check_bce + asm | Production Safe |
| `_ = b[N]` BCE proof | Compiler | fixed-width decode | repeated checks ↓ | low | check_bce | Production Safe |
| PGO | Compiler | representative prod CPU profile | inline/devirt | low | A/B build | Production Safe |
| `[]T` instead of `[]*T` | CPU/GC | pointer-heavy sequence | locality/GC ↑ | medium semantics | benchmark/profile | Conditional |
| Pointer→index | CPU/GC | graph/tree scan hot | noscan/locality | model complexity | GC + CPU bench | Conditional |
| Hot/cold split | CPU | large struct hot subset | working set ↓ | extra pointer | benchmark/perf | Conditional |
| AoS→SoA | CPU | field-oriented batch loop | cache/bandwidth | model complexity | benchmark/perf | Conditional |
| Cache-line padding | CPU | false sharing | coherence ↓ | memory ↑ | scaling + perf | Conditional |
| Sharding | Concurrency | true sharing hotspot | contention ↓ | complexity | scaling | Production Safe/Conditional |
| Local accumulation | Concurrency | hot counters | atomic/lock ↓ | merge semantics | benchmark | Production Safe |
| Batching | Concurrency | high sync frequency | sync ops ↓ | latency/batching | benchmark | Production Safe |
| Single writer | Concurrency | mutable aggregate state | locks disappear | architecture | service benchmark | Conditional |
| Immutable atomic snapshot | Concurrency | read-mostly config | read path cheap | immutability | benchmark/race | Production Safe |
| RWMutex | Concurrency | long concurrent reads | read parallelism | reader counter | compare Mutex | Conditional |
| Lock-free CAS | Concurrency | proven lock bottleneck | nonblocking progress | ABA/retry | correctness + scaling | Library-Level |
| Tagged pointer | Lock-free | ABA | identity/version | implementation | proof/tests | Library/Runtime |
| Backoff | Lock-free | CAS retry storm | contention ↓ | tuning | retries + benchmark | Conditional/Library |
| `sync.Pool` | Memory | temp reusable objects | churn ↓ | retention | allocs/RSS | Conditional |
| Pool capacity cap | Memory | occasional giant buffers | retention ↓ | policy | heap/RSS | Production Safe |
| Copy small sub-slice | Memory | giant backing retention | live heap ↓ | extra copy | heap profile | Production Safe |
| `GOGC` tuning | GC | GC CPU vs memory trade | frequency control | RSS | metrics/bench | Conditional |
| `GOMEMLIMIT` | GC | fixed memory budget | soft memory control | thrash if low | metrics | Production Safe/Conditional |
| `GOGC=-1 + limit` | GC | maximize memory→CPU trade | GC frequency ↓ | pressure behavior | load test | Conditional |
| `FreeOSMemory` phase boundary | GC | big temporary phase ends | RSS release | forced GC | RSS/profile | Conditional |
| Pointer fields first | GC | scan-heavy large structs | scan prefix ↓ | layout | GC benchmark | Conditional |
| Manual slab/index arena | Memory | same-lifetime many objects | alloc/object graph ↓ | manual semantics | benchmark | Conditional |
| Experimental arena | Memory | bulk lifetime | bulk free | experimental | experiment | Experimental |
| Weak pointer | GC | non-owning cache | ownership root ↓ | nondeterminism | heap behavior | Conditional |
| `unsafe.String` bytes→string | Unsafe | copy proven hot | alloc/copy ↓ | lifetime alias | micro + system | Library-Level |
| `unsafe.Slice` native view | Unsafe | native/mmap memory | no copy | lifetime/alignment | tests | Library-Level |
| mmap | OS/Unsafe | large file-backed dataset | read/copy ↓ | lifetime/RSS | system benchmark | Library-Level |
| Pinner | FFI | C holds Go address | safe pin | lifecycle | correctness | Library-Level |
| cgo.Handle | FFI | C stores Go identity | safe handle | lifecycle | tests | Production/Library |
| `#cgo noescape` | FFI | hot C call no pointer retention | stack allocation | false contract | `-m` + tests | Library-Level |
| `#cgo nocallback` | FFI | hot C call never callbacks | boundary overhead ↓ | false contract | benchmark | Library-Level |
| `//go:noescape` | Compiler/Asm | assembly implementation | escape info | corruption if wrong | low-level tests | Library-Level |
| `//go:nosplit` for speed | Runtime | desire remove stack check | tiny local saving | severe | n/a | Do Not Recommend |
| `//go:linkname procPin` | Runtime | per-P tricks | contention/locality | compatibility | specialized only | Runtime Internal |
| manual noescape hack | Runtime | fool escape analysis | stack allocation | corruption | n/a | Never Recommend |
| Heap ballast | GC | historical GC frequency hack | GC frequency ↓ | VM/runtime hack | historical | Historical |
| reflect Header zero-copy | Unsafe | old conversion trick | copy ↓ | brittle | n/a | Historical |

---

## 3. Decision by Symptom

### GC CPU high

Check:

1. allocation rate；
2. live heap；
3. pointer density；
4. memory limit；
5. GC assist。

Candidate：

- fix escape；
- preallocate；
- pool；
- pointer→index；
- GOGC / GOMEMLIMIT。

### High-core scaling collapse

Check:

1. atomic hotspot；
2. mutex profile；
3. false sharing；
4. memory bandwidth。

Candidate：

- sharding；
- batching；
- local state；
- padding；
- single writer。

### Hot parser

Check:

1. BCE；
2. allocation；
3. interface dispatch；
4. data layout。

Candidate：

- len guard；
- `_ = b[N]`；
- append-style API；
- concrete hot path；
- PGO；
- carefully justified zero-copy。

### RSS high

Check:

1. live heap；
2. backing capacities；
3. scavenger；
4. fragmentation；
5. mmap/cgo。

Candidate：

- release retention；
- pool cap；
- copy tiny views；
- phase FreeOSMemory；
- fix external mapping lifecycle。
