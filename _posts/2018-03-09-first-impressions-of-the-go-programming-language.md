---
layout: post
title: First impressions of the Go programming language
comments: true
---

Most of my early career was spent using C++ (and to a lesser extent, good ol' C) and while it's a potent tool, it felt like a needlessly complicated and unproductive work environment. 40+ years of backward compatibility left a plethora of inconsistencies, and in my opinion, the ever-increasing set of language features that gets piled over every new iteration of the language only makes it worse. There are so many subtleties that it takes *years* to ~master it~ get to an acceptable level.

Considering these flaws, it took a very long time for C++ main influencers to standardize its best practices. Maybe no one had enough influence to do what [Crockford did with Javascript](http://shop.oreilly.com/product/9780596517748.do). At least, we now have the official [C++ Core Guidelines](https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md). I checked out of curiosity, and as of 2018-03-09, this is a continually expanding ~500 pages document that you have to read... after having read [The C++ Programming Language](https://www.amazon.com/C-Programming-Language-4th/dp/0321563840) by Bjarne Stroustrup which is itself a 1,368 pages book!

Heck, my own experience is that there is already a lot to learn *once* you have learned the language. Like, we have an actual product to build with its problem space. I can't fathom why someone fresh out of college in 2018 would choose to spend multiple years learning C++ before being considered productive when they can use something like Python instead and save the hassle.

Once I left Microsoft (and C++), I spent time working with ActionScript, Python, C#, PHP, JavaScript. As expected, I felt an order of magnitude more productive in these environments than with C++ (even with ActionScript, which isn't a bad language in itself). No more access violation, no more memory corruption and no more leaks. Even a mediocre developer can output useable code in any of these languages. Try doing that with C++ and watch your whole application burn down because of 20 lines causing random memory corruption.

It's true that all of these languages are also relatively complex beasts, although never to the same extent as C++. They do all miss is the ability to compile directly to the machine. Runtimes (JVM, CLR) are lovely and desirable for most cases, but they do tend to make binary distribution and installation harder, and although many argue the opposite, in practice there is a performance cost to them. Once in a while, it can become an issue.

So it's with these experiences in mind that I stumbled upon an article by Eric Raymond [listing alternatives to C](http://esr.ibiblio.org/?p=7711). He lists Go as C's most likely successor. Shortly after reading that, I noticed that a couple of companies I had a high opinion of were using Go on their server too. That caught my attention.

Go tries to keep what makes native code desirable (namely, speed) and bring what makes higher level languages more attractive: high productivity, code safety, concise. As far as I can tell, they didn't try to make a pretty language, but a pragmatic one. That makes for a very unusual combination that I wanted to check out.

Go is officially the language with *the smallest learning curve* that I know. I bought the excellent and aptly named book [The Go Programming Language](https://www.amazon.com/Go-Programming-Language-Alan-Donovan/dp/0134190440). In 400 pages, it covers the whole language, concurrency through goroutines, testing including performance benchmarks and code coverage, profiling and the rarely used low-level (unsafe) programming.

Here are some highlights that I found very promising so far.

## A Very Small Set of Features

Go is surprisingly small. Maybe a bit too small to my taste (I already miss generics), but I can appreciate the intention behind it. Every other language seems to have a natural tendency to grow and grow over time, which makes the language harder to master for newcomers. Often for features that are seldom used and only save little extra code. Go tries the opposite approach and I'm looking forward to seeing how that work out in practice.

So far, it's been a very, very promising start. After only a couple of weeks of on and off coding, not only do I already feel productive, but I also understand more of how the language implementation works than with, say Python or C# after an equivalent amount of time using it. This experience tells me that achieving mastery could be much faster than with the other languages familiar to me.

## Easy Memory management
Go makes memory management almost as simple as with higher level languages and it still provides decent control and transparency to the developer so that they can develop high-performance code without resorting to workarounds like mem pooling everything.

Go brings garbage collection to compiled code and it automatically manages where to allocate memory. It will aggressively use the stack if possible so that the allocation count can be much lower than with your typical language with a runtime.

It's even safe to return pointers to local objects. The compiler will simply silently allocate on the heap instead of the stack. Example:

```go
func doSomethingThenReturnPtr() [3]int {
  values := [3]int{1, 2, 3}
  // ...
  return &values // Pointer to the array
}
```

If you come from a C / C++ background, this is very different behaviour.

My initial concern was that because the allocation location (stack or heap) is implicit, it would be hard to write high-performance code that tries to limit heap memory allocations. It turns out that it's so easy to do a memory benchmark in Go that investigation is almost trivial.

That leads to my next point.

## Great Tools

For such as young language, I'm very impressed by the toolset around it. It comes out of the box with commands to do code formatting, run correctness tests (like unit tests and functional tests), performance benchmarks (CPU and memory), profiling, etc.

I started with the free and multi-platform [Visual Studio Code](https://code.visualstudio.com/) + the [Golang extension](https://code.visualstudio.com/docs/languages/go), and I must say that it's been better than expected. The packages are installed automatically, auto-completion and code navigation are excellent, running tests is a breeze. It uses the [Delve](https://github.com/derekparker/delve) debugger which I found quite nice although VSC + GE doesn't seem to allow the display of disassembly, which would be useful for a low-level language like Go. I'm hoping they'll expose this feature as Delve appears to support it already.

I think Microsoft did a splendid job with Visual Studio Code and the Go community provided some very potent productivity tools.

## A Very Consistent Codebase

I think that having *a* standard way of doing this is very important, but I attach much less importance to having *The* standard. In other words, it's the consistency that I find essential. Indeed, TAB VS space, or where to put the bracket are not very productive debates so I enjoy when a language comes with clear best practices so that everybody can shut up about it :-).

The Go authors place a lot of value on consistency and code cleanness. For example, the go command will automatically reformat your code when you build. Small things that go unnoticed in other languages, like unused imports or variables, are compile-time *bugs*, not warnings! All these little attentions make Go code very predictable to read. I read code from the builtin package, and it was all very simple and natural to read.

Alternatively, I believe that the C++ STL is the ugliest and most opaque code I've been blessed to lay my eyes on.

I was concerned that Go's style restrictions could become a burden, but luckily, tons of tools come out of the box to automate these tasks so, from the first line of Go code I wrote, I never had to worry about them. In Visual Studio Code + the Golang extension, this is all done every time I hit CTRL+S.

## No Inheritance!

Go doesn't support inheritance! This was a surprise to me. I'm the only person I know that dislike inheritance so much! I'm not alone! Instead, it makes it easy to replace it with  composition with the following syntactic sugar:

```go
type myBaseType struct {
    foo int
    bar int
}

type myType struct {
    myBaseType
    foo2 int
}

func doAccess() {
    mt := myType{}
    mt.foo = 2
    fmt.Printf("%d\n", mt.foo) // 2
}
```

I noticed from experience that code using inheritance could almost always use composition instead and that very often when inheritance was chosen, as requirements would evolve and features were being added, the code would tend to (de)evolve into something hard to maintain and bug-prone. Once we got there, it was also harder to disentangle code using inheritance as the [coupling](https://en.wikipedia.org/wiki/Coupling_(computer_programming)) is very strong: it favours usage of internal class member values instead of a clean communication interface between two (or more) pieces of code.

In the worse case, I've seen subclasses that couldn't realistically be broken down anymore because they were so big and so intermingled with other classes that refactoring them would have taken weeks and most likely introduced many regressions. So instead of risking their whole career, people kept bolting new features on them, which only made the problem worse.

When I co-founded Ululab, one simple test I made early on was always to try to use composition instead of inheritance in the Slice Fractions codebase. It started as an experiment really, but the results were surprisingly good. I almost never used inheritance from then on.

Furthermore, whenever we hired a contractor developer to help us to develop new features, I kept noticing that when they started to introduce inheritance, they'd struggle to evolve that code without getting tangled in quality issues (whether bugs, convolutedness, or both).

## Much More

There are many other things to like about go like its simple concurrency model (goroutines and channels), helper functions + slices as simple views for regular arrays that make the need for higher level containers like C++ std::vector<> or C# List<> moot.

A lot of people seem to be concerned by the absence of exceptions in Go. While unusual, I don't think it's that big of a deal and for very large project, we know it scales.

In my old team, we used C++ without exception. One of the reasons being because much of the Windows codebase is C-based and C code isn't exception safe. Exceptions would have led to leaked resources and all kind of similar badness.

Depending on which team you were on, we had two patterns for error management that didn't rely on exceptions.

The first is "triangle code":

```c++
HRESULT MyFunction()
{
  HRESULT hr = Function1();
  if (SUCCEEDED(hr))
  {
    hr = Function2();
    if (SUCCEEDED(hr))
    {
      // ...
    }  
    else
    {
      // local error handling
    }
  }  

  if (FAILED(hr))
  {
    // function-level error handling
  }

  return hr
}
```

While not pretty, it's consistent and predictable. If "the triangle" gets too big for the function to be easily understandable, it's a sign you should likely make a subfunction.

The other one was the [goto error pattern](https://stackoverflow.com/questions/788903/valid-use-of-goto-for-error-management-in-c).

```c++
HRESULT MyFunction()
{
  HRESULT hr = Function1();
  if (FAILED(hr))
  {
    goto Error;
  }

  hr = Function2();
  if (FAILED(hr))
  {
    // local error handling
    goto Error;
  }

  return S_OK;

Error:
  // function-level error handling
  return hr;
}
```

That pattern looks a bit like a primitive version of exception handling. It's one of the very few cases where using the goto command is considered OK.

Go essentially use the goto error pattern, but with a much prettier and safer wrap: the 'defer' statement. 'defer' will execute a function (which can be a closure) in a predictable order once the function ends.

```go
func doAccess() error {

  // Allocate some resources, open a file, etc.

  defer func() {
        fmt.Println("function-level cleanup") // Will always be called
    }()

  e := function1()
  if e != nil {
    return e
  }

  defer func() {
    fmt.Println("function1() cleanup")
  }()

  e := function2()
  if e != nil {
    return e
  }

  return nil // No error

  // Console:
  //   function1() cleanup
  //   function-level cleanup
}
```

## Not So Nice

While there is a lot that I like about Go, I have a, perhaps unfounded, concern about their interface implementation as a bug factory. More specifically the dual way that they handle [interfaces to nil values](https://golang.org/doc/faq#nil_error). In a nutshell:

```go
func nilInterfaces() {

    var v []int

    var interf interface{} // Equivalent to a base Object in C#/Java or void* in C++

    interf = v
    if interf == nil {
        fmt.Println("This interface points to a nil slice")
    } else {
        fmt.Println("An interface that points to a nil object is not nil")
    }

    interf = nil
    if interf == nil {
        fmt.Println("This is a nil interface")
    }

    // Console:
    //   An interface that points to a nil object is not nil
    //   This is a nil interface
}

```

For someone coming from another language, this is completely counter-intuitive. From what I read, it's the inevitable consequence of a series of choices made by the language designer. There are best practices to make this less of an issue (assign nil directly to the interface and don't return nil from a function as an error. Use an error object instead).

I'll see if that indeed becomes much of a hurdle or if it's relatively contained in real life.

No generics is also a big deal to me. Even the containers package has to rely on typecasting, and like many, I don't understand the reasons for the tradeoff.

The language designers are aware of the previous issues, so I hope that future versions of Go will bring better solutions.

In the meantime, they don't get in my way for the moment, and it's still many times less problematic to me than working in C++ so it's not bad at all :-).
