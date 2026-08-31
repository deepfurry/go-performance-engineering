# 03. Concurrency / Synchronization

## 1. 总纲

性能问题不应该简化成：

```text
atomic > mutex > channel
```

真正的成本模型是：

```text
多少核心
 ×
多高写频率
 ×
多少 shared cache lines
 ×
critical section 时长
 ×
失败后的等待策略
```

---

## 2. Atomic RMW

```go
counter.Add(1)
```

不是单纯“一条 CPU 指令”。

它意味着 Read-Modify-Write，需要：

- 原子语义；
- cache-line ownership；
- coherence traffic；
- ordering；
- contention handling。

### 区分

```text
atomic.Load
atomic.Store
atomic.Add / Swap
CAS
```

它们成本不同。

read-mostly atomic snapshot 和 high-contention atomic counter 不属于同一种 workload。

---

## 3. Immutable Snapshot

read-heavy 配置：

```go
type Store struct {
    config atomic.Pointer[Config]
}
```

read：

```go
cfg := s.config.Load()
```

update：

```go
next := buildConfig()
s.config.Store(next)
```

优势：

- reader 不需要 RMW；
- 不需要 reader count；
- publish 后 immutable。

适用：

```text
very high reads
very rare writes
immutable snapshot
```

---

## 4. Atomic 适合什么

适合：

- flag；
- counter；
- pointer；
- compact bit state。

不适合自动用于：

```text
多个字段必须同时一致
```

例如：

```go
balance.Store(...)
version.Store(...)
```

并不能形成 atomic pair。

### Rule

> Atomic 适合 atomic state；Mutex 适合 compound invariant。

---

## 5. Mutex Fast Path

现代 Go `sync.Mutex` 不是“Lock 就进入 kernel”。

基本模型：

```text
CAS fast path
  ↓ fail
short adaptive spin
  ↓
runtime semaphore
  ↓
park goroutine
```

uncontended mutex 非常便宜。

因此看到：

```go
mu.Lock()
```

不能直接建议 lock-free。

---

## 6. Spin 与 Park

短 critical section：

```text
马上释放
```

短暂 spin 可能比 park/wake 便宜。

长 critical section：

```text
milliseconds
```

持续 spin 是浪费。

Go runtime 会根据运行状态保守决定是否 active spin，而不是无限自旋。

### Anti-pattern

```go
for !mu.TryLock() {
}
```

不要用来替代 runtime 已有的 adaptive strategy。

---

## 7. Normal / Starvation Mode

Go Mutex 当前实现包含：

- normal mode；
- starvation mode。

Normal：

- 偏 throughput；
- 新 arriving goroutine 有优势。

等待过久时进入 starvation：

- 更强调 handoff；
- 限制尾部 waiter starvation。

这说明 Mutex 本身就是 throughput / fairness / latency 的 adaptive algorithm。

---

## 8. Critical Section

优先级极高的优化：

Before：

```go
mu.Lock()
data := expensiveTransform(input)
cache[key] = data
mu.Unlock()
```

After：

```go
data := expensiveTransform(input)

mu.Lock()
cache[key] = data
mu.Unlock()
```

很多时候：

```text
缩短 hold time
```

比：

```text
Mutex → Atomic
```

重要得多。

粗略：

```text
contention pressure
≈ arrival rate × hold time
```

---

## 9. RWMutex

错误条件反射：

```text
读多写少
→ RWMutex
```

`RLock/RUnlock` 自身仍然需要管理共享 reader state。

极短 read section + 高 core count 可能使 reader counter 成为 hotspot。

评估至少考虑：

- read critical-section duration；
- write frequency；
- number of concurrent readers；
- core count。

### Recommendation

Benchmark `Mutex` vs `RWMutex`，不要凭 read/write ratio 决定。

---

## 10. Batching

Before：

```go
for _, event := range events {
    mu.Lock()
    stats[event.Type]++
    mu.Unlock()
}
```

After：

```go
local := map[Type]int{}
for _, event := range events {
    local[event.Type]++
}

mu.Lock()
for k, n := range local {
    stats[k] += n
}
mu.Unlock()
```

本质：

```text
synchronization frequency ↓
```

这是最强的通用并发优化之一。

---

## 11. Sharding

```text
one map + one lock
```

变：

```text
hash(key)%N
   ↓
Shard0
Shard1
...
```

每个 shard：

```go
type Shard struct {
    mu sync.Mutex
    m  map[string]Value
}
```

降低共享范围。

注意结合 CPU 章节检查：

```text
adjacent shard metadata false sharing
```

---

## 12. Single Writer

```text
Worker0 ─┐
Worker1 ─┤
Worker2 ─┼→ Aggregator
Worker3 ─┘
```

Aggregator 独占 mutable state：

```go
stats[key]++
```

不需要 atomic/mutex。

这是 ownership redesign：

```text
shared mutable state
→ message passing + ownership
```

---

## 13. Channel

Channel 提供：

- communication；
- queue；
- blocking；
- backpressure；
- ownership handoff。

因此不能只比较：

```text
channel send ns/op
vs
mutex Lock ns/op
```

它们语义不同。

### Rule

- mutual exclusion → Mutex；
- ownership / task transfer / queueing → Channel；
- single atomic state → Atomic。

---

## 14. CAS Loop

```go
for {
    old := state.Load()
    next := calculate(old)
    if state.CompareAndSwap(old, next) {
        return
    }
}
```

高竞争：

```text
many goroutines calculate
one succeeds
others discard and retry
```

形成 retry amplification。

### Rule

Lock-free ≠ contention-free。

---

## 15. Progress Guarantees

粗略：

- blocking；
- obstruction-free；
- lock-free；
- wait-free。

`lock-free` 表示系统整体持续有人取得进展，不保证单个 goroutine 在有限步骤内完成。

因此：

```text
lock-free
≠ wait-free
≠ starvation-free
≠ faster
```

---

## 16. ABA

状态：

```text
A → B → C
```

G1 读 head=A。

G2：

```text
pop A
pop B
push A
```

head 又等于 A。

G1 的 CAS 只看到：

```text
A == A
```

无法知道状态经历过变化。

### 关键

ABA 不要求对象被 free/reallocate。

同一个 node 被 remove/reinsert 就可能发生逻辑 ABA。

---

## 17. GC 与 Lock-Free

Go GC 能显著减少 C/C++ lock-free 世界的 use-after-free / memory reclamation 复杂度：

只要 goroutine 仍持有正常 GC-visible pointer，对象不会因为从数据结构 unlink 就立即消失。

但 GC 不解决：

- logical ABA；
- linearizability；
- retry；
- starvation；
- cache contention；
- invariant。

### Rule

> GC helps reclamation; it does not prove the algorithm correct.

---

## 18. Tagged Pointer / Version

典型 ABA 防御：

```text
(pointer, version)
```

A 第一次：

```text
(A, 17)
```

重新插入：

```text
(A, 18)
```

旧 CAS 失败。

Go runtime 的 lock-free stack 是理解这种机制的好案例，但其 tagged-pointer / non-GC-visible node 设计属于 runtime internal，不能照抄到普通 Go heap object。

---

## 19. Backoff

Naive：

```text
CAS fail
→ immediately CAS again
```

高竞争会导致：

- retry storm；
- shared line pressure；
- loser 影响 winner。

Backoff：

```text
fail
→ short delay / yield
→ retry
```

目标是降低 shared memory pressure。

### 不能机械化

参数依赖：

- architecture；
- core count；
- contention；
- operation length。

不要自动加入固定 `time.Sleep(...)`。

---

## 20. Linearization Point

Lock-free operation 必须能够说明：

> 在哪一个瞬间，这个操作在抽象数据结构意义上生效？

例如 stack push：

```text
prepare next
↓
successful CAS ← linearization point
↓
return
```

如果无法清晰指出 linearization point，自设计 lock-free structure 是高风险信号。

---

## 21. Race Detector 的边界

race-free 不意味着：

- ABA-free；
- linearizable；
- invariant correct；
- starvation-free。

Atomic-only bug 完全可能通过 `go test -race`。

---

## 22. Pool 与 Lock-Free

`sync.Pool` / freelist 加速 node reuse 会改变：

```text
identity / reuse assumptions
```

可能增加 ABA 风险。

不能只看到：

```text
B/op ↓
```

就判定 lock-free + pool 更好。

---

## 23. Synchronization Benchmark

至少测：

```text
1
2
4
8
16
32
```

cores / workers 的 scaling curve。

关注：

- throughput；
- P50/P99；
- CPU；
- retries；
- allocations；
- GC；
- lock wait。

单线程 `atomic.Add` benchmark 对生产 64-core hotspot 没有代表性。

---

## 24. Diagnostics

### Mutex Profile

回答：

> 谁持有 contended lock，让别人等待？

### Block Profile

回答：

> goroutine 在哪里发生 blocking？

### Trace

回答：

> 哪个时间点 run / runnable / blocked / GC / syscall？

### perf / PMU

必要时分析：

- cache coherence；
- cache miss；
- cycles/op。

---

## 25. Decision Tree

```text
shared mutable state
        │
        ▼
能否 immutable snapshot？
   │             │
  yes           no
   │             │
atomic.Pointer   单一 word？
                 │
          ┌──────┴──────┐
         yes            no
         │              │
       atomic         mutex
         │              │
         └──── high contention?
                       │
                       ▼
              redesign sharing
                       │
          sharding / batching /
          local state / single writer
```

---

## 26. Skill Rules

1. Atomic ≠ free。
2. Atomic Load 和 RMW 不是同类成本。
3. Lock-free 是 progress property，不是 performance property。
4. Uncontended Mutex 很便宜。
5. Atomic 适合单一状态，Mutex 适合 invariant。
6. 优先缩短 critical section。
7. RWMutex 不是 Mutex Pro。
8. TryLock loop 不是推荐的 spin strategy。
9. 高 contention 先减少 sharing。
10. Sharding 与 padding 解决不同问题。
11. Channel 根据 communication semantics 选择。
12. GC 减少 reclamation complexity，但不消除 ABA。
13. Race detector 不能证明 lock-free correctness。
14. Lock-free benchmark 必须测 retries、CPU 和 scaling。
15. 严重 contention 首先是 ownership / architecture 问题。
