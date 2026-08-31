# 06. Unsafe / Runtime Boundary / Zero-Copy

## 1. 总纲

`unsafe` 不是“让 Go 更快”的按钮。

它用于：

> 在 copy、representation、FFI 或 runtime boundary 已经被证明是瓶颈时，绕过部分语言安全成本。

进入 unsafe 后，程序员接管：

- ownership；
- lifetime；
- aliasing；
- alignment；
- GC visibility；
- portability；
- runtime/compiler compatibility。

---

## 2. Escalation Levels

### L0 Safe Go

- compiler optimization；
- `copy`；
- preallocation；
- pool；
- layout。

### L1 Public Unsafe APIs

- `unsafe.String`；
- `unsafe.StringData`；
- `unsafe.Slice`；
- `unsafe.SliceData`；
- `unsafe.Add`。

### L2 OS / FFI

- mmap；
- cgo；
- Pinner；
- cgo.Handle。

### L3 Compiler Contracts

- `//go:noescape`；
- cgo noescape/nocallback。

### L4 Runtime Internal

- `//go:nosplit` performance use；
- `//go:linkname` private runtime；
- procPin；
- manual noescape trick。

默认只允许逐级升级。

---

## 3. `[]byte → string` Zero-Copy

Safe：

```go
s := string(b)
```

通常建立独立 immutable representation。

Unsafe：

```go
func bytesToString(b []byte) string {
    if len(b) == 0 {
        return ""
    }
    return unsafe.String(unsafe.SliceData(b), len(b))
}
```

不 copy。

### Contract

只要 string 仍存在：

```text
backing bytes must not mutate
must not be reused
```

---

## 4. Ownership

普通 copy：

```text
[]byte owns A
string owns B
```

Zero-copy：

```text
[]byte ─┐
        ├→ backing A
string ─┘
```

因此：

> Zero-copy is an ownership optimization.

你省掉：

- allocation；
- memcpy；
- bandwidth。

换来：

- lifetime coupling；
- aliasing；
- pool restrictions；
- retention risk。

---

## 5. Pool + Zero-Copy 危险组合

错误：

```go
buf := pool.Get().([]byte)
fill(buf)

s := unsafe.String(unsafe.SliceData(buf), len(buf))

pool.Put(buf[:0])

return s
```

别的 goroutine 可修改 backing bytes。

后果：

- string content mutation；
- data race；
- map key / hash inconsistency；
- corrupted logs。

---

## 6. `string → []byte`

用 `unsafe.Slice(unsafe.StringData(s), len(s))` 构造 writable slice alias 风险更高。

`[]byte` API 语义允许 mutation，而 string 是 immutable。

因此：

```text
string → writable []byte alias
```

默认应判为 High Risk。

---

## 7. Zero-Copy Retention

16 MiB buffer 中截取 100B token。

Zero-copy token 可能保活：

```text
16 MiB
```

Copy：

```text
100B allocation
```

反而能释放大 backing storage。

因此必须比较：

```text
copy cost
vs
retention cost
```

---

## 8. Public Unsafe APIs

现代代码优先：

- `unsafe.String`；
- `unsafe.StringData`；
- `unsafe.Slice`；
- `unsafe.SliceData`；
- `unsafe.Add`。

历史：

- `reflect.StringHeader`；
- `reflect.SliceHeader` giant-array conversion。

后者应归类 Historical Unsafe Pattern。

---

## 9. `uintptr` 不是 Pointer

```go
u := uintptr(unsafe.Pointer(p))
```

对 GC：

```text
u = integer
```

不是 reference。

即使它保存一个地址，也不会因此保活对象。

### Rule

不要长期用 integer 保存 Go object pointer identity。

---

## 10. Pointer Arithmetic

优先：

```go
unsafe.Add(ptr, offset)
```

而不是：

```go
unsafe.Pointer(uintptr(ptr) + offset)
```

尤其不能长期保存 uintptr 后任意时间转换回来。

官方合法 unsafe patterns 本质上都围绕：

```text
GC lifetime remains visible
```

展开。

---

## 11. Source Scope ≠ GC Lifetime

变量还“写在函数作用域里”不代表 compiler 认为它仍 live。

最后一次实际使用后，value 可能变 unreachable。

这在：

- finalizer；
- cleanup；
- syscall；
- FFI；

场景很重要。

---

## 12. `runtime.KeepAlive`

```go
syscallSomething(obj.fd)
runtime.KeepAlive(obj)
```

定义：

```text
obj remains reachable until here
```

它不是修复非法 unsafe pattern 的万能胶。

`uintptr` misuse 不会因为 KeepAlive 自动变合法。

---

## 13. cgo Pointer Rules

Go pointer 交给 C 时，GC 不知道 C 如何保存它。

因此需要明确：

- 只在调用期间使用；
- 跨调用保存时必须 pin；
- pointer-containing Go value 有更严格限制。

---

## 14. `runtime.Pinner`

如果 C 需要在 Go call 结束后继续持有某个 Go memory address：

```go
var p runtime.Pinner
p.Pin(obj)

C.remember(unsafe.Pointer(obj))

...

C.forget()
p.Unpin()
```

Pinner 表达：

```text
address stability / pinning contract
```

不要因为当前 Go GC non-moving 就跳过正式 contract。

---

## 15. `runtime/cgo.Handle`

需要 C 保存 Go object identity 时，优先：

```go
h := cgo.NewHandle(value)
```

给 C 保存 integer handle。

回来：

```go
v := h.Value()
```

结束：

```go
h.Delete()
```

比手工：

```text
Go pointer → uintptr registry
```

安全得多。

---

## 16. `#cgo noescape`

对于严格保证不会保存 pointer 的 C function，可声明 noescape，帮助 compiler 避免保守 heap escape。

这是：

```text
correctness contract
```

不是普通 hint。

声明错误可能导致：

- dangling pointer；
- memory corruption；
- crash。

---

## 17. `#cgo nocallback`

如果 C function 保证绝不 callback Go，可以减少 cgo boundary 的一些 runtime 准备。

只适合：

- 高频 FFI；
- 明确 C contract；
- benchmark 证明 boundary overhead 是问题。

---

## 18. mmap

mmap：

```text
file-backed pages
→ process virtual address
→ direct byte view
```

可减少显式 read/copy。

适合：

- DB/index；
- large immutable dataset；
- binary assets。

但 mmap memory：

```text
not normal Go heap
```

因此：

- HeapAlloc 不完整反映；
- RSS 可能包含；
- GOMEMLIMIT 不是 process RSS hard limit。

---

## 19. mmap Lifetime

```go
s := unsafe.String(...)
munmap(...)
use(s)
```

是典型 use-after-unmap。

Zero-copy 后：

```text
Go value lifetime
must not exceed
mapping lifetime
```

---

## 20. `//go:noescape`

用于没有 Go body 的 function（常见 assembly）：

```go
//go:noescape
func hash(p *byte, n uintptr) uint64
```

告诉 compiler：

```text
pointer does not escape this external implementation
```

本质：

> Human-assisted escape analysis contract.

撒谎会破坏 stack lifetime correctness。

---

## 21. `//go:nosplit`

它用于 low-level runtime constraints。

不是：

```text
//go:fast
```

普通代码为了省 stack-check 指令使用属于错误方向。

分类：

```text
Runtime Internal / Do Not Recommend
```

---

## 22. `//go:linkname`

允许突破 package encapsulation 并链接 private symbol。

风险：

- compatibility；
- type-safety；
- runtime implementation changes。

例如 private `runtime.procPin` 技巧属于 Runtime Internal。

### Rule

标准库使用 runtime private contract，不代表第三方代码获得同样兼容性保证。

---

## 23. Manual Noescape Trick

通过 pointer→integer 等方式欺骗 escape analysis，本质是对 compiler 撒谎。

一旦 pointer 实际 escape：

```text
object may stay stack
caller keeps pointer
→ memory corruption
```

分类：

```text
Runtime Archaeology
Never auto-recommend
```

---

## 24. checkptr / race / vet

unsafe-heavy package 至少应拥有：

```bash
go vet ./...
go test -race ./...
go test -gcflags=all=-d=checkptr=2 ./...
```

这些用于 correctness，不是 performance baseline。

---

## 25. Architecture Testing

unsafe / mmap / FFI 还应考虑：

- amd64 / arm64；
- alignment；
- endianness；
- OS semantics；
- 32/64-bit assumptions。

任何隐含：

```text
pointer fits certain tag bits
```

的设计尤其危险。

---

## 26. Microbenchmark Trap

Safe：

```text
200 ns/op
```

Unsafe：

```text
2 ns/op
```

看起来 100x。

但如果这一步只占系统：

```text
0.2%
```

系统收益仍非常低。

不能仅因为 local speedup 巨大就接受 ownership / unsafe complexity。

---

## 27. Unsafe Isolation

推荐：

```text
internal/zerocopy/
internal/native/
internal/ffi/
```

将 unsafe 封装到最小 surface。

避免：

```text
business package 到处 unsafe.Pointer
```

---

## 28. API 应暴露危险语义

不要：

```go
func ToString(b []byte) string
```

内部偷偷 zero-copy。

可考虑：

```go
func UnsafeBytesToString(b []byte) string
```

或者完全限制在 internal。

目标：

```text
make ownership contract visible in code review
```

---

## 29. Escalation Decision

```text
hot copy/allocation?
      │
      ▼
compiler can remove?
      │
 safe refactor?
      │
 ownership clearly defined?
      │
 public unsafe API
      │
 benchmark
      │
 end-to-end validation
```

`linkname` / `nosplit` / manual noescape 不属于普通 ladder。

---

## 30. Skill Rules

1. Unsafe 必须有 profile 证据。
2. 优先现代 public unsafe API。
3. Zero-copy 本质是 ownership change。
4. bytes→string 要保证 immutable backing lifetime。
5. string→writable bytes 默认高风险。
6. Zero-copy 必须评估 retention。
7. `uintptr` 不保活 Go object。
8. 优先 `unsafe.Add`。
9. KeepAlive 定义 liveness boundary，但不修复非法 pointer pattern。
10. FFI 跨调用 pointer 要遵守 pinning。
11. cgo.Handle 优先于手工 pointer identity registry。
12. noescape/nocallback 是 correctness contract。
13. mmap 不属于普通 Go heap accounting。
14. view 生命周期不能超过 mapping。
15. `//go:noescape` 仅用于低层 external implementation。
16. `//go:nosplit` 不是性能 hint。
17. `//go:linkname` 是红区。
18. runtime hack 默认只用于解释/审计。
19. unsafe package 必须有更严格 correctness tests。
20. microbenchmark 收益必须传导到系统指标。
21. unsafe surface 最小化并隔离。
22. API 明示 alias/lifetime contract。
