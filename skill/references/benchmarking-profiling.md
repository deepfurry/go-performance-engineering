# 07. Benchmarking / Profiling / Performance Investigation

## 1. Profile 与 Benchmark

### Profile

回答：

> 现在成本花在哪里？

### Benchmark

回答：

> A 与 B 谁更好？

正确顺序通常：

```text
production / representative profile
↓
hotspot
↓
reproducible benchmark
↓
candidate optimization
↓
A/B
```

---

## 2. Benchmark 目标与 Guardrails

不要只看：

```text
ns/op
```

根据系统目标同时考虑：

- throughput；
- P50/P99；
- CPU；
- B/op；
- allocs/op；
- RSS；
- GC；
- lock wait；
- scalability。

---

## 3. `testing.B.Loop`

现代 benchmark 推荐：

```go
func BenchmarkFoo(b *testing.B) {
    input := prepare()

    for b.Loop() {
        Foo(input)
    }
}
```

优势：

- setup/cleanup 更自然；
- 降低 benchmark body 被 compiler DCE 的风险；
- 避免手工 `b.N` 细节。

---

## 4. B.Loop 不能修复错误 Workload

错误：

```go
func BenchmarkSort(b *testing.B) {
    xs := randomData()

    for b.Loop() {
        slices.Sort(xs)
    }
}
```

第一次后数据已经 sorted。

因此必须保证：

> 每一个 benchmark iteration 表示同一种真实 operation。

必要时 `StopTimer/StartTimer` 重置状态。

---

## 5. Workload Validity

必须检查：

- input size distribution；
- hit/miss；
- branch distribution；
- read/write ratio；
- concurrency；
- cache warm/cold；
- mutation；
- connection reuse；
- object lifetime；
- failure rate。

---

## 6. Distribution Benchmark

生产：

```text
p50 payload  = 512B
p90          = 8KiB
p99          = 256KiB
```

至少：

```go
b.Run("512B", ...)
b.Run("8KiB", ...)
b.Run("256KiB", ...)
```

不要只选择“一个典型大小”。

---

## 7. Concurrency / Scaling Benchmark

同步 primitive 必须测试：

```text
1
2
4
8
16
32
```

workers / CPUs。

看：

- throughput curve；
- ns/op；
- CPU；
- P99；
- contention。

单线程 atomic benchmark 几乎不能说明高并发热点。

---

## 8. Repeat + benchstat

Before：

```bash
go test \
  -run='^$' \
  -bench='BenchmarkFoo$' \
  -benchmem \
  -count=15 \
  ./... > old.txt
```

After：

```bash
... > new.txt
```

Compare：

```bash
benchstat old.txt new.txt
```

不要用一次 `100ns → 95ns` 宣布成功。

---

## 9. Statistical vs Engineering Significance

统计显著：

```text
difference stable
```

不等于工程价值大。

```text
50.00 ns → 49.98 ns
```

可能没有任何系统意义。

反之大 effect 但噪声高：

> 应先改善 benchmark 环境。

---

## 10. Environment Control

记录：

- `go version`；
- GOOS / GOARCH；
- CPU model；
- GOMAXPROCS；
- build flags；
- GOEXPERIMENT；
- commit；
- kernel / environment。

版本尤其重要，因为：

- compiler；
- allocator；
- GC；

都会变化。

---

## 11. Thermal / Frequency Noise

现代 CPU boost / power / thermal 会使 microbenchmark 波动。

尽量：

- 重复；
- 减少 background load；
- old/new 近时间交错；
- 避免跨天比较极小变化。

---

## 12. `-benchmem`

同时看：

```text
ns/op
B/op
allocs/op
```

但不要机械：

```text
0 alloc = best
```

系统性能取决于总成本。

---

## 13. CPU Profile

测试：

```bash
go test \
  -bench=BenchmarkFoo \
  -cpuprofile=cpu.pprof
```

生产可通过 `net/http/pprof`。

CPU profile回答：

> active CPU time 主要在哪里？

不直接回答：

- lock wait；
- network wait；
- channel wait；
- sleep。

---

## 14. pprof Flat vs Cumulative

```text
flat
```

表示函数自身采样。

```text
cum
```

表示包括 downstream call tree。

不要看到 Handler `cum=50%` 就认为 Handler 自身几行代码花了 50%。

---

## 15. Runtime Hotspot 需要反查 Application Cause

例如：

```text
runtime.mallocgc
runtime.scanobject
```

不要结论：

> runtime 慢。

继续问：

- 谁分配？
- 什么对象？
- live 多少？
- pointer density？
- churn？

---

## 16. Heap vs Allocs

### Heap / inuse

回答：

> 谁现在还活着、占着内存？

适合：

- retention；
- cache；
- leak；
- backing array。

### Allocs / alloc_space

回答：

> 谁历史上不断制造 allocation？

适合：

- churn；
- GC frequency；
- temporary objects。

不要读反。

---

## 17. Memory Profile Sampling

Memory profile 不是每次 allocation 的精确日志。

默认抽样。

因此：

```text
profile 没出现
```

不能证明：

```text
绝对没有 allocation
```

---

## 18. Mutex Profile

回答：

> 哪些 critical section 造成别人等待？

适合：

- lock contention；
- long hold；
- hot shared state。

---

## 19. Block Profile

回答：

> goroutine 在哪里被阻塞？

涵盖：

- mutex；
- channel；
- cond；
- waitgroup；
- select。

CPU 低但吞吐低时尤其重要。

---

## 20. Execution Trace

pprof：

```text
aggregate
```

trace：

```text
timeline
```

适合：

- P99；
- scheduler；
- runnable delay；
- GC assist；
- blocking；
- syscall；
- phase interaction。

---

## 21. Flight Recorder

适合：

```text
偶发 P99.99 spike
罕见 stall
每日一次异常
```

维护最近一段 trace window，异常时 dump。

比“异常发生后才开始 trace”更适合 rare event。

---

## 22. Goroutine Leak Profile

现代 Go 可使用 goroutine leak profile 帮助识别一类永久无法解除阻塞的 goroutine。

它不能发现所有 leak。

因此：

```text
no detected leak
```

不等于：

```text
no leak possible
```

---

## 23. runtime/metrics

用于长期 time-series：

- GC CPU；
- GC assist；
- heap live；
- heap goal；
- memory classes；
- cycles；
- released heap。

推荐两层：

```text
continuous metrics
↓
detect when

pprof / trace
↓
explain why
```

---

## 24. `GODEBUG=gctrace=1`

适合快速第一眼：

- GC frequency；
- heap before/after；
- goal；
- GC CPU。

不适合替代正式：

- metrics；
- profile；
- trace。

---

## 25. Compiler Diagnostics 在调查中的位置

如果 hot path 涉及：

- allocation；
- BCE；
- interface；
- inlining；

按：

```text
-m
↓
check_bce
↓
SSA
↓
objdump
```

逐层升级。

---

## 26. perf / PMU

只有 Go-level evidence 无法解释时再上。

基础：

```bash
perf stat ./program
```

常见：

- cycles；
- instructions；
- branches；
- branch-misses；
- cache refs/misses。

架构级进一步可看：

- LLC；
- TLB；
- memory bandwidth；
- coherence。

不要用一个 generic `cache-misses` 就断言 false sharing。

---

## 27. IPC

粗略：

```text
IPC = instructions / cycles
```

如果：

```text
instructions similar
cycles much higher
```

可能需要看：

- cache；
- branch；
- dependencies；
- contention。

它是线索，不是结论。

---

## 28. NUMA

大型多 socket 机器：

```text
CPU socket 0 ↔ local memory
CPU socket 1 ↔ local memory
```

remote memory latency 不同。

当：

```text
large-core scaling abnormal
```

NUMA 是应控制/排除的实验变量。

普通 Go 服务无需自动 NUMA tuning。

---

## 29. Tool Intrusion

不要同时打开所有 profiling instrumentation 然后把结果当无扰动 baseline。

某些：

- precise memory profiling；
- trace；
- block sampling；
- race；
- checkptr；

会影响运行行为。

尽量分开采集。

---

## 30. Race / checkptr Benchmark

```bash
go test -race -bench=.
```

可用于 correctness stress。

不能把它的 ns/op 当 production performance。

同理 sanitizer/checkptr。

---

## 31. End-to-End Validation

microbenchmark：

```text
unsafe conversion -80%
```

如果 service：

```text
+0.1%
```

不一定值得。

正确层级：

```text
micro
↓
component
↓
service
↓
production/canary
```

---

## 32. Production Profile

生产 workload 自带：

- 真实 input；
- traffic mix；
- contention；
- cache hit；
- branch；
- lifetime。

非常适合建立 hypothesis。

随后需要 benchmark 做 A/B 因果验证。

---

## 33. PGO Profile 与异常 Diagnostic Profile

不要把专门抓 rare spike 的 CPU profile 机械放入 `default.pgo`。

PGO 需要 representative workload。

诊断 profile 则可以故意偏向异常窗口。

用途不同。

---

## 34. Regression Guard

稳定 hot path 可保留 benchmark：

```text
BenchmarkDecode
BenchmarkRouter
BenchmarkCounter
```

CI 共享机器不适合 1% strict threshold。

可考虑：

- 专用 perf runner；
- trend；
- large regression threshold。

---

## 35. Allocation Contract

某些极热 API 可把：

```text
0 alloc/op
```

作为可测 invariant。

这有时比：

```text
100ns → 103ns
```

在 noisy CI 更稳定。

但只针对真正有必要的路径，不把 0 alloc 宗教化。

---

## 36. Final Investigation Workflow

```text
Goal
↓
Baseline
↓
Classify symptom
↓
pprof / metrics / trace
↓
find bottleneck
↓
cost model
↓
hypothesis
↓
representative benchmark
↓
candidate change
↓
repeated A/B
↓
benchstat
↓
component/service
↓
production/canary
↓
regression guard
```

---

## 37. Skill Rules

1. Profile 找问题，benchmark 比方案。
2. 先定义 objective + guardrails。
3. CPU low + throughput low 时优先查 wait。
4. Heap=inuse；Allocs=churn。
5. profile sampling 不是完整事件日志。
6. tail latency 用 trace / FlightRecorder。
7. 长期 runtime 趋势用 metrics。
8. 新 benchmark 优先 B.Loop。
9. B.Loop 不能修复错误 workload。
10. benchmark 必须模拟生产 distribution。
11. synchronization 必须测 scaling。
12. 反复运行 + benchstat。
13. 同时看统计意义与工程意义。
14. 记录版本、架构和环境。
15. microbenchmark 必须尽量传导到系统指标。
16. PMU 是后层工具，不是第一层。
17. diagnostics 工具尽量分开采集。
18. race/checkptr 不作为性能 baseline。
19. unsafe 优化要求更高证据门槛。
20. 允许结论是“不优化”。
