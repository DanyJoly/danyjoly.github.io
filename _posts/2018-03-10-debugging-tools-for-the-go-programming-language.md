---
layout: post
title: Debugging tools for the Go programming language
comments: true
---

I've been dabbling with Go recently as I was very interested to know what it felt like to work with it to solve production-like problems.

One thing I'm very impressed with Go is the ability to test and benchmark right out of the box. No complicated dependency to add, or framework to review, it all just works.

# Benchmarks

During one of those sessions where I measured the performance of a function I wrote, I noticed something peculiar:

```
BenchmarkMyFunction-4             10     141429310 ns/op      320000 B/op       10000 allocs/op
PASS
ok      algo    1.670s
```

I wasn't expecting any allocation and yet, there were 10,000 allocations of 32 bytes each! That gave me the opportunity to test how powerful were Go debugging tools.

After a few minutes trying to reduce the repro to a minimum, I ended up with this:

```go
// --- Repro file ---
func memAllocRepro(values []int) *[]int {

  for {
        break
    }

    return &values
}

// --- Benchmark file ---
func BenchmarkMemAlloc(b *testing.B) {

    values := []int{1, 2, 3, 4}

    for i := 0; i < b.N; i++ {
        memAllocRepro(values)
    }
}
```

```
BenchmarkMemAlloc-4       50000000            40.2 ns/op          32 B/op           1 allocs/op
PASS
ok      memalloc_debugging    2.113s
Success: Benchmarks passed.
```

At that point, I thought this was very peculiar! If I either remove the useless for loop or if I return the slice directly instead of a pointer to the slice, no heap allocation is done:

```go
// --- Repro file ---
func noAlloc1(values []int) *[]int {

    return &values // No alloc!
}

func noAlloc2(values []int) []int {
  for {
        break
    }

    return values // No alloc!
}

// --- Benchmark file ---
func BenchmarkNoAlloc(b *testing.B) {

    values := []int{1, 2, 3, 4}

    for i := 0; i < b.N; i++ {
        noAlloc1(values)
        noAlloc2(values)
    }
}
```

```
BenchmarkNoAlloc-4       300000000             4.20 ns/op           0 B/op           0 allocs/op
PASS
ok      memalloc_debugging    1.756s
Success: Benchmarks passed.
```

If you already know Go pretty well, you may have already picked up what happened. For the others, follow me :-).

# Disassembly Dump

A wee bit confused at the results, and because I wanted to try the lower level debugging tools offered by the Go ecosystem, I decided to try my luck by looking at the disassembly, for which I found Go documentation [here](https://golang.org/cmd/objdump/) and [here](https://golang.org/doc/asm). Indeed, we can see a heap allocation being made in memAllocRepro:

```
> go tool objdump main.exe > disasm.txt

-- File Content --
TEXT main.memAllocRepro(SB) memalloc_debugging/main.go
  (...)
  main.go:10        0x44ce34        488d0525880000        LEAQ runtime.types+34656(SB), AX
  main.go:10        0x44ce3b        48890424        MOVQ AX, 0(SP)                
  main.go:10        0x44ce3f        e8bcebfbff        CALL runtime.newobject(SB)        
  (...)
```

We can see that the address *runtime.types+34656* (the address of the type of the object to allocate?) is loaded in the register AX (LEAQ runtime.types+34656(SB), AX), then onto the stack (MOVQ AX, 0(SP)) as arguments for the function call runtime.newobject. Since Go is open source, It's easy to confirm that this is indeed the object type by looking at the source code for runtime.newobject defined in [malloc.go](https://golang.org/src/runtime/malloc.go):

```go
// implementation of new builtin
// compiler (both frontend and SSA backend) knows the signature
// of this function
func newobject(typ *_type) unsafe.Pointer {
    return mallocgc(typ.size, typ, true)
}LEAQ runtime.types+34656(SB), AX
```

# Delve Debugger

Unfortunately, mapping runtime.types+34656 to \*\_type is a wee bit opaque to me. Ideally, I'd be able to explore this under the debugger.

I'm working from the [Visual Studio Code IDE](https://code.visualstudio.com/) + [Golang extension](https://code.visualstudio.com/docs/languages/go), which integrates the [Delve](https://github.com/derekparker/delve) debugger. While it's not as powerful as more mature solutions from other languages, is still quite acceptable.

Unfortunately, while Delve has basic support for disassembly, Visual Studio code doesn't expose it. I decided to pop the [Delve documentation](https://github.com/derekparker/delve/tree/master/Documentation/cli) and launch it from the command line.

```
> dlv exec ./main.exe
(dlv) b memAllocRepro
Breakpoint 1 set at 0x44ce26 for main.memAllocRepro() memalloc_debugging/main.go:10
(dlv) c
> main.memAllocRepro() memalloc_debugging/main.go:10 (hits goroutine(1):1 total:1) (PC: 0x44ce26)
(...)
(dlv) b *0x44ce3f <-- Just before the call to runtime.newobject
Breakpoint 2 set at 0x44ce3f for main.memAllocRepro() memalloc_debugging/main.go:10
(dlv) c
> main.memAllocRepro() memalloc_debugging/main.go:10 (hits goroutine(1):1 total:1) (PC: 0x44ce3f)
(dlv) regs
(...)
   Rax = 0x0000000000455660
 (...)
```
Unfortunately, while I now have the address to the \*\_type, I can see in the runtime source code that accessing the string name requires non-trivial pointer manipulation. It's not something I find very palatable to do manually.

# Compiler Optimizations

I decided to go searching for better debugging tools. Since I don't have many friends or colleagues well versed in Go yet, I decide to try my luck and ask on [Stack Overflow](https://stackoverflow.com/questions/49203089/golang-doing-unexpected-heap-memory-allocation?answertab=votes#tab-top). Luckily enough, peterSO was kind enough to rapidly jump in and use go compiler flags to show me what's happening. Here is what he found:

```
> go build -a -gcflags="-m -m" main.go
# command-line-arguments
.\main.go:10:6: cannot inline memAllocRepro: unhandled op FOR
.\main.go:5:6: cannot inline main: non-leaf function
.\main.go:19:6: can inline noAlloc1 as: func([]int) *[]int { return &values }
.\main.go:24:6: cannot inline noAlloc2: unhandled op FOR
.\main.go:16:9: &values escapes to heap
.\main.go:16:9:         from ~r1 (return) at .\main.go:16:2
.\main.go:10:37: moved to heap: values
.\main.go:6:17: []int literal escapes to heap
.\main.go:6:17:         from values (assigned) at .\main.go:6:9
.\main.go:6:17:         from values (passed to call[argument escapes]) at .\main.go:7:19
.\main.go:21:9: &values escapes to heap
.\main.go:21:9:         from ~r1 (return) at .\main.go:21:2
.\main.go:19:32: moved to heap: values
.\main.go:24:31: leaking param: values to result ~r1 level=0
.\main.go:24:31:        from ~r1 (return) at .\main.go:28:2
```

The go build -gcflags argument let us pass arguments down to the go compiler:

```
>go help build
usage: go build [-o output] [-i] [build flags] [packages]
(...)
-gcflags '[pattern=]arg list'
        arguments to pass on each go tool compile invocation.
(...)
```

... and the '-m -m' arguments tells the compiler to print the optimization decisions.

```
> go tool compile -?
flag provided but not defined: -?
usage: compile [options] file.go...
(...)
  -m    print optimization decisions
(...)
```

This is very interesting because we see clearly that the compiler decides to put &values on the heap:

```
.\main.go:21:9: &values escapes to heap
.\main.go:21:9:         from ~r1 (return) at .\main.go:21:2
```

We also see why noAlloc1 doesn't allocate on the heap:
```
.\main.go:19:6: can inline noAlloc1 as: func([]int) *[]int { return &values }
```

Oops, I should have caught that by looking at the disassembly!

That investigation showed a hole in my burgeoning knowledge of Go. *I was under the assumption that slices we passed by reference*. If that were the case, returning a pointer to the reference wouldn't have required a heap allocation. But it's not. *Everything* in Go is passed by value. That means that to be able to pass a pointer to the slice values[]int, it needs to be on the heap. Otherwise, it would have been stored on the stack with the function local variables. Its lifetime wouldn't have been longer than the function call.

I was so confident that I read somewhere that slices were passed by reference that I had to check back where I read that. Sure enough, I was wrong:

> Arguments are passed by value, so the function receives a copy of each argument; modifications to the copy do not affect the caller. However, if the argument contains some kind of reference, like a pointer, slice, map, function, or channel, then the caller may be affected by any modifications the function makes to variables indirectly referred to by the argument.
- Donovan, Alan A. A.. The Go Programming Language (Addison-Wesley Professional Computing Series) (pp. 120-121). Pearson Education. Kindle Edition.

So there you go. Like it so often happens, hard bugs are when your mental model doesn't fit reality.

# Reflection and Memory Layout

Something I found interesting in Go is the ability to explore the memory layout of structs. This information is context dependent. I assume it can change from architecture to architecture and from one Go version to the other, but it's interesting nonetheless.

I won't cover it over here since I found it to be already well explained in this [article](https://syslog.ravelin.com/go-and-memory-layout-6ef30c730d51) by Phil Pearl.

Other references I found interesting:

Go Functions In Assembly: <https://github.com/golang/go/files/447163/GoFunctionsInAssembly.pdf>

Profiling Go Programs: <https://blog.golang.org/profiling-go-programs>
