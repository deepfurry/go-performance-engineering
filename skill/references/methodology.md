# 01. Go Performance Engineering Methodology

## 1. 核心目标

性能工程不是“让程序更快”，而是在给定约束下优化系统成本。

常见目标：

- throughput：req/s、ops/s、MB/s；
- latency：P50 / P95 / P99 / P99.9；
- CPU efficiency：CPU-seconds/request、cycles/op；
- memory：RSS、live heap、B/op；
- cost efficiency：每 10 万 QPS 所需 CPU / memory；
- concurrency scalability：1→N cores / workers 的扩展效率。

任何优化开始前都应明确：

```text
Primary Objective
+
Guardrails
```

例如：

```text
目标：
P99 < 20 ms

Guardrails：
CPU 不增加超过 5%
RSS 不增加超过 10%
吞吐不下降
```

---

## 2. 总体工作流

```text
Performance Goal
      ↓
Baseline
      ↓
Symptom Classification
      ↓
Targeted Measurement
      ↓
Hotspot / Bottleneck
      ↓
Cost Model
      ↓
Hypothesis
      ↓
Candidate Change
      ↓
Representative Benchmark
      ↓
Repeated A/B + benchstat
      ↓
Component / End-to-End Validation
      ↓
Production / Canary
      ↓
Regression Guard
```

### 第一原则

**不要跳过 Measurement 直接进入 Candidate Change。**

---

## 3. 症状分类

### CPU 高

优先：

- CPU pprof；
- runtime metrics；
- 进一步检查 compiler / GC / atomic / cache。

### CPU 不高但吞吐低

优先：

- block profile；
- mutex profile；
- execution trace；
- IO / syscall；
- serialization / single-thread bottleneck。

### Memory 高

区分：

- Go live heap；
- allocation churn；
- RSS；
- mmap / cgo / external memory；
- backing-array retention；
- fragmentation / scavenger。

### Allocation 高

优先：

- allocs profile；
- `-gcflags=-m=2/-m=3`；
- preallocation；
- append-style API；
- pool / reuse。

### Tail Latency 抖动

优先：

- execution trace；
- FlightRecorder；
- GC assist；
- scheduler latency；
- lock/block；
- syscall / network wait。

### 高并发后性能崩

优先：

- scaling benchmark；
- atomic contention；
- mutex / RWMutex；
- false sharing；
- cache coherence；
- memory bandwidth；
- NUMA。

### 单个 hot loop 慢

优先：

- microbenchmark；
- BCE / escape / inline diagnostics；
- SSA；
- objdump；
- perf / PMU。

---

## 4. 从“热点”到“成本模型”

一个 hotspot 不是结论。

例如 CPU profile：

```text
runtime.scanobject 20%
```

不能直接结论：

> Go GC 太慢。

正确问题：

```text
为什么需要扫描这么多？
```

可能来自：

- pointer-rich object graph；
- live heap 变大；
- allocation churn；
- roots 变大；
- memory limit 太紧。

同样：

```text
sync/atomic.(*Uint64).Add 15%
```

不能直接：

> atomic 太慢。

应继续问：

- 是否所有核心写同一个 cache line？
- 是否 false sharing？
- 是否可以 sharding？
- 是否可以 local accumulation？
- 是否根本不应该共享写状态？

---

## 5. 建立假设

好的性能假设必须可证伪。

差：

> `sync.Pool` 应该能快。

好：

> allocs profile 显示 64 KiB scratch buffer 每秒产生约 4 GiB allocation churn；复用该 buffer 应显著降低 B/op 和 GC cycle frequency，同时不增加 retained RSS。

差：

> interface 慢。

好：

> CPU profile 显示 hot parser 的 interface call 占显著 CPU；compiler diagnostics 显示无法 devirtualize / inline。将 hot path receiver concrete 化后应减少 indirect call 并打开 inlining。

---

## 6. Optimization Budget

高级优化前回答四个问题。

### Hotness

这个路径占系统总成本多少？

例如：

```text
hot path = total CPU 0.5%
```

即使无限加速，CPU 最大收益也小于 0.5%。

### Ceiling

理论收益上限是多少？

### Risk

引入了多少：

- unsafe；
- concurrency complexity；
- maintenance burden；
- runtime implementation dependency。

### Validation Cost

能否可靠重现和测量？

概念上：

```text
Optimization Value
≈ Expected System Gain
  --------------------
  Complexity + Risk + Maintenance
```

---

## 7. 证据升级路径

### 默认路径

```text
Source
 ↓
Profile / Metrics
 ↓
Benchmark
 ↓
Compiler / Runtime Diagnostics
 ↓
SSA / Assembly / PMU
```

不要倒过来。

### 什么时候看 Assembly

适合：

- bounds check 是否真的消失；
- interface 是否 devirtualize；
- hot atomic / SIMD / codec loop；
- compiler transformation 仍无法解释 benchmark 差异。

### 什么时候上 perf / PMU

只有 Go 层证据已经不足以解释：

- cache；
- TLB；
- branch；
- IPC；
- memory bandwidth；
- coherence。

---

## 8. 风险与证据门槛

| 优化 | 最低建议证据 |
|---|---|
| preallocation | L1–L2 |
| shrink critical section | L1–L2 |
| sharding | L2–L3 |
| AoS→SoA | L2–L3 |
| PGO | L3/L4 workload profile |
| unsafe zero-copy | L2 + L3，最好 L4 |
| lock-free rewrite | L2/L3 + correctness proof |
| runtime private hack | 默认不建议 |

---

## 9. Stop Conditions

出现任一情况，应停止当前优化方向。

### 没有热点证据

仅仅看到：

```text
allocation
mutex
interface
copy
bounds check
```

不能证明它们是问题。

### 收益上限太低

例如 hot path 只占总 CPU 0.2%。

### Benchmark 不代表生产

无法模拟：

- input distribution；
- contention topology；
- cache hit ratio；
- object lifetime。

### 局部收益无法传导

```text
microbenchmark +40%
service benchmark +0.1%
```

若复杂度显著增加，应回滚。

### 破坏 guardrail

```text
throughput +5%
P99 +80%
```

不能称为成功。

### 依赖未文档化 runtime 行为

除非这是明确承担维护成本的底层库。

---

## 10. 性能调查示例：GC CPU 回归

症状：

```text
P99: 30 ms → 90 ms
CPU: 60% → 85%
RSS: 基本不变
```

CPU profile：

```text
gcBgMarkWorker
scanobject
```

runtime metrics：

```text
live heap ≈ unchanged
allocation bytes/sec ↑↑
mark assist CPU ↑
```

allocs profile：

```text
new []map[string]any transformation
```

compiler：

```text
temporary values escape
```

假设：

> allocation churn 导致 GC frequency 和 assist 增加。

修改：

- typed representation；
- preallocated slice；
- append-style encoder。

验证：

```text
B/op -70%
allocs/op -80%
microbench ns/op -25%
service P99 -30%
GC CPU -45%
```

这是完整证据链。

---

## 11. 性能调查示例：多核扩展崩溃

症状：

```text
8 cores → 100k ops/s
32 cores → 110k ops/s
CPU = 100%
```

profile：

```text
atomic.Add
```

scaling benchmark：

```text
1 core   excellent
8 core   good
16 core  degradation
32 core  collapse
```

成本模型：

```text
all workers
   ↓
same atomic cache line
   ↓
coherence contention
```

候选：

- local accumulation；
- sharded counters；
- single writer。

验证：

- benchmark；
- CPU efficiency；
- perf / PMU 必要时确认 cache/coherence。

---

## 12. 最终完成条件

一项优化完成必须同时满足：

1. 问题有基线；
2. bottleneck 有证据；
3. 修改对应明确成本模型；
4. A/B 可重复；
5. effect size 有工程意义；
6. system metric 有改善；
7. 没破坏 guardrails；
8. 重要路径留下 regression guard。

**“代码看起来更聪明”不属于完成条件。**
