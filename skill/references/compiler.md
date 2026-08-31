# 04. Compiler / SSA / Escape / BCE / PGO

## 1. 总纲

最好的优化不是让某项工作更快，而是让 compiler 证明它根本不需要存在。

```text
heap allocation → stack
bounds check    → eliminated
interface call  → direct call
function call   → inline
constant branch → dead code
```

---

## 2. Compiler Pipeline

现代 Go compiler 可以粗略理解：

```text
Source
 ↓
Typecheck / IR
 ↓
Devirtualization
Inlining
Escape Analysis
 ↓
SSA
 ↓
Rewrite / Prove / DCE / BCE
 ↓
Architecture Lowering
 ↓
Register Allocation
 ↓
Machine Code
```

这些 pass 不是孤立的。

最重要的关系：

```text
devirtualization
   ↓
inlining
   ↓
escape / BCE / constant propagation
   ↓
DCE
```

---

## 3. SSA

SSA = Static Single Assignment。

概念：

```go
x := 1
if cond {
    x = 2
}
return x
```

转换思维：

```text
x0 = 1
x1 = 2
x2 = φ(x0, x1)
```

这样 compiler 更容易追踪：

- value 来源；
- constant；
- control flow；
- redundant checks；
- dead values。

---

## 4. Escape Analysis

核心问题不是：

```text
有没有 &
有没有 new
```

而是：

```text
value 的 lifetime
是否允许留在 stack？
```

核心安全要求：

- stack object pointer 不能进入比它更长寿的 heap；
- stack object pointer 不能在 stack object 生命周期结束后仍被使用。

### 示例

```go
func f() {
    x := Foo{}
    use(&x)
}
```

如果 `use` 不保存 pointer：

```text
x may stay on stack
```

### 返回 pointer

```go
func newFoo() *Foo {
    x := Foo{}
    return &x
}
```

单独看 callee，`x` 不能只依赖 callee stack。

但如果函数被 inline，caller context 可能让 compiler 重新证明 lifetime。

---

## 5. Inlining 的真正价值

不只是省 CALL。

例：

```go
func enabled(flags uint32) bool {
    return flags&1 != 0
}

func process() {
    const flags = 0
    if enabled(flags) {
        expensive()
    }
}
```

Inline 后：

```text
constant propagation
↓
false condition
↓
DCE
↓
entire expensive path disappears
```

所以：

> Inlining is an enabling optimization.

---

## 6. 不要针对 Inline Budget 编程

compiler 有 heuristics、budget、call-site scoring、PGO hotness。

不要写规则：

```text
函数控制在 N 个 AST node 以下
```

这种做法依赖 implementation detail。

正确方式：

```bash
go build -gcflags='-m=2' ./...
```

看当前 compiler 自己的决定。

---

## 7. Compiler Diagnostics

```bash
go build -gcflags='-m=2' ./...
```

重点观察：

```text
can inline
cannot inline
moved to heap
does not escape
leaking param
```

更详细：

```bash
go build -gcflags='-m=3' ./...
```

### 性能工作流

```text
alloc hotspot
↓
-m
↓
为什么 escape？
↓
针对 lifetime / API shape 修复
```

而不是第一反应 `sync.Pool`。

---

## 8. Pointer Receiver 不等于 Heap

```go
func (x *Foo) Value() int
```

并不意味着 Foo 必须 heap allocate。

stack object 完全可以有 pointer receiver，只要 pointer lifetime 合法。

---

## 9. Interface 也可能导致 Escape

```go
func f() any {
    x := Foo{}
    return x
}
```

即使源码没有显式 `&`，representation / lifetime 仍可能导致 heap allocation。

因此：

> 从语法猜 allocation 不可靠。

---

## 10. Closure

Closure 是否 heap allocate 仍然取决于 lifetime。

返回 closure：

```go
func counter() func() int {
    n := 0
    return func() int {
        n++
        return n
    }
}
```

`n` 必须活过 `counter` 返回。

局部立即调用 closure 则可能完全不同。

---

## 11. Large Non-Escaping Objects

`does not escape` 也不意味着“必定 stack”。

过大 object 仍可能因为 stack constraints 被放到 heap。

所以：

```text
escape diagnostics
```

只是 lifetime 维度的信息，不是最终 placement 的全部事实。

---

## 12. Bounds Check Elimination

Go 必须保证：

```go
s[i]
```

满足：

```text
0 <= i < len(s)
```

若 compiler 已能证明，bounds check 可消失。

### Range

```go
for i := range xs {
    sum += xs[i]
}
```

compiler 很容易知道 `i` 合法。

### Guard

```go
if len(b) < 4 {
    return 0
}

return uint32(b[0]) |
    uint32(b[1])<<8 |
    uint32(b[2])<<16 |
    uint32(b[3])<<24
```

成功路径提供：

```text
len(b) >= 4
```

作为 proof。

---

## 13. `_ = b[N]`

典型 BCE-friendly pattern：

```go
_ = b[7]

x0 := b[0]
x1 := b[2]
x2 := b[7]
```

一次最大 index check 可以让后续访问共享 proof。

它不会绕过安全语义，只是集中检查。

---

## 14. 检查 BCE

```bash
go build \
  -gcflags='-d=ssa/check_bce/debug=1' \
  ./...
```

重要：

```text
Found IsInBounds
```

表示：

> 这里仍然存在 bounds check。

不是“成功消除了”。

---

## 15. Inlining + BCE

callee：

```go
func byte3(b []byte) byte {
    return b[3]
}
```

caller：

```go
func parse(b []byte) byte {
    if len(b) < 4 {
        return 0
    }
    return byte3(b)
}
```

如果 inline，caller 的 guard 可以直接证明 `b[3]` 安全。

因此：

```text
Inlining
→ BCE
```

---

## 16. Interface Dispatch

Interface 代价不仅是 indirect call。

更重要的是：

```text
callee unknown
↓
cannot inline
↓
less escape information
↓
less constant propagation
↓
less BCE
```

所以 interface 的性能影响常常是“优化链被截断”。

---

## 17. Static Devirtualization

如果 compiler 能静态确定 interface receiver concrete type：

```text
interface call
→ direct call
```

之后可能继续 inline。

### Rule

不要因为看到 interface 就立即 concrete-ize。

先看 compiler 是否已能 devirtualize。

---

## 18. PGO Devirtualization

真实 profile 可能显示：

```text
95% receiver = *bytes.Reader
```

compiler 可以为 hottest receiver 构造 direct-call fast path，并保留 generic fallback。

这将：

```text
real production type distribution
```

反馈给 compiler。

---

## 19. PGO

典型：

```text
main/
├── main.go
└── default.pgo
```

`go build` 默认可自动使用。

也可：

```bash
go build -pgo=/path/profile.pprof
go build -pgo=off
```

### 关键

PGO profile 必须 representative。

不要把一次异常 P99 spike 的 profile 直接当成长期默认 profile。

---

## 20. Generics

不要把 Go generics 简化成：

```text
C++ template-style full monomorphization
```

也不能简化成：

```text
interface dispatch
```

具体 implementation 可能涉及：

- instantiation；
- shape；
- dictionary；
- compiler optimization。

### Rule

性能结论看最终生成代码，不看语言标签。

---

## 21. Dead Code / Constant Propagation

Inlining 后经常产生：

```text
known constant
known branch
known receiver
```

然后：

```text
constant propagation
↓
branch simplification
↓
DCE
```

这解释了为什么“小而清晰”的 hot helper 往往对 optimizer 友好。

---

## 22. GOSSAFUNC

```bash
GOSSAFUNC=ParseHeader go build ./...
```

生成 SSA HTML。

适合：

- 为什么 check 没消；
- 为什么 branch 仍在；
- proof pass 如何理解代码；
- rewrite / lowering。

不要把它作为所有性能问题的第一步。

---

## 23. Assembly

```bash
go build -gcflags='-S' ./...
```

或：

```bash
go tool objdump -s 'mypkg\.Foo' ./binary
```

适合确认：

- CALL 是否存在；
- bounds panic path；
- indirect call；
- atomic instruction；
- generated hot loop。

Assembly 是最终事实，但通常不是第一层诊断。

---

## 24. Disable Optimization 作为实验

```bash
go test -gcflags='all=-l' -bench=...
```

可实验：

- 是否高度依赖 inlining。

`-N` 可关闭优化用于研究。

这些只用于诊断实验，不用于 production build。

---

## 25. Compiler Investigation Workflow

```text
Profile
 ↓
Representative Benchmark
 ↓
-m=2/-m=3
 ↓
BCE diagnostics
 ↓
SSA
 ↓
Assembly
 ↓
A/B benchmark
 ↓
Production validation
```

---

## 26. Skill Rules

1. 先让 compiler 删除工作，再手工加速工作。
2. 不从 `&` / `new` / pointer receiver 猜 heap placement。
3. Inlining 主要价值是打开后续优化。
4. 不针对当前 inline budget 常量编程。
5. hot helper 保持简单、类型信息明确。
6. control flow 可以成为 compiler proof。
7. `Found IsInBounds` 表示 bounds check 仍存在。
8. 优先安全 BCE，而不是 unsafe bypass。
9. interface 成本包括优化链被阻断。
10. 不机械移除 interface。
11. PGO profile 必须代表真实 workload。
12. generics 性能以生成代码为准。
13. `does not escape` 不等于绝对 stack。
14. 逐级升级：`-m → BCE → SSA → asm`。
15. 如果 profile 不证明 hotness，就不要为 compiler trick 重构代码。
