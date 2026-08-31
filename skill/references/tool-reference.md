# 09. Tool Reference

## 1. Benchmark

### Run

```bash
go test -run='^$' -bench=. -benchmem ./...
```

### Repeat

```bash
go test \
  -run='^$' \
  -bench='BenchmarkFoo$' \
  -benchmem \
  -count=15 \
  ./...
```

### CPU scaling

```bash
go test \
  -run='^$' \
  -bench='BenchmarkFoo$' \
  -benchmem \
  -cpu=1,2,4,8,16 \
  ./...
```

---

## 2. benchstat

```bash
benchstat old.txt new.txt
```

回答：

- A/B difference；
- variance；
- effect size；
- statistical confidence。

不要把 statistical significance 自动等同 engineering significance。

---

## 3. CPU Profile

```bash
go test \
  -run='^$' \
  -bench=BenchmarkFoo \
  -cpuprofile=cpu.pprof
```

```bash
go tool pprof cpu.pprof
```

常用：

```text
top
top -cum
list <func>
web
```

回答：

> active CPU time 花在哪里？

---

## 4. Heap / Allocs

生产通常通过 pprof endpoint。

概念：

```text
heap / inuse
→ currently retained/live

allocs / alloc_space
→ cumulative allocation churn
```

---

## 5. Mutex / Block

启用：

```go
runtime.SetMutexProfileFraction(...)
runtime.SetBlockProfileRate(...)
```

回答：

```text
mutex
→ who causes contention

block
→ where goroutines block
```

---

## 6. Execution Trace

```bash
go test -trace=trace.out ./...
```

```bash
go tool trace trace.out
```

适合：

- scheduler；
- latency timeline；
- GC assist；
- blocking；
- syscalls。

---

## 7. FlightRecorder

适合 rare production event：

```text
keep recent trace window
↓
detect anomaly
↓
dump
```

具体 API 以当前 `runtime/trace` package 文档为准。

---

## 8. runtime/metrics

用于长期：

- GC CPU；
- assist；
- heap live/goal；
- memory classes；
- released memory。

推荐通过：

```go
metrics.All()
```

获取当前 toolchain 实际支持的 metric descriptions，不要把所有 metric names 永久硬编码到工具逻辑。

---

## 9. GC Trace

```bash
GODEBUG=gctrace=1 ./app
```

快速观察：

- cycle frequency；
- heap before/after；
- goal；
- GC CPU。

只适合作为第一眼。

---

## 10. Escape / Inline Diagnostics

```bash
go build -gcflags='-m=2' ./...
```

更详细：

```bash
go build -gcflags='-m=3' ./...
```

观察：

```text
moved to heap
does not escape
can inline
cannot inline
```

---

## 11. BCE

```bash
go build \
  -gcflags='-d=ssa/check_bce/debug=1' \
  ./...
```

注意：

```text
Found IsInBounds
```

表示 residual bounds check。

---

## 12. SSA

```bash
GOSSAFUNC=Foo go build ./...
```

生成 `ssa.html`。

用于：

- prove；
- rewrite；
- control-flow；
- optimization pass。

---

## 13. Assembly

Compiler output：

```bash
go build -gcflags='-S' ./...
```

Binary：

```bash
go tool objdump -s 'pkg\.Foo' ./app
```

检查：

- CALL；
- indirect call；
- bounds panic path；
- atomic instruction；
- hot loop。

---

## 14. Disable Inline / Optimization（实验）

```bash
go test -gcflags='all=-l' -bench=. ./...
```

`-N` 可用于关闭优化做研究。

不要用于 production build。

---

## 15. Race

```bash
go test -race ./...
```

用于 concurrency correctness。

不用于 production performance baseline。

---

## 16. checkptr

unsafe-heavy code：

```bash
go test \
  -gcflags=all=-d=checkptr=2 \
  ./...
```

用于发现一部分 invalid unsafe pointer use。

---

## 17. Vet

```bash
go vet ./...
```

对 unsafe / concurrency / API pattern 有基础静态检查价值。

Vet clean 不等于 unsafe correctness proof。

---

## 18. perf stat

Linux：

```bash
perf stat ./app
```

或选择 events：

```bash
perf stat \
  -e cycles,instructions,branches,branch-misses \
  ./app
```

用于 Go-level evidence 不足时分析 CPU microarchitecture。

---

## 19. IPC

```text
IPC = instructions / cycles
```

只能作为粗线索。

不能独立诊断：

- cache；
- false sharing；
- branch；
- memory stalls。

---

## 20. Version Capture

每次严肃 benchmark 建议保存：

```bash
go version
go env GOOS GOARCH GOEXPERIMENT GOMAXPROCS
```

以及：

- CPU model；
- kernel；
- git commit。

---

## 21. Tool Selection Matrix

| Symptom | First Tools | Deep Tools |
|---|---|---|
| CPU high | CPU pprof | compiler / perf |
| allocation churn | allocs | `-m`, benchmark |
| retained memory | heap | runtime metrics / RSS |
| lock contention | mutex/block | trace / perf |
| tail latency | trace / FlightRecorder | metrics + targeted profile |
| compiler missed optimization | `-m`, BCE | SSA / asm |
| high-core collapse | scaling benchmark | perf / PMU |
| unsafe bug | vet/race/checkptr | architecture-specific tests |
