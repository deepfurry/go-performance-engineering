# 05. Memory / Allocation / GC Engineering

## 1. 核心模型

GC 优化不是“尽量少 GC”。

真正需要控制：

```text
allocation rate
live heap
scannable heap
pointer density
object lifetime
retention
RSS
```

总模型：

```text
Application allocations
      ↓
Allocator
      ↓
Heap growth
      ↓
GC pacing
      ↓
Mark / Scan
      ↓
Sweep
      ↓
Reusable heap
      ↓
Scavenger
      ↓
OS physical memory
```

---

## 2. Allocator

small allocations 通常经过：

```text
P-local mcache
 ↓
mcentral
 ↓
mheap
 ↓
OS
```

目标是让大部分 small allocation：

- 不碰 OS；
- 不碰全局锁；
- 走 per-P fast path。

因此：

> 单次 heap allocation 并不自动是严重性能问题。

---

## 3. Allocation 有三类成本

### Allocator CPU

创建对象本身的开销。

### GC Frequency

allocation bytes 消耗 heap headroom：

```text
allocation rate ↑
→ GC frequency ↑
```

### Scan / Retention

如果对象：

- 长期 live；
- pointer-rich；

每轮 GC 都要承担扫描成本。

---

## 4. Allocation Rate

非常粗略：

```text
GC frequency ≈ allocation rate / available heap headroom
```

因此：

```text
live heap unchanged
```

也可能因为：

```text
allocation churn ↑
```

导致 GC CPU 飙升。

---

## 5. Temporary Objects 不是免费

即使对象马上死亡：

```text
request
→ allocate
→ dead
```

它们仍然：

- 消耗 allocator CPU；
- 吃 heap headroom；
- 增加 GC cycle frequency；
- 可能触发 GC assist。

所以优化 allocation churn 有价值。

---

## 6. Heap → Stack

Escape analysis 成功后：

```text
heap object
→ stack value
```

收益不仅是少一次 malloc。

更重要：

```text
object exits entire heap/GC lifecycle
```

不再需要：

- heap tracking；
- marking；
- sweeping；
- heap object metadata。

---

## 7. Live Heap vs Scannable Heap

1 GiB：

```go
[]byte
```

和 1 GiB：

```go
[]*Node
```

不是同一种 GC workload。

`[]byte` backing array pointer-free：

```text
noscan
```

pointer graph：

```text
scan pointers
→ follow objects
```

GC 成本主要来自 marking/scanning，因此 pointer density 极其重要。

---

## 8. Pointer-Free Design

典型：

```go
type Node struct {
    Left  uint32
    Right uint32
}
```

配合：

```go
nodes []Node
```

替代：

```go
type Node struct {
    Left  *Node
    Right *Node
}
```

收益可能同时来自：

- object count ↓；
- pointer count ↓；
- GC scan ↓；
- cache locality ↑；
- TLB locality ↑。

---

## 9. Pointer 不只 `*T`

GC-visible pointer-bearing values 包括：

- string；
- slice；
- map；
- chan；
- interface。

所以：

```text
struct 没有显式 *T
```

不等于 noscan。

---

## 10. Struct Pointer Fields 前置

GC 可在 value 的最后一个 pointer 之后停止扫描。

因此 pointer-heavy / GC-heavy workload 中，可考虑：

```go
type T struct {
    Ptr1 *A
    Name string

    LargeNonPointerData [1024]byte
}
```

而不是把 pointer 放在最后。

### 注意

Go 1.26 默认 Green Tea GC 后 collector 内部 scanning 机制已发生演进。

因此这属于：

```text
Production Safe
+
Conditional Benefit
```

必须基于当前 Go 版本 profile / benchmark。

---

## 11. Green Tea GC

Go 1.26 默认启用 Green Tea GC。

重点改善：

- small-object marking/scanning locality；
- CPU scalability；
- GC-heavy workload overhead。

这意味着：

> 旧 Go 版本上的 pointer-small-object benchmark 不能直接推导现代 Go 的收益。

但 pointer density 与 object graph complexity 的基本成本模型仍然存在。

---

## 12. GOGC

现代模型：

```text
Target heap =
Live heap +
(Live heap + GC roots) * GOGC / 100
```

因此：

```text
GOGC ↑
→ GC frequency ↓
→ memory headroom ↑
```

本质：

```text
CPU ↔ memory
```

trade-off。

---

## 13. GOMEMLIMIT

`GOMEMLIMIT` / `debug.SetMemoryLimit` 是 runtime soft memory budget。

它和 GOGC 不是同一维度：

```text
GOGC
→ proportional GC pacing

GOMEMLIMIT
→ absolute runtime-memory pressure
```

典型固定容器：

```text
container = 8 GiB
GOMEMLIMIT < 8 GiB
```

应预留：

- cgo；
- mmap；
- external library；
- kernel / non-Go memory；
- operational margin。

---

## 14. `GOGC=-1 + GOMEMLIMIT`

```go
debug.SetGCPercent(-1)
debug.SetMemoryLimit(limit)
```

用途：

```text
disable ordinary proportional GC trigger
+
let memory limit become dominant pressure
```

适合：

- fixed memory budget；
- high throughput；
- memory can be traded for CPU。

必须 benchmark tail latency 与 memory pressure behavior。

---

## 15. GC Thrashing

如果：

```text
live heap ≈ memory limit
```

而大部分数据真的是 live：

```text
GC
→ little reclaimed
→ allocate
→ GC again
```

形成 thrashing。

### Rule

如果数据是 live：

> 更频繁的 GC 无法解决 live-set size。

需要：

- 提高 limit；
- 减少 retained/live data。

---

## 16. GC Pacer 与 Assist

GC 与 application 并发运行。

如果 application allocation 太快，background marking 跟不上，allocating goroutine 可能被要求承担 mark assist。

因此：

```text
small STW
```

不等于：

```text
GC 对 tail latency 没影响
```

现代 GC latency 还包括：

- assist；
- GC CPU competition；
- pointer write barrier；
- root scanning。

---

## 17. Write Barrier

mark phase 期间 application 仍在修改 pointer graph：

```go
a.Next = b
```

runtime 必须保证 concurrent GC 不漏掉新的 reachable edge。

因此 pointer-heavy mutation workload 还可能承担 write-barrier cost。

pointer→index 可以同时减少：

- scan；
- write barrier；
- object graph complexity。

---

## 18. Mark vs Sweep

Mark：

> 哪些对象 live？

Sweep：

> dead slots 重新变成 allocator 可用。

Go 当前 sweeping 相比 scanning 通常不是主要 GC CPU 成本。

优化优先级通常：

```text
mark / scan
```

高于：

```text
sweep micro-optimization
```

---

## 19. GC Reclaim ≠ RSS Release

对象死亡：

```text
unreachable
↓
GC marks dead
↓
sweep makes memory reusable
```

但：

```text
runtime still owns pages
```

所以 RSS 不一定马上下降。

这不是自动意味着 memory leak。

---

## 20. Scavenger

Scavenger 负责把不再需要的 physical pages 返回 OS。

完整：

```text
dead object
↓
sweep
↓
free heap space
↓
free pages
↓
scavenger
↓
OS
```

因此：

```text
HeapAlloc ↓
RSS unchanged
```

需要继续检查：

- scavenging；
- fragmentation；
- stacks；
- runtime metadata；
- mmap/cgo。

---

## 21. `debug.FreeOSMemory`

适合 phase transition：

```text
startup build/index
↓
large temporary dataset dies
↓
steady-state
```

此时可强制 GC + 尽量 release OS memory。

不适合：

```text
steady server every minute
```

定时调用。

那是在与 runtime pacer / scavenger 对抗。

---

## 22. Fragmentation

span：

```text
live dead live dead live
```

即使 dead slots 可复用，只要 span 仍有 live objects，就不一定能整页释放。

所以：

```text
live heap
```

与：

```text
physical RSS
```

不存在精确 1:1。

---

## 23. Size Classes

small object 会向上匹配 allocator size class。

因此：

```text
struct size +1 byte
```

有时可能跨 size-class boundary，带来超过 1 byte 的真实 heap footprint 变化。

只在以下场景值得主动优化：

```text
millions of same type
+
heap profile shows large footprint
```

不要为了 size class 到处扭曲业务 struct。

---

## 24. Tiny / Large Allocations

现代 Go allocator 对 tiny pointer-free object 有额外优化。

而较大的 allocation 会走不同路径。

这些 boundary 属于 implementation detail：

- 了解成本模型；
- 不应硬编码业务设计到某一版本 threshold。

---

## 25. `sync.Pool`

适合：

```text
temporary reusable objects
```

例如：

- buffers；
- encoders；
- temporary scratch。

收益：

- allocation rate ↓；
- GC frequency ↓。

### 不是永久 Cache

runtime 可以在 GC 周期清理 pool。

---

## 26. Pool Poisoning / Retention

普通 buffer：

```text
64 KiB
```

偶发：

```text
32 MiB
```

如果把巨型 capacity 放回 pool：

```text
later tiny requests retain huge backing storage
```

因此常用 retention policy：

```go
if cap(buf) <= maxPooled {
    pool.Put(buf[:0])
}
```

---

## 27. Slice Backing-Array Retention

```go
buf := make([]byte, 1<<30)
small := buf[:100]
return small
```

逻辑 100 bytes，却可能保活整个大 backing array。

解决：

```go
small := append([]byte(nil), buf[:100]...)
```

这说明：

> 增加一次 allocation/copy 可以是更好的内存优化。

---

## 28. Capacity 是 Memory Ownership

```go
buf := make([]byte, 0, 64<<20)
```

`len=0` 并不意味着 footprint=0。

`cap` 对 retention 非常重要。

---

## 29. Preallocation

```go
xs := make([]Item, 0, n)
```

可以同时减少：

- realloc；
- copy；
- allocation bytes；
- GC churn。

Map 也可合理 hint capacity。

### 反面

巨大过度 preallocation：

- retained heap ↑；
- RSS ↑。

---

## 30. Manual Slab / Index Arena

```go
type Arena struct {
    nodes []Node
}
```

逻辑 object 用 index：

```text
many heap objects
→ one/few backing arrays
```

适合：

- common lifetime；
- parser；
- graph；
- temporary query state；
- ECS。

这仍然是普通 Go heap-managed storage，不等于真正 manual free。

---

## 31. Experimental Arenas

`GOEXPERIMENT=arenas` 类型能力属于实验功能。

最终 Skill 应区分：

```text
slice/index slab
→ production conditional

experimental arena package
→ experimental
```

---

## 32. Weak Pointer

适合：

> cache 本身不应该拥有对象 lifetime。

例如：

- canonicalization；
- dedup；
- optional reuse。

不适合替代：

- TTL；
- LRU；
- max-byte cache policy。

GC timing 不应承担业务 eviction semantics。

---

## 33. Cleanup / Finalizer

外部资源优先确定性释放：

```text
Close
defer Close
```

Cleanup 可以作为：

- native memory；
- mmap；
- external resource fallback。

Finalizer 更应视为 legacy / specialized mechanism。

不要把 GC 当 destructor scheduler。

---

## 34. Heap Ballast

历史技巧：

```go
var ballast = make([]byte, 10<<30)
```

机制：

```text
large long-lived noscan object
→ live heap artificially ↑
→ GOGC target ↑
→ GC frequency ↓
```

依赖 virtual-memory behavior，属于 GC tuning hack。

现代 Go 应优先：

- `GOMEMLIMIT`；
- `GOGC`；
- profile-driven tuning。

分类：

```text
Historical
```

---

## 35. Memory Diagnostic Matrix

### GC CPU 高 + live heap 小

优先：

```text
allocation churn
allocs profile
escape
preallocation
reuse
```

### Live heap 大 + pointer 少

优先：

```text
retention / capacity / memory budget
```

而不是 pointer scanning trick。

### Live heap 中等 + pointer graph 很复杂

优先：

```text
flatten
pointer→index
[]T
noscan
```

### Memory limit binding

检查：

```text
reclaim ratio
live set
thrashing
```

### HeapAlloc 降但 RSS 不降

检查：

```text
scavenger
fragmentation
mmap/cgo
stacks
retained capacity
```

---

## 36. Skill Rules

1. 单次 allocation 不自动昂贵。
2. 同时看 alloc count 和 allocation bytes。
3. Allocation rate 会直接影响 GC frequency。
4. Heap→stack 让对象退出整个 GC lifecycle。
5. Live heap ≠ scannable heap。
6. Pointer-free data 可大幅降低 GC work。
7. string/slice/map/interface 也包含 GC-visible pointer。
8. Pointer-field layout 属于 conditional optimization。
9. 现代 Go 版本必须重新 benchmark GC tricks。
10. GOGC 与 GOMEMLIMIT 是不同旋钮。
11. Memory limit 太低会 thrash。
12. GC latency 不等于 STW。
13. Heap reclaimed 不等于 RSS returned。
14. FreeOSMemory 主要用于 phase boundary。
15. slice/string view 要检查 backing retention。
16. Copy 有时比 zero-copy 更省内存。
17. Pool 必须有 retention policy。
18. Preallocation 不能无限过度。
19. Manual slab/index 是生产可用条件技巧。
20. Weak pointer 不替代业务 cache policy。
21. deterministic resource release 优先。
22. Ballast 只保留为历史案例。
