# 10. Version-Sensitive Notes

> 性能技巧最容易失效的原因之一不是代码变化，而是 Go compiler/runtime 已经变了。

本文以 **Go 1.27（2026-08）** 为当前基线。

---

## 1. Go 1.24

重要性能工程相关能力：

- `testing.B.Loop` 引入，现代 benchmark 推荐逐步迁移；
- weak pointer 能力进入标准生态；
- `runtime.AddCleanup` 让资源 fallback cleanup 比传统 finalizer 更可控；
- 低层 GC / lifetime 工具进一步丰富。

### 影响

旧 benchmark：

```go
for i := 0; i < b.N; i++ { ... }
```

仍可工作，但新文档/Skill 应优先使用 `B.Loop`。

---

## 2. Go 1.25

重要：

- `runtime/trace.FlightRecorder` 用于保留最近一段 trace window；
- 更适合 rare production latency/stall；
- Green Tea GC 曾作为实验阶段继续演进。

### 影响

对于：

```text
“问题发生后才开始 trace”
```

可以升级为：

```text
always-on recent trace window
→ anomaly dump
```

---

## 3. Go 1.26

### Green Tea GC 默认启用

重点改善：

- small-object marking/scanning locality；
- CPU scalability；
- GC-heavy workload overhead。

Go 官方预期 real-world GC-heavy program 的 GC overhead 有显著下降空间，具体 workload 差异很大。

### Compiler

更多 slice backing store 可以在 stack 上分配。

### cgo

baseline cgo call overhead 获得明显降低。

### Experimental SIMD

出现 architecture-specific experimental SIMD package。

### 影响

旧结论需要重新 benchmark：

- “small pointer objects 一定非常差”；
- “某个 object pooling 在 GC-heavy workload 必然收益巨大”；
- cgo boundary 的旧数据；
- slice escape / backing allocation。

---

## 4. Go 1.27

### Faster Small Allocation

compiler/runtime 对一部分非常小的 allocation 使用更专门的 fast path，降低部分 small allocation 成本。

### Generic Methods

语言新增 generic methods。

这不意味着 generics 性能模型突然等价于 C++ template；仍需检查 compiler-generated code。

### Goroutine Leak Profile

goroutine leak profile 进入正式可用阶段。

### 影响

任何“每个小 allocation 非常贵”的历史规则都必须弱化。

真正关注：

```text
allocation rate
object count
GC
lifetime
```

而不是单次 malloc 的刻板印象。

---

## 5. Green Tea 对旧 GC 技巧的影响

仍然成立的成本模型：

- pointer density matters；
- scannable heap matters；
- allocation rate matters；
- GOGC / GOMEMLIMIT trade-off；
- retention matters。

但具体收益可能变化：

- pointer-field ordering；
- small-object pooling；
- flatten small pointer graph；
- object-count reduction。

因此：

> implementation-specific GC optimization 必须在当前 toolchain benchmark。

---

## 6. Compiler Implementation Detail

以下尤其需要跨版本重新验证：

- inline decision；
- escape result；
- BCE；
- devirtualization；
- generic lowering；
- size-specialized allocation；
- architecture-specific instruction selection。

不要在文档或 Skill 中写：

```text
某个函数永远会 inline
某个代码形态永远不会 escape
某个 size 永远对应某固定 fast path
```

除非这属于语言/API contract，而不是 compiler implementation。

---

## 7. Runtime Private APIs

以下不能依赖 Go 1 compatibility：

- `runtime.procPin` private symbol；
- internal tagged pointer；
- private lfstack；
- internal cache-line details；
- linkname targets。

Skill 应将它们归为：

```text
Runtime Archaeology
```

而不是稳定 performance API。

---

## 8. Upgrade Checklist

升级 Go version 后，对性能关键项目至少检查：

1. run existing benchmark suite；
2. benchstat old toolchain vs new；
3. production-like service benchmark；
4. GC CPU / assist / heap metrics；
5. CPU profile top hotspots 是否变化；
6. escape / inline / BCE 是否变化；
7. unsafe/cgo tests；
8. PGO profile 是否仍 representative；
9. runtime private dependencies（若有）；
10. perf regression only when system metric shows anomaly。

---

## 9. Version Rule

最终 Skill 应强制：

> 对 implementation-specific optimization，如果 benchmark 数据不是来自当前或目标 Go toolchain，则只能作为假设，不能作为事实结论。
