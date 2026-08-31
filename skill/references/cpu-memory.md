# 02. CPU Cache / Memory Layout

## 1. 核心模型

CPU 不按照 Go variable 或 struct field 搬运数据，而是按照 cache line 和 page 工作。

```text
Registers
   ↓
L1
   ↓
L2
   ↓
L3 / LLC
   ↓
Memory Controller
   ↓
DRAM
```

性能重点不是死记延迟数字，而是理解数量级和访问模式。

---

## 2. Cache Line

主流 amd64 上通常以 64-byte cache line 为核心理解模型。

读取：

```go
x := obj.Value
```

并不是只“拿回一个 uint64”。

CPU 通常把包含它的整个 cache line 拉入 cache。

因此同一 line 上的数据可以形成：

- 优秀 spatial locality；
- 或严重 false sharing。

---

## 3. Spatial Locality

```go
type Point struct {
    X float64
    Y float64
}

points := []Point{...}
```

内存近似：

```text
X Y | X Y | X Y | X Y
```

连续遍历：

```go
for i := range points {
    sum += points[i].X
}
```

一次 cache line fetch 会顺带带入多个相邻对象。

### Rule

如果 workload 是顺序批量扫描，优先考虑：

- contiguous slice；
- compact struct；
- flat representation。

---

## 4. Pointer Chasing

```go
type Node struct {
    Value uint64
    Next  *Node
}
```

遍历：

```text
load Node0
   ↓
read Next
   ↓
才能知道 Node1 地址
   ↓
load Node1
```

这形成 dependent load chain。

成本不仅是“一次 pointer dereference”：

- cache miss；
- hardware prefetch 失效；
- memory-level parallelism 下降；
- TLB pressure；
- GC pointer scanning。

### 高风险结构

- linked list；
- pointer-heavy tree；
- graph；
- `[]*T` 大量随机对象；
- chained hash structure。

---

## 5. Hardware Prefetcher 与 MLP

连续：

```text
addr
addr+8
addr+16
addr+24
```

CPU 容易预测未来地址，并提前拉取 cache lines。

因此顺序扫描一个远大于 L1/L2 的 slice 仍然可能很快。

而 linked list：

```text
next address unknown until current node loaded
```

很难提前发起足够多 independent memory requests。

### Rule

优先优化：

```text
data layout
+
access pattern
```

而不是手工 software prefetch。

runtime / assembly 级别可以使用 prefetch，但普通 Go 应用通常不应主动复制这种技巧。

---

## 6. TLB 与 Page Locality

虚拟地址访问还需要：

```text
Virtual Page
   ↓
TLB
   ↓
Physical Page
```

连续 slice：

- 同一 page 可容纳大量 element；
- TLB translation 可重复利用。

随机 pointer graph：

- 跨大量 pages；
- 更容易 TLB miss；
- cache miss 与 TLB miss 可能叠加。

### Compact Representation 的额外收益

把对象从 32B 压缩到 16B，不只是省内存：

```text
objects/cache-line ↑
objects/page ↑
TLB coverage ↑
cache capacity ↑
bandwidth ↓
```

因此：

> struct size 是性能参数，不只是 RSS 参数。

---

## 7. Working Set

hot loop 真正频繁访问的数据构成 working set。

如果 working set：

```text
fits L1
```

通常极快。

逐步超过：

```text
L1 → L2 → LLC
```

就会增加 eviction 和更慢层级访问。

因此某些优化即使：

- 增加一个 pointer；
- 增加一个 allocation；

只要显著缩小 hot working set，也可能更快。

---

## 8. Hot / Cold Splitting

Before：

```go
type Conn struct {
    FD      int
    State   uint32
    Flags   uint32

    Username string
    Address  string
    Headers  map[string]string
    Debug    *DebugInfo
    Metadata map[string]any
}
```

hot path 只使用：

```text
FD
State
Flags
```

After：

```go
type Conn struct {
    FD    int
    State uint32
    Flags uint32
    cold  *connCold
}

type connCold struct {
    Username string
    Address  string
    Headers  map[string]string
    Debug    *DebugInfo
    Metadata map[string]any
}
```

收益：

- hot working set 变小；
- cache pollution 降低。

代价：

- pointer；
- 额外 allocation；
- cold path dereference。

### Recommendation

Conditional。必须 benchmark。

---

## 9. `[]T` vs `[]*T`

### `[]T`

```text
T T T T T T
```

优势：

- contiguous；
- cache-friendly；
- 少 pointer chasing；
- 通常少 GC scan；
- 可能少 allocation。

### `[]*T`

```text
ptr ptr ptr ptr
 ↓   ↓   ↓
random objects
```

可能增加：

- pointer chasing；
- cache miss；
- TLB；
- GC；
- allocator object count。

但 `[]*T` 在以下场景仍有合理性：

- stable identity；
- mutation semantics；
- 对象非常大且避免复制；
- polymorphic ownership。

### Rule

不要机械替换，必须同时考虑：

```text
CPU locality
GC
allocation
mutation semantics
identity
```

---

## 10. AoS vs SoA

### AoS

```go
type Entity struct {
    X, Y   float32
    VX, VY float32
    HP     uint32
    Name   string
}
```

```text
X Y VX VY HP Name | X Y VX VY HP Name
```

适合：

> 经常按对象读取多个字段。

### SoA

```go
type Entities struct {
    X, Y   []float32
    VX, VY []float32
    HP     []uint32
    Name   []string
}
```

适合：

> 经常对某一两个字段批量处理。

### Rule

```text
object-oriented access → AoS often better
field-oriented batch access → SoA candidate
```

不是“上 SoA 就高级”。

---

## 11. False Sharing

```go
type Metrics struct {
    Requests atomic.Uint64
    Errors   atomic.Uint64
}
```

两个 goroutine 写不同字段，逻辑上无共享状态，但如果处于同一 cache line：

```text
Core0 writes Requests
→ ownership line

Core1 writes Errors
→ ownership same line
```

cache line 在核心之间 ping-pong。

这就是 false sharing。

### 直接解法

- padding；
- 调整 struct layout；
- 分离 shard。

例如：

```go
type Shard struct {
    Counter atomic.Uint64
    _       cpu.CacheLinePad
}
```

### 不适合机械 Padding

padding 会增加：

- object size；
- cache footprint；
- memory footprint。

只适用于：

```text
few objects
+
hot shared write-heavy state
```

---

## 12. True Sharing

如果所有核心写同一个：

```go
counter.Add(1)
```

即使 counter 独占 cache line，也仍然存在 true sharing。

解法不是 padding，而是：

- sharding；
- batching；
- local accumulation；
- single writer。

### Rule

```text
different variables same line
→ false sharing
→ layout

same variable
→ true sharing
→ reduce sharing
```

---

## 13. Memory Bandwidth Bound

例如：

```go
for i := range src {
    dst[i] = src[i]
}
```

大型 streaming workload 可能最终受 DRAM bandwidth 限制。

因此：

```text
workers ↑
```

到一定程度后：

```text
throughput stops scaling
```

甚至下降。

### Rule

CPU 未满不自动意味着增加 goroutine 会更快。

先区分：

- compute bound；
- latency bound；
- synchronization bound；
- bandwidth bound。

---

## 14. Bulk Primitive

如果语义是：

```text
copy bytes
```

优先：

```go
copy(dst, src)
```

而不是自写循环。

runtime/stdlib 可能具有：

- architecture-specific implementation；
- vectorized path；
- optimized memmove；
- special large-copy strategy。

### Rule

不要轻易用 Go loop 重写已经高度优化的 standard primitive。

---

## 15. Branch Prediction

```go
if x >= threshold {
    ...
}
```

branch 成本高度依赖 data distribution。

可预测：

```text
false false false false true true true
```

通常便宜。

近随机：

```text
T F T T F F T F
```

misprediction 会破坏 pipeline。

### Benchmark 陷阱

固定 synthetic input 可能训练出远比生产理想的 branch pattern。

所以 benchmark 应模拟：

- HTTP method mix；
- hit/miss ratio；
- data classification distribution；
- parser shape distribution。

---

## 16. Branchless 不是默认优化

把：

```go
if cond {
    x++
}
```

改成 bit trick / mask 不自动更快。

如果 branch 99.9% predictable，额外 arithmetic 可能反而更贵。

只有：

```text
branch-miss 已被证明是瓶颈
+
benchmark / PMU 证明 branchless 更好
```

才保留。

---

## 17. Instruction Cache

hot function 包含大量 rare slow-path：

- logging；
- formatting；
- debug；
- fallback。

可能扩大 instruction working set。

可考虑：

```go
func hot() {
    if rare {
        slowPath()
    }
}
```

但不要为了“看起来 hot/cold”盲目拆函数。

PGO / benchmark 更重要。

---

## 18. Huge Pages / Manual Prefetch / Streaming Stores

这些属于高级、平台相关优化。

使用前至少要求：

- PMU / perf 证据；
- 明确 memory / TLB / bandwidth 问题；
- workload benchmark；
- deployment 环境稳定。

普通 Go 服务不应把它们当默认建议。

---

## 19. 诊断规则

怀疑 memory-layout 问题时：

1. profile 找 hot path；
2. benchmark 重现；
3. 对比 representation；
4. 测 scaling；
5. 必要时 perf / PMU；
6. 观察：
   - cycles；
   - instructions；
   - branch miss；
   - cache / LLC；
   - bandwidth；
   - TLB；
   - coherence。

---

## 20. Skill Rules

1. 连续、紧凑的数据布局优先于随机 pointer chasing。
2. Pointer chasing 要同时考虑 cache、prefetch、TLB 和 GC。
3. Struct size 影响 cache/TLB coverage。
4. `[]T` 与 `[]*T` 不能只从 allocation 判断。
5. AoS/SoA 根据访问模式选择。
6. False sharing 与 true sharing 必须区分。
7. Padding 只适合少量 hot write-heavy shared state。
8. Sharding 解决 true sharing，padding 解决 false sharing。
9. per-worker 并不自动避免 false sharing。
10. Memory bandwidth saturation 后增加 goroutine 不会继续线性扩展。
11. Branch performance 取决于数据分布。
12. 不默认 branchless 更快。
13. 普通代码优先让 hardware prefetch 工作，而不是手工 prefetch。
14. 只有 Go-level evidence 不够时才进入 PMU / NUMA / huge page 层。
