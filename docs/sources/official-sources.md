# Official Sources

> 优先使用 Go 官方文档、release notes、标准库文档和 runtime/compiler 源码。  
> 以下链接是本知识体系的主要依据。

## Current Release

- Go 1.27 Release Notes  
  https://go.dev/doc/go1.27
- Go 1.27 Release Blog  
  https://go.dev/blog/go1.27
- Go 1.26 Release Notes（Green Tea GC）  
  https://go.dev/doc/go1.26

## GC / Memory

- A Guide to the Go Garbage Collector  
  https://go.dev/doc/gc-guide
- runtime allocator source  
  https://go.dev/src/runtime/malloc.go
- runtime scavenger source  
  https://go.dev/src/runtime/mgcscavenge.go
- runtime pacer source  
  https://go.dev/src/runtime/mgcpacer.go
- runtime metrics  
  https://pkg.go.dev/runtime/metrics

## Compiler

- Compiler README  
  https://go.dev/src/cmd/compile/README
- compile command documentation  
  https://pkg.go.dev/cmd/compile
- escape analysis source  
  https://go.dev/src/cmd/compile/internal/escape/escape.go
- devirtualization source  
  https://go.dev/src/cmd/compile/internal/devirtualize/
- BCE checker  
  https://go.dev/src/cmd/compile/internal/ssa/checkbce.go
- PGO guide  
  https://go.dev/doc/pgo

## Concurrency / Runtime

- Go Memory Model  
  https://go.dev/ref/mem
- sync package  
  https://pkg.go.dev/sync
- sync/atomic  
  https://pkg.go.dev/sync/atomic
- internal Mutex implementation  
  https://go.dev/src/internal/sync/mutex.go
- runtime lock-free stack  
  https://go.dev/src/runtime/lfstack.go

## Unsafe / FFI

- unsafe package  
  https://pkg.go.dev/unsafe
- runtime.KeepAlive / runtime docs  
  https://pkg.go.dev/runtime
- runtime/cgo  
  https://pkg.go.dev/runtime/cgo
- cmd/cgo  
  https://pkg.go.dev/cmd/cgo
- runtime.Pinner  
  https://pkg.go.dev/runtime#Pinner

## Profiling / Benchmark

- Diagnostics  
  https://go.dev/doc/diagnostics
- runtime/pprof  
  https://pkg.go.dev/runtime/pprof
- net/http/pprof  
  https://pkg.go.dev/net/http/pprof
- runtime/trace  
  https://pkg.go.dev/runtime/trace
- Flight Recorder  
  https://go.dev/blog/flight-recorder
- testing.B.Loop  
  https://go.dev/blog/testing-b-loop
- testing package  
  https://pkg.go.dev/testing
- x/perf / benchstat  
  https://pkg.go.dev/golang.org/x/perf

## CPU / OS

- golang.org/x/sys/cpu  
  https://pkg.go.dev/golang.org/x/sys/cpu
- Linux perf stat manual  
  https://man7.org/linux/man-pages/man1/perf-stat.1.html

## Source Quality Rule

对版本敏感性能事实，优先级：

```text
1. 当前 release notes
2. 当前标准库/package docs
3. 当前 runtime/compiler source
4. Go official blog/design docs
5. 外部 benchmark/blog（仅作为案例）
```

不要用旧博客替代当前 runtime/compiler 事实。
