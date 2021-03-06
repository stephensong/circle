# Circle
The C++ Automation Language  
2019  
Sean Baxter

> _"it's like c++ that went away to train with the league 
                     of shadows and came back 15 years later and became batman"_

## Contents

** **[Circle for GPU](NVVM.md)**

[Abstract](#abstract)  
[TLDR;](#tldr)  
[Hello world](#hello-world)  
[Hello harder](#hello-harder)  
[What is Circle](#what-is-circle)  
[Why I wrote Circle](#why-i-wrote-circle)  
[Circle is a triangle](#circle-is-a-triangle)  
[Meta statements](#meta-statements)  
[Integrated interpreter](#integrated-interpreter)  
[Same-language reflection](#same-language-reflection)  
[Duff's device](#duffs-device)  
[Dynamic names](#dynamic-names)  
[Automating enums](#automating-enums)  
[Introspection keywords](#introspection-keywords)  
[Introspection on enums](#introspection-on-enums)  
[Object serialization](#object-serialization)  
[Better object serialization](#better-object-serialization)  
[Metafunctions](#metafunctions)  
[Parsing command specifiers](#parsing-command-specifiers)  
[Contract safety](#contract-safety)  
[Code injection](#code-injection)  
[Configuration-oriented programming](#configuration-oriented-programming)  
[Building functions from JSON](#building-functions-from-json)  
[Errors in configuration files](#errors-in-configuration-files)  
[Kernel parameterization](#kernel-parameterization)  
[Querying JSON](#querying-json)  
[Querying Lua](#querying-lua)  
[Template metaprogramming](#template-metaprogramming)  
[SFINAE](#sfinae)  
[Typed enums](#typed-enums)  
[case-typename](#case-typename)  
[Variant](#variant)  
[Generic dispatch](#generic-dispatch)  

## Abstract

> Circle is a compiler that extends C++17 for new introspection, reflection and compile-time execution. An integrated interpreter allows the developer to execute any function during source translation. Compile-time flavors of control flow statements assist in generating software programmatically. The configuration-oriented programming paradigm captures some of the conveniences of dynamic languages in a statically-typed environment, helping separate a programmer's high-level goals from the low-level code and allowing teams to more effectively reason about their software.

## TLDR;

Circle is C++ 17 with three new things:
1. [An integrated interpreter](#integrated-interpreter) - Run anything at compile time.
1. [Same-language reflection](#same-language-reflection) - Use _meta control flow_ to guide source translation. Real statements inside meta scopes fall through and nest in the innermost real scope, allowing you to programmatically deposit statements in your AST.
1. [Introspection keywords](#introspection-keywords) - Access data member and enumerator info at compile time. Use meta control flow to iterate over all members or enumerators and apply transformations.

These three feature domains work together to bring new paradigms to C++ development.

The [configuration-oriented programming](#configuration-oriented-programming) paradigm practices a new sort of separation of concerns: identifying configurable dimensions in your software and separating _logic_ from _code_. Settings for these configurable dimensions are stored in external sources, like JSON files or Lua scripts. How do we process?
1. While your program is compiling, use the **interpreter** to load these configuration resources.
1. Use **same-language reflection** to programmatically generate functionality from them. 
1. **Introspection** facilitates communication between the loaded resources and your C++ code, automatically and safely converting between C++ structs and Lua tables, for example.

Disciplined application of these ideas allows you to make generic Circle libraries which consume application-specific configuration. The generic code goes into a library. The configuration stays with your source. 

Circle takes you to the next level in generic programming, while being very easy to learn. If you know C++, you'll understand Circle within hours. 

Circle's primary syntactic element is the `@meta` keyword, which runs the prefixed statement during source translation (or during template instantiation in dependent contexts).
* Meta expression statements are executed by the interpreter at compile time.
* Meta object declarations have automatic storage duration over the lifetime of the enclosing scope. This scope may be a namespace, a _class-specifier_, an _enum-specifier_ or a block scope. For example, you can create meta objects inside a class definition to help define the class; these meta objects are torn down when the class definition is complete.
* Meta control flow is executed at compile time. These loops are structurally unrolled, and the child statement is _translated into AST at each iteration_. Programs are grown algorithmically as if by deposition: each real statement in a meta scope deposits itself in the innermost enclosing scope. By controlling how real statements are visited with meta control flow, we can guide the program's creation.

Compiler details:
* Custom C++ frontend.
* LLVM backend. 
* [Itanium C++ ABI](http://itanium-cxx-abi.github.io/cxx-abi/) support for wide compatibility. 
* gdb/DWARF debug metadata.
* Only 100,000 lines for fast feature development. 

The Circle compiler is nearing the end of private development. I look forward to a public release.

## Hello world

```cpp
template<typename... types_t>
struct ??? {
  @meta for(int i = 0; i < sizeof...(types_t); ++i)
    types_t...[i] @(i); 
};
```
Can you figure out what this class is? It's written in a new language, Circle, which brings C++ constructs into the compile-time domain. 

> There's this `@meta` tag which I've never seen, but "meta" sounds like "metaprogramming," so it's likely related to that.

Right, `@meta` executes that statement at compile time.

> The loop inside the _class-specifier_ isn't allowed in C++, but counting through elements of the template parameter pack seems good. The `...[]` operator isn't C++ either, but that has to subscript the parameter pack at `i` to get a type out of it.

`...[]` works on all four kinds of packs, too.

> `@(i)` is like nothing I've seen before. If the statement were a class _member-specifier_, that's where the name of the member would go. `@(i)` probably resolves to an identifier that's a function of its argument, in this case the loop index. We can't simply name each data member `x` or `data`, because after the first loop iteration we'd be walking into redeclaration errors, so we'd need a way to programmatically generate names.

Very astute, reader.

> Basically, when the class template is instantiated it looks like we're looping over types in the parameter pack and creating a data member for each type and giving it a numeric name. It's a tuple, isn't it!

```cpp
template<typename... types_t>
struct tuple_t {
  @meta for(int i = 0; i < sizeof...(types_t); ++i)
    types_t...[i] @(i); 
};
```
This is the dream that inspired Circle. We're able to use C++'s imperative constructs to write code in the way usually we think about coding. In this case, for each element in a parameter pack, make a non-static data member of that type. It's easy to read. It's easy to write. It's easy to learn. You don't have to rely on expert developers of template libraries to provide generic tools when Circle has cleverly repurposed C++ to allow you to write it yourself.

## Hello harder

```cpp
template<typename... types_t>
struct ??? {
  enum { count = sizeof...(types_t) };
  @meta @mtype x[] { @dynamic_type(types_t)... };
  @meta std::sort(x, x + count);
  @meta size_t count2 = std::unique(x, x + count) - x;
  typedef tuple_t<@pack_type(x, count2)...> type;
};
```
Can you figure out this class?

> There's quite a lot going on. Some new operator `@dynamic_type` is being used on the parameter pack and is expanded into an array `x`. Okay, so `x` likely has `count` elements, holding some kind of "dynamic type" of each of the template parameters. 

`@mtype` is a new builtin type that holds a type. It's really just an opaque pointer to type info that's already maintained by the compiler. But it has overloads for all the comparison operators, so it's easy to work with.

> So you sort and unique this array using functions from `<algorithms>`. Much easier than trying to manipulate those type list things. `@dynamic_type` appears to box a type into an `@mtype` variable. It has to be that `@pack_type` goes the other direction and packages the array of `@mtype` variables as a parameter pack.

The actual complement to `@dynamic_type` is (wait for it) `@static_type`, which takes an `@mtype` variable and yields a type. You can use it wherever you expect a _type-id_ production. Sort of like `decltype`, but instead of getting the type from an operation it gets it from a variable. `@pack_type` is the next level of this, and sweeps up an array of `@mtype` and puts them into an unexpanded parameter pack.

> It's clear to me now. 
> 1. Expand the pack out into an array of `@mtype`.
> 2. Sort and unique the type array.
> 3. Expand that uniqued array into a `tuple_t` class template.
> 
> What you've got here is a unique-type tuple.

```cpp
template<typename... types_t>
struct unique_type_tuple_t {
  enum { count = sizeof...(types_t) };
  @meta @mtype x[] { @dynamic_type(types_t)... };
  @meta std::sort(x, x + count);
  @meta size_t count2 = std::unique(x, x + count) - x;
  typedef tuple_t<@pack_type(x, count2)...> type;
};
```
Are you surprised it works? We've just created an array of `@mtype` variables that allow you to manipulate types as if they were variables inside of a class definition and sort and unique them at compile time using standard functions from `<algorithm>`. Lastly, we were able to repackage this array of type-representing variables back up into a parameter pack with the `@pack_type` operator, and expand that into a template. That's a lot of metaprogramming in five lines.

## What is Circle

Circle is a language that extends C++ 17. Aside from some very straight-forward introspection operators, there's very little mechanistically new here for you to learn. If you already know C++, you know Circle. Circle doesn't require you to change how you work. C++ is the de facto systems programming language of the computing industry, and Circle supports it fully, sharp edges and all.

But Circle isn't just C++. What we've done is rotate C++ into the compile-time realm and went where that took us. So while the syntax is C++, and the signature features of C++ remain, like name lookup, overload resolution, implicit conversions, argument deduction, the C preprocessor, and on and on, their usage in Circle has context-dependent consequences.

## Why I wrote Circle

Circle is designed with different goals than recent iterations of C++. I intend to [Keep It Simple, Stupid](https://en.wikipedia.org/wiki/KISS_principle). No feature in Circle requires anything from C++17, or 14, or 11 or even C++ at all (except for the `...[]` and `@sfinae` operators). You don't have to use object-oriented programming, but you can. You don't have to use templates, but you can. You don't have to use the STL, or lambdas, or constraints metaprogramming, but you can. Circle isn't the next version of C++ as much as it is a revisionist history of C. Imagine that a programmer from the future went back to 1990, added reflection, introspection and a compile-time interpreter to C, then let that compiler evolve into the present day.

Ever since template metaprogramming began in the early 2000s, C++ practice has put an ever-increasing cognitive burden on the developer for what I feel has been very little gain in productivity or expressiveness and at a huge cost to code clarity. I think this is mainly caused by an insistence to bolt a functional aparatus onto an imperative language.

Extremely common tasks became puzzles, and hundreds of millions of hours of developer time are spent each year in solving these puzzles. How much of that time and effort can we reclaim and redirect on more creative and productive work?

Consider the trivial example of unrolling a loop. What's the best practice for an unrolled loop using C++14?
```cpp
// The primary template is recursive. Call f on I then go to the next I.
template<int I, int N>
struct unroll_t {
  template<typename F>
  static void go(F f) {
    f(std::integral_constant<int, I>());
    unroll_t<I + 1, N>::go(f);
  }
};

// The partial template stops the recursion by doing nothing.
template<int N>
struct unroll_t<N, N> { 
  template<typename F>
  static void go(F f) { }
};

// Call this function template at each iteration.
template<int I>
void func(int& x) {
  x *= 2;
  printf("After I = %d, x = %d\n", I, x);
}

void foo() {
  int x = 1;

  // f is the body of the loop and invoked once per iteration.
  // The generic lambda function has its own scope, so we need to use & to
  // capture by reference the variable x that we need inside the loop.
  auto f = [&](auto i) {
    // Use decltype(i) to get the type, which we expect to be an
    // integral_constant. Use ::value to access the loop index contained in the
    // structure.
    enum { I = decltype(i)::value };

    // Now do the thing we wanted to do in the body of the loop.
    func<I>(x);
  };

  // Make 5 loop iterations.
  unroll_t<0, 5>::go(f);  
}
```
```
After I = 0, x = 2
After I = 1, x = 4
After I = 2, x = 8
After I = 3, x = 16
After I = 4, x = 32
```
We just want `foo` to enter a loop that calls `func` with the template argument `I` ranging from 0 to 4. We can't write a normal for loop and hope for the optimizer to help us, because the loop iterator must be `constexpr` since we're specializing a function template with it. So how do we effect an unrolled loop without language support for one? Create a class template `unroll_t` that is specialized over the range of the loop. A partial template terminates execution when `I` equals `N`. A static member function calls the provided functor and passes in instance of `integral_constant`, which is a class template that defines an integer constant called `I`. The class template then instantiates itself on `I + 1` and invokes the static member function `go` to begin the next iteration.

How do we use such a thing? Write a generic lambda (C++14 required) that takes an `auto` parameter i. This is actually a function template where `auto` stands in for an invented type parameter. We actually want a non-type parameter `I` (the loop index), but the generic lambda doesn't allow it. That's why we embed the loop index into the class template `integral_constant`. How do we get the loop index back out? Use `decltype` (C++11 required) to infer the type of the function parameter and read out the enum `I`.

Without the generic lambdas that C++14 provides, the process is even more intrusive. You need to put your loop's body into a function template so that it can receive the `integral_constant` argument (although it can now take the loop index as a non-type template parameter directly), but you also need to capture variables like `x` that are in the context of the loop. The only way is to write a class that has reference data members to explicitly capture the loop's context and a non-static `operator()` member function template to receive the loop index argument and execute the loop body.

One could say that generic lambdas lessen the pain of loop unrolling, but let's ask ourselves why we're incorporating a lot of functional programming boilerplate to emulate the most quintessentially-imperative task: stepping through a loop.
```cpp
// Call this function template at each iteration.
template<int I>
void func(int& x) {
  x *= 2;
  printf("After I = %d, x = %d\n", I, x);
}

void foo() {
  int x = 1;
  @meta for(int i = 0; i < 5; ++i)
    func<i>(x);
}
```
```
After I = 0, x = 2
After I = 1, x = 4
After I = 2, x = 8
After I = 3, x = 16
After I = 4, x = 32
```
Circle provides compile-time variants of control flow statements like `if`, `for` and `while`. Executing the loop at compile time has the effect of unrolling the loop. The child statement is translated once for each loop iteration, creating a new function template instantiation at each step.

I tried to bring the imperative constructs that C++ inherited from C into the compile-time realm. Collectively these new features support generic programming in a familiar, proven, imperative style. In the examples below, each statement has _intent_ (meaning it's clear what it does) and _heft_ (meaning it does something). Those are the traits that C brought to software design. I hope Circle is able to carry them over into generic programming, so that we can automate the tedious aspects of engineering and spend more time on the creative parts of design.

Finally, I wrote Circle because I was tired of waiting. Why wait for 2023 or 2026 or 2030 for technical reports to become adopted and pushed out in future C++ compilers? I've wanted to use introspection, reflection and unrestricted compile-time execution since I started programming 22 years ago. The barriers of entry for compiler development are modest: you just need a computer, time, and a high tolerance for pain. When it comes to the tools that I use every day to do my work, I felt it was time to control my own destiny and write the language I've always wanted to use.

## Circle is a triangle

Circle is built on three main technologies:
1. [Integrated interpreter](#integrated-interpreter)  
    C++ extended the evolution of C, and C, a very bare language, is essentially a portable assembler. Few people would try to run C code in an interpreter, because C-flavored dynamic languages like Perl are far more productive.  

    But as software projects grew bigger and bigger, a need for metaprogramming developed. If programming is the automation of tasks, _metaprogramming is the automation of programming_. So what language was chosen to automate C++? It wasn't C++. It was the language of partial template deduction. It wasn't an intentional choice, but the result of people pushing hard on the language to do more and more things it wasn't really equiped to do. Things devolved into riddles.

    Circle fixes this by providing a second backend that does almost everything the code generator does, and quite a bit it doesn't. Now you can run any statement at compile time. In this white paper we load and parse JSON files at compile time. We even host a Lua interpreter at compile time and call Lua function during template instantiation. Because the metaprogramming language is standard C++, we use familiar, existing libraries for all of this.

1. [Same-language reflection](#same-language-reflection)  
    The integrated interpreter only runs code, but we want to create new code programmatically. Do we provide an API for programmatically creating objects, functions and types? Not only is it hard to design an API at a level of granularity that satisfies everyone, it's hard to design one that satisfies everyone. This is exactly why programming languages exist in the first place: we want a convenient syntax for defining a program.

    Circle uses same-language reflection. If you want to declare a class member, just write the C++ for a _member-specifier_. If you want to declare a function, write a function declaration. But how do we do this in an algorithmic or data-driven way? Use compile-time control flow!

    By putting a real statement inside a meta _if-statement_, we guide _if_ that statement is translated into the program. 

    By putting a real statement inside a meta _for-statement_, we guide _how many times_ that statement is translated into the program.

1. [Introspection keywords](#introspection-keywords)  
    C++ programs don't use a runtime environment like Java or C#, so they don't have access to runtime type information. Circle doesn't add a runtime environment. Instead, it exposes to the programmer type information that's already maintained by the compiler.

    Introspect on class types:  
    * `@member_count(type)`
    * `@member_name(type, index)`
    * `@member_ptr(type, index)`
    * `@member_ref(object, index)`
    * `@member_type(type, index)`

    Introspect on enum types:  
    * `@enum_count(type)`
    * `@enum_name(type, index)`
    * `@enum_type(type, index)`
    * `@enum_value(type, index)`

    Generic type operators:  
    * `@type_name(type)`
    * `@type_id(name)`
  
    The arguments for all the operators above must be known at compile time. This sounds limiting--how do we benefit at runtime? Use same-language reflection to write metaprograms that query the introspection operators and bake the type information into functions. That is, the type information you want is moved into the final assembly simply by your using the operators in functions.

The extensions in Circle cover a lot of territory, but they're federated to serve one purpose: help automate the software development process. Features like coroutines, lazy evaluation, garbage collection and modules might be worthwhile additions, but since they don't extend the vision of automation, they didn't make the cut.

## Meta statements

`@meta` does it at compile time. 

[**hello.cxx**](examples/hello/hello.cxx) [(output)](examples/hello/output.txt)  
```cpp
int main(int argc, char** argv) {
  printf("Hello world\n");
  @meta printf("Hello circle\n");
  return 0;
}
```
```
$ circle hello.cxx
Hello circle
./hello
Hello world
```
It executes expression statements. It creates compile-time objects from declaration statements. It does control flow at compile time. It even supports compile-time exception handling.
```cpp
@meta try {
  @meta throw 5;
} catch(int x) {
  @meta printf("I got %d.\n", x);
}
```
```
$ circle foo.cxx
I got 5.
```

Consider this totally not contrived example:
```cpp
@meta time_t t = std::time(nullptr);
@meta tm* now = gmtime(&t);
@meta if(1 == now->tm_wday) {
  @include("monday.h");
}
```
Yes, that does `#include "monday.h"`, but only Mondays. Do you know how to do this with your build system? I don't. But C++ gives us a `time` function and Circle provides compile-time control flow and the `@include` source injection keyword. We can write our own build tools in the language we're most familiar with--the language we've chosen for the program itself.

[**scopes.cxx**](examples/scopes/scopes.cxx) [(output)](examples/scopes/output.txt) 
```cpp
// Meta statements work in global scope.
@meta printf("%s:%d I'm in global scope.\n", __FILE__, __LINE__);

namespace ns {
  // Or in namespace scope.
  @meta printf("%s:%d Hello namespace.\n", __FILE__, __LINE__);

  struct foo_t {
    // Also in class definitions.
    @meta printf("%s:%d In a class definition!\n", __FILE__, __LINE__);

    enum my_enum {
      // Don't forget enums.
      @meta printf("%s:%d I'm in your enum.\n", __FILE__, __LINE__);
    };

    void func() const {
      // And naturally in function/block scope.
      // Ordinary name lookup finds __func__ in this function's 
      // declarative region.
      @meta printf("%s ... And block scope.\n", __func__);
    }
  };
}
```
```
$ circle scopes.cxx
scopes.cxx:6 I'm in global scope.
scopes.cxx:10 Hello namespace.
scopes.cxx:14 In a class definition!
scopes.cxx:18 I'm in your enum.
ns::foo_t::func ... And block scope.
```
All meta statements are admissible in any curly-brace scope. This is because the context for a statement's execution is independent of the runtime situation and is the same in any scope: the meta statement is run when the code is translated by the compiler.

## Integrated interpreter

[**fibonacci.cxx**](examples/fibonacci/fibonacci.cxx) [(output)](examples/fibonacci/output.txt)  
```cpp
// An ordinary function.
std::vector<int> fib(int count) {
  std::vector<int> vec(count);
  vec[0] = 0;
  vec[1] = 1;
  for(int i = 2; i < count; ++i)
    vec[i] = vec[i - 2] + vec[i - 1];
  return vec;
}

// Another ordinary function.
void print_numbers(int count) {
  std::vector<int> vec = fib(count);
  for(int i = 0; i < vec.size(); ++i)
    printf("%3d: %8d\n", i, vec[i]);
}

int main(int argc, char** argv) {
  // Parse Fibonacci number count at runtime.
  int count = atoi(argv[1]);

  // Call ordinary function at runtime.
  print_numbers(count);

  // Call externally-defined functions at compile time.
  @meta printf("How many numbers? (Must be >= 2)\n  ");
  @meta int count2 = 2;
  @meta scanf("%d", &count2);

  // Call ordinary function at compile time.
  @meta print_numbers(count2);
  return 0;
}
```
```
$ circle fibonacci.cxx 
How many numbers? (Must be >= 2)
  8
  0:        0
  1:        1
  2:        1
  3:        2
  4:        3
  5:        5
  6:        8
  7:       13
$ ./fibonacci 11
  0:        0
  1:        1
  2:        1
  3:        2
  4:        3
  5:        5
  6:        8
  7:       13
  8:       21
  9:       34
 10:       55
```
What kind of flexibility do we have in meta statements? Circle translates functions into AST in the usual way. Here, `fib` and `print_numbers` are ordinary functions, parsed once and added to the AST. They have external linkage, so they are also emitted to the executable during code generation. `print_numbers` is called during runtime, using the count provided in `argv[1]`. 

The Fibonacci numbers are also computed during compile time. There is a meta `scanf` call, which halts the compiler process and waits for user input from the terminal. The input is converted to an integer and compilation continues. This time, `print_numbers` is called in a meta statement, causing it to be executed during source translation.

Some alarm bells are going off in the heads of C++ devotees:
* `printf` has side effects. That's not constexpr.
* `scanf` pauses the process and waits for keystrokes. That's super not constexpr.
* `std::vector` uses dynamic memory. That's not constexpr.
* It calls `print_numbers`, which calls `fib`. Neither of those are constexpr.

The solution is an integrated interpreter. The interpreter is a mirror of the LLVM code generator backend.
* Class objects have standard data layout, for interoperability with compiled code.
* Internally-defined functions are executed by interpreting the compiled AST.
* Externally-defined symbols typically rely on a linker. In the interpreter, the function's name is mangled and searched for in the pre-loaded standard binaries: `libc`, `libm`, `libpthread`, `libstd++` and `libc++abi`. Additional libraries may be loaded with the -M compiler switch. When the requested function is found, a foreign-function call is made, and arguments are passed from the interpreter to the native code implementing the function.
* RTTI, exceptions, virtual functions and virtual inheritance are implemented and work exactly as expected.
* Functions may be called from native code via function pointers or virtual functions. A foreign-function closure is created for each function exported out of the interpreter, which provides a callable address. A trampoline function loads the function arguments and executes the function's definition through the interpreter before returning the result back out through the closure.

## Same-language reflection

Circle allows automated code generation in a wonderfully natural way. Rather than providing a complex API that's fine-grained enough to emit C++ code, you just write C++ code. What's the trick that gives you enough leverage to be productive doing this? Meta control flow.

```cpp
void func() {
  // func's scope is A.
  if(true) {
    // The loop's scope is B.

    // j is declared in scope B.
    int j = 0;
  }

  // Error: j was declared in scope B, and we're back in scope A. scope B is
  // inaccessible from here.
  int m = j;
}
```
In C++ and most other languages, objects are pinned into the scope in which they're declared. They may only be accessed from that scope or a child scope. This is a good choice when your intent is to prepare a program for later execution. However, for automating programming itself, there are other options.

Circle implements these meta control statements:
* `@meta { }`
* `@meta try`
* `@meta if`
* `@meta for`
    * `@meta break`
    * `@meta continue`
* `@meta while`
    * `@meta break`
    * `@meta continue`
* `@meta do`
    * `@meta break`
    * `@meta continue`

Each of these statements opens at least one new scope, and that scope is **meta**. Meta declarations are nested into the current declarative region, as usual. Non-meta (i.e. _real_) declarations drop through the curlies and are nested in the innermost enclosing _real_ scope. Meta control flow serves as a scaffold, a framework that allows you to hoist real declarations into place. When the meta program is complete, all the scaffolding comes down, and your program is built.

```cpp
void func() {
  // func's scope is A. It's a real scope.
  @meta if(true) {
    // The if statement's scope is B. But it's a meta scope!

    // j is declared in scope B.
    @meta int j = 0;

    // k is real and declared in the innermost enclosing real scope.
    // That's scope A!
    int k = 1;
  }

  // Error: j was declared in scope B, and we're back in scope A. scope B is
  // inaccessible from here.
  int m = j;

  // k was declared in scope A, and we're in scope A, so we have access to
  // k.
  int n = k;
}
```

## Duff's device

[**duff1.cxx**](examples/duff/duff1.cxx)
```cpp
void duff_copy1(char* dest, const char* source, size_t count) {
  const char* end = source + count;
  while(size_t count = end - source) {
    switch(count % 8) {
      case 0: *dest++ = *source++; // Fall-through to case 7
      case 7: *dest++ = *source++; // Fall-through to case 6...
      case 6: *dest++ = *source++;
      case 5: *dest++ = *source++;
      case 4: *dest++ = *source++;
      case 3: *dest++ = *source++;
      case 2: *dest++ = *source++;
      case 1: *dest++ = *source++;
      break;
    }
  }
}
```
Reproduced above is a simplified version of Duff's device, an infamous memcpy function designed to reduce the amount of branching in the operation. Once we enter the switch, perform an assignment and unconditionally progress to the next case statement. This algorithm cries out for automation. The case statements have indices that run from 8 down to 1, modulo 8. Can we give it the Circle treatment?

[**duff2.cxx**](examples/duff/duff2.cxx)
```cpp
void duff_copy2(char* dest, const char* source, size_t count) {
  const char* end = source + count;
  while(size_t count = end - source) {
    switch(count % 8) {
      @meta for(int i = 8; i > 0; --i)
        case i % 8: *dest++ = *source++;
      break;
    }
  }
}
```
The while and switch statements are real, and produce real scopes. The metafor is compile-time unrolled and opens a meta scope with each iteration. The contents of the metafor are translated _once for each iteration_. This is critically important and fundamentally different from the behavior of conventional control flow. Because the loop is executed at compile time, the value of the loop index `i` is available as a constant for the _case-statement_. The _case-statement_, which is a real statement in a meta scope, drops through the metafor's scope and embeds itself in the switch's scope, which is the innermost real scope.

When the meta scaffolding is broken down, we're left with a program that lowers to the desired IR:
```
  switch i64 %27, label %76 [
    i64 0, label %28
    i64 7, label %34
    i64 6, label %40
    i64 5, label %46
    i64 4, label %52
    i64 3, label %58
    i64 2, label %64
    i64 1, label %70
  ]
```

[**duff3.cxx**](examples/duff/duff3.cxx)
```cpp
template<size_t N>
void duff_copy3(char* dest, const char* source, size_t count) {
  static_assert(N > 0, "argument N must be > 0");

  const char* end = source + count; 
  while(size_t count = end - source) {
    switch(count % N) {
      @meta for(int i = N; i > 0; --i)
        case i % N: *dest++ = *source++;
      break;
    }
  }
}
```
Circle's metaprogramming facilities are as template-compatible as any other C++ feature. In the function template version, the loop count is factored out into a template parameter. When the function template is instantiated, the metafor is executed, and its child statement is translated `N` times, resulting in a more generic and flexible function.

Meta `break` and `continue` statements apply to the innermost meta `for`, `while` or `do` loop. Real statements that complete source translation are preserved in the AST, even if the containing meta control statement exits with an exception or a break/continue operation. 

## Dynamic names

Consider declarations inside metafors:

```cpp
@meta for(int i = 0; i < 3; ++i)
  int x;
```        
* On i = 0, x is declared in the innermost real scope.
* On i = 1, x is declared in the innermost real scope. **Redeclaration!**
* On i = 2, x is declared in the innermost real scope. **Redeclaration!**
* A problem, or an opportunity?

The dynamic name operator `@()` turns strings and integers into identifiers. You can use it in:
* _class-specifiers_ to generate member names.
* _enum-specifiers_ to generate enumerator names.
* Namespace or block scopes to generate functions or objects.

A dynamic name operator with a value-dependent argument is a _dependent-name_. It's turned into an actual identifier during substitution.

```cpp
@("Hello")              // -> Hello
@(std::string("Howdy")) // -> Howdy
@(5)                    // -> _5
@(15 - 25)              // -> _n10
```

## Automating enums

Circle supports any meta statement in any curly-brace scope. This seems like a problem for enums, because the C++ _enum-specifier_ only supports a single comma-separated list of enumerators. Circle extends the _enum-specifier_: you can now break the comma-separated lists up into semicolon-separated lists. This is an impactful change, because we can sneak in meta statements.

[**enums2.cxx**](examples/enums/enums2.cxx)
```cpp
enum class my_enum {
  a, b;                       // Declare a, b

  // Loop from 'c' until 'f'
  @meta for(char x = 'c'; x != 'f'; ++x) {
    // Get a string ready.
    @meta char name[2] = { x, 0 };

    // Declare the enum.
    @(name);
  }

  f, g;                       // Declare f, g
};
```
In this example, enumerators `a` and `b` are declared in the usual way. We terminate their enumerator list with a semicolon so we can introduce a meta for loop. This translates the contained _compound-statement_ from 'c' to 'f'. During each iteration, a string is composed in the automatic array variable `name`. Because `name` is a meta object, it lives in the meta for's scope, and is freed each time the closing brace is met. 

The dynamic name operator `@()` takes this string, which must be known and is known at compile time, and transforms it to the identifier token with the equivalent spelling. Because this is a real declaration (it's not prefixed with meta), it finds the innermost enclosing real scope--the _enum-specifier_--and is interpreted as an _enumerator-list_. On the first go-around, the enumerator `c` is declared. On the second, `d` is declared. On the third iteration, `e` is declared. The loop condition then fails, and execution continues to the real statement that declares enumerators `f` and `g`.

Circle makes it easy to feed enumeration and class definitions from external configurations, which is prime exercise of configuration-oriented programming.

## Introspection keywords

Introspect on class types:
* `@member_count(type)` - Number of non-static data members in class.
* `@member_name(type, index)` - String literal name of i'th non-static data member.
* `@member_ptr(type, index)` - Expression returning pointer-to-data-member of i'th member.
* `@member_ref(object, index)` - Expression returning lvalue to i'th member of object.
* `@member_type(type, index)` - Type of i'th non-static data member.

Introspect on enum types:
* `@enum_count(type)` - Number of enumerators with unique numerical values.
* `@enum_name(type, index)` - String literal name of i'th unique enumerator.
* `@enum_type(type, index)` - Type associated with i'th enumerator. For [typed enums](#typed-enums) only.
* `@enum_value(type, index)` - Expression returning i'th unique enumerator prvalue.

Generic type operators:
* `@type_name(type)` - Convert a type to a string literal.
* `@type_id(name)` - Convert a name to a type.

Circle provides simple introspection keywords for accessing name, type and value information of enumerators and non-static data members. There is no runtime support to this feature; these mechanisms simply expose information that all languages must maintain at compile-time. To access the introspection data in your executable, you'll need to use it in a function. Fortunately Circle's meta control flow makes it simple to automatically visit all the type information for a type and bake it into a function.

## Introspection on enums

[**enums.cxx**](examples/enums/enums.cxx)
```cpp
template<typename type_t>
const char* name_from_enum(type_t x) {
  static_assert(std::is_enum<type_t>::value);
  
  switch(x) {
    @meta for(int i = 0; i < @enum_count(type_t); ++i) {
      case @enum_value(type_t, i):
        return @enum_name(type_t, i);
    }
    default:
      return nullptr;
  }
}
```
In the [Duff's device](#duffs-device) examples we used a meta for to automatically generate _case-statements_. We use this technique again here, but in a function that is critically useful. `name_from_enum` loops over all unique enumerators in the template parameter, compares that introspection-provided enumerator to the argument value, and if they match, returns a string literal with the spelling of the enumerator.

This technique bakes into the executable the type information for each enumeration that `name_from_enum` was instantiated on.
```cpp
template<typename type_t>
std::optional<type_t> enum_from_name(const char* name) {
  static_assert(std::is_enum<type_t>::value);

  @meta for(int i = 0; i < @enum_count(type_t); ++i) {
    if(0 == strcmp(@enum_name(type_t, i), name))
      return @enum_value(type_t, i);
  }
  return { };
}
```
Going the other direction is just as easy. We can't switch over a string, so instead we emit a sequence of `strcmp` operations, each comparing the name of an enumeration with the argument string. If the strings match, the corresponding enumerator is returned. Note that in both of these functions, all arguments to the introspection operators _are known at compile time_.

You may think that these two functions are the alpha and omega of enum introspection. They are not. Circle's metaprogramming capability has revealed enumerations as a deeply versatile data structure. They aren't just identifiers with constant values... They are immutable sets that require no storage, can specialize templates, and have enumerators with spellings that can overload other declarations in a namespace.

## Object serialization

Consider the task of streaming out the contents of a structure as a key/value store, either for machine storage or visual inspection. Without introspection, you'd need to manually write a serialize function for each type. Or maybe you'd use a macro to define both the structure and a corresponding table of string literals. Or maybe you subscribe to a preprocessor like [Protobuf](https://developers.google.com/protocol-buffers/), write your class definitions in a DSL and use a special-purpose compiler to generate bindings and type info.

Circle makes object serialization much easier. Just meta for over the data members and write out the member's name, type and value.

[**serialize.cxx**](examples/serialize/serialize.cxx) [(output)](examples/serialize/output.txt)
```cpp
template<typename type_t>
void stream(std::ostream& os, const type_t& obj) {
  // @member_name won't work on non-class types, so check that here.
  static_assert(std::is_class<type_t>::value, "stream requires class type");

  // Stream the type name followed by the object name.
  os<< @type_name(type_t)<< " {\n";

  // Iterate over each member of type_t.
  @meta for(int i = 0; i < @member_count(type_t); ++i) {
    // Stream the member name and the member value.
    os<< "  "<< 
      @type_name(@member_type(type_t, i))<< " "<< 
      @member_name(type_t, i)<< ": "<<
      <<'\"'<< @member_ref(obj, i)<< "\"\n";
  }
  os<< "}\n";
}
```
This simple version uses the stream operator overloads for iostreams to pretty print objects in a format similar to JSON. No markup of the object is necessary at all; by instantiating this template you bake all of that type's member information into the function.

```cpp
struct struct1_t {
  char c;
  double d;
  const char* s;
};

struct struct2_t {
  std::string string;
  long l;
  bool b;
};

int main(int argc, char** argv) {
  struct1_t one {
    'X',
    3.14159,
    "A C string"
  };
  stream(std::cout, one);
  
  struct2_t two {
    "A C++ string",
    42,
    true
  };
  stream(std::cout, two);

  return 0;
}
```
```
$ circle serialize.cxx
$ ./serialize
struct1_t {
  char c: "X"
  double d: "3.14159"
  const char* s: "A C string"
}
struct2_t {
  std::string string: "A C++ string"
  long l: "42"
  bool b: "1"
}
```

## Better object serialization

The first attempt at object serialization has a defect: it doesn't print out any recursive structure that the object may have. In static languages, objects of the same type technically have the same layout, but data structures like vectors, maps and optional fields can be used to make each object much more dynamic. Can we capture this enhanced structure in a generic serialization routine?

Let's build support for certain kinds of data members:
* **enums** - Write the name of the enumerator rather than the integral value of the enum. Use `name_from_enum` to convert the integral to a string using compile-time introspection.
* **string** - std::string is already support by iostreams, but all class types not specifically handled by `stream` are broken apart and recursively processed, so std::string requires a carve-out.
* **vector** - Follow the JSON convention and print an std::vector as a comma-separated list of items inside square brackets [ ]. The `stream` function is used to recursively print each element.
* **map** - Follow the JSON convention and print std::map as a collection of key-value pairs in curly braces { }. Unlike in JSON, a map key can be a complex type, so the `stream` function is used to recursively print both the key and value.
* **optional** - Follow the JSON convention and print the key name. If the optional structure has a value, call `stream` to recursively print that. If it doesn't have a value, print `null`.

For all other class types, use class introspection to loop over non-static data members, as in the previous example.

[**serialize2.cxx**](examples/serialize/serialize2.cxx) [(output)](examples/serialize/output2.txt)  
```cpp
template<typename type_t>
void stream(std::ostream& os, const type_t& obj, int indent) {

  os<< @type_name(type_t)<< " ";

  if constexpr(std::is_enum<type_t>::value) {
    os<< '\"';
    if(const char* name = name_from_enum<type_t>(obj))
      // Write the enumerator name if the value maps to an enumerator.
      os<< name;
    else
      // Otherwise cast the enum to its underlying type and write that.
      os<< (typename std::underlying_type<type_t>::type)obj;
    os<< '\"';

  } else if constexpr(is_spec_t<std::basic_string, type_t>::value) {
    // Carve out an exception for strings. Put the text of the string
    // in quotes. We could go further and add character escapes back in.
    os<< '\"'<< obj<< '\"';

  } else if constexpr(is_spec_t<std::vector, type_t>::value) {
    // Special treatment for std::vector. Output each element in a comma-
    // separated list in brackets.
    os<< "[";
    bool insert_comma = false;
    for(const auto& x : obj) {
      // Move to the next line and indent.
      if(insert_comma)
        os<< ',';
      os<< "\n"<< std::string(2 * (indent + 1), ' ');
      
      // Stream the element.
      stream(os, x, indent + 1);

      // On the next go-around, insert a comma before the newline.
      insert_comma = true;
    }
    os<< "\n"<< std::string(2 * indent, ' ')<< "]";

  } else if constexpr(is_spec_t<std::map, type_t>::value) {
    // Special treatment for std::map.
    os<< "{";
    bool insert_comma = false;
    for(const auto& x : obj) {
      if(insert_comma)
        os<< ",";
      os<< "\n"<< std::string(2 * (indent + 1), ' ');

      // stream the key.
      stream(os, x.first, indent + 1);

      os<< " : ";

      // stream the value.
      stream(os, x.second, indent + 1);

      // On the next go-around, insert a comma before the newline.
      insert_comma = true;
    }  
    os<< "\n"<< std::string(2 * indent, ' ')<< "}";

  } else if constexpr(is_spec_t<std::optional, type_t>::value) {
    // For an optional member, either stream the value or stream "null".
    if(obj)
      stream(os, *obj, indent);
    else
      os<< "null";

  } else if constexpr(std::is_class<type_t>::value) {
    // For any other class, treat with circle's introspection.
    os<< "{";
    bool insert_comma = false;
    @meta for(size_t i = 0; i < @member_count(type_t); ++i) {
      if(insert_comma) 
        os<< ",";
      os<< "\n"<< std::string(2 * (indent + 1), ' ');

      // Stream the name of the member. The type will be prefixed before the
      // value.
      os<< @member_name(type_t, i)<< " : ";

      // Stream the value of the member.
      stream(os, @member_ref(obj, i), indent + 1);

      // On the next go-around, insert a comma before the newline.
      insert_comma = true;
    }
    os<< "\n"<< std::string(2 * indent, ' ')<< "}";

  } else {
    // For any non-class type, use the iostream overloads.
    os<< '\"'<< obj<< '\"';
  }
}
```
Note how few Circle mechanisms are used in this example. Standard C++ is all that's needed for implementing the special cases. Circle provides the muscle for breaking a class object down into its constituent members, but all other parts are implemented with existing mechanisms.

```cpp
struct vec3_t {
  double x, y, z;
};
typedef std::map<std::string, vec3_t> vec_map_t;

enum class robot_t {
  T800,
  R2D2,
  RutgerHauer,
  Mechagodzilla,
  Bishop,
};

struct struct1_t {
  std::string s;
  std::vector<int> a;
  vec3_t vec;
  robot_t r1, r2;
  vec_map_t axes;
  std::optional<int> opt_1;
  std::optional<vec3_t> opt_2;
  int x;
};

int main() {
  struct1_t obj { };
  obj.s = "struct1_t instance";
  obj.a.push_back(4);
  obj.a.push_back(5);
  obj.a.push_back(6);
  obj.vec = vec3_t { 1, 2, 3 };
  obj.r1 = robot_t::R2D2;
  obj.r2 = robot_t::RutgerHauer;
  obj.axes["x"] = vec3_t { 1, 0, 0 };
  obj.axes["y"] = vec3_t { 0, 1, 0 };
  obj.axes["z"] = vec3_t { 0, 0, 1 };
  obj.opt_1 = 500;
  // Don't set opt_2.
  obj.x = 600;

  stream(std::cout, obj, 0);
  std::cout<< std::endl;
  return 0;
}
```
```
$ circle serialize2.cxx
$ ./serialize2 
struct1_t {
  s : std::string "struct1_t instance",
  a : std::vector<int, std::allocator<int> > [
    int "4",
    int "5",
    int "6"
  ],
  vec : vec3_t {
    x : double "1",
    y : double "2",
    z : double "3"
  },
  r1 : robot_t "R2D2",
  r2 : robot_t "RutgerHauer",
  axes : vec_map_t {
    std::string "x" : vec3_t {
      x : double "1",
      y : double "0",
      z : double "0"
    },
    std::string "y" : vec3_t {
      x : double "0",
      y : double "1",
      z : double "0"
    },
    std::string "z" : vec3_t {
      x : double "0",
      y : double "0",
      z : double "1"
    }
  },
  opt_1 : std::optional<int> int "500",
  opt_2 : std::optional<vec3_t> null,
  x : int "600"
}
```

Automatically converting structs to text (and back again) is useful, but just a starting point. In [Querying Lua](#querying-lua), class member introspection is used to automatically convert between C++ structs and Lua tables. These new features help deal with the interface between all kinds of data representations.

## Metafunctions

* Programs are composed of functions.
* Metaprograms are composed of metafunctions.
* Metafunctions extend the _meta_-ness of objects into functions.

Consider extending the `stream` function from **serialize2.cxx** into an all-purpose printf-like function. Here are the design requirements:

1. Mix text and arguments in a printf-like format specifier string.
1. Rely on type inference to determine the type of an argument rather than encoding it in the format specifier.
1. For class types, use a simplified version of the `stream` function which recursively prints dynamic structure.
1. Be type safe. If the number of argument escape tokens in the format specifier doesn't match the number of arguments in the call, we want a compilation error.
1. Easily extensible. It should be obvious how to amend the format specifier parser to accept new options.

This is a tall order given the mechanisms encountered to this point. #5 is especially difficult--assuming we pass arguments in a function parameter pack, how can the implementation of the print function know what the contents of the format specifier are in order to verify that the number of escapes matches the number of arguments?

We need a way to access the value of the format specifier from inside the print function's definition. Let's call a parameter that has this capability a _meta parameter_. This requires that the argument is known at compile time. It can be a meta object, a constexpr object or a numeric or literal.

But if the function definition access the value of a parameter, and the function can be called with different arguments, this implies that **the function's definition must be recompiled with each call!** This is the distinguishing feature of metafunctions.

* The metafunction's definition is compiled every time you call it!

Here's how you use them:

* Put `@meta` in front of the function.
  * Now it's a metafunction.
* Put `@meta` in front of a parameter.
  * Now it's a meta parameter.
* Put `@meta` after the parameters in a non-static member metafunction.
  * Now the implicit object is a meta parameter.
* Pass meta arguments for each meta argument.
* The function definition has compile-time access to meta parameter values.

Ordinary name lookup, argument deduction and overload resolution rules apply. You're free to overload the same function name with meta and non-meta functions. If overload resolution chooses a metafunction, each argument corresponding to a meta parameter must be constexpr/meta or else you'll be slapped with a compilation error.

[**format.cxx**](examples/format/format.cxx) [(output)](examples/format/output.txt)  
```cpp
template<typename... args_t>
@meta int cirprint(@meta const char* fmt, args_t&&... args);

struct vec3_t {
  double x, y, z;
};
typedef tuple_t<const char*, double, char> my_tuple;

enum class heroes_t {
  Newman,
  Frank,
  Estelle,
  Peterman,
  Bania,
  Whatley,
  BobAndCedric,
};

int main(int argc, char** argv) {
  cirprint("My vector is %.\n", vec3_t { 1, 1.5, 1.8e-12 });
  cirprint("My tuple is %.\n", my_tuple { "a tuple's string", 3.14159, 'q' });
  cirprint("My heroes are %.\n", cir_tuple(heroes_t::Frank, heroes_t::Whatley));
  return 0;
}
```
```
$ circle format.cxx
$ ./format
My vector is { x : 1, y : 1.5, z : 1.8e-12 }.
My tuple is { _0 : a tuple's string, _1 : 3.14159, _2 : q }.
My heroes are { _0 : Frank, _1 : Whatley }.
```

In the example program, we simply use a '%' character to indicate an argument escape. The argument type is inferred from the function parameter pack. The format specifier is a meta parameter.

## Parsing command specifiers

[**format.cxx**](examples/format/format.cxx) [(output)](examples/format/output.txt)  
```cpp
inline const char* scan_to_escape(const char* p) {
  while(*p && '%' != *p)
    ++p;
  return p;
}

template<typename... args_t>
@meta std::string cirformat(@meta const char* fmt, args_t&&... args) {
  const size_t num_args = sizeof...(args);
  @meta size_t len = strlen(fmt);
  @meta size_t cur_arg = 0;

  // The output stream is runtime.
  std::ostringstream oss;

  @meta+ while(*fmt) {
    // The @meta+ keyword makes this entire block recursively @meta, unless
    // opted-out by @emit.
    if('%' == fmt[0] && '%' != fmt[1]) {

      // Check that we aren't doing an out-of-bounds pack index.
      static_assert(cur_arg < num_args, 
        "cirformat replacement is out-of-range");

      // Stream the argument to the output. Use indent=-1 to turn off newlines.
      @emit stream_simple(oss, args...[cur_arg]);

      // Advance to the next argument and format specifier character.
      ++cur_arg;
      ++fmt;

    } else {
      // In an %% sequence, move past the first %.
      if('%' == fmt[0])
        ++fmt;

      // Scan to the next format specifier or the end of the format string.
      const char* end = scan_to_escape(fmt + 1);

      // Stream a string literal of the substring between the escapes.
      @emit oss<< @string(std::string(fmt, end - fmt));

      // Advance the format string to the end of this substring.
      fmt = end;
    }
  }

  static_assert(cur_arg == num_args, 
    "not all cirformat arguments used in format string");

  return oss.str();
}
```

## Contract safety

## Code injection

* `@include(filename)` - Injects a file during translation.
* `@statements(text, name)` - Parse text as a sequence of statements.
* `@expression(text)` - Parse text as a primary expression.
* `@type_id(text)` - Parse text as type-id.

These four operators turn strings into tokens, which are then parsed according to the indicated grammatical production. Specific operators are required for each grammatical production when the argument text is value-dependent. For example, `@expression(x)` where `x` is value-dependent parses like a _primary-expression_ at definition even though the tokens aren't available until substitution.

This is an important distinction between these mechanisms and the preprocessor tricks you may be familiar with: the Circle extensions yield tokens during translation, not preprocessing.

```cpp
@meta time_t t = std::time(nullptr);
@meta tm* now = gmtime(&t);
@meta if(1 == now->tm_wday) {
  #include "monday.h";
}
```
In this scenario, the `monday.h` is included and expanded _unconditionally_ during preprocessing, before `time` tells us its Tuesday. The file is still expanded inside the meta if statement's braces, which may confer some prophylactic security, but the damage is done: Monday has spread its infidel ideas through Tuesday's impressionable translation unit.

```cpp
@meta time_t t = std::time(nullptr);
@meta tm* now = gmtime(&t);
@meta if(1 == now->tm_wday) {
  @include("monday.h");
}
```
Circle's code injection mechanisms are real frontend features. They don't run until source translation, or, in the case of value-dependent arguments, template instantiation. However, they still pass through the preprocessor and can use and modify all of that pass's macros.

## Configuration-oriented programming


## Building functions from JSON

Consider establishing a _domain-specific language_ in the form of a JSON file/Circle code contract. Certain fields in the JSON carry special meaning in the Circle code. In this example, we whip up a DSL for defining mathematical special functions using special JSON fields. The Circle code that executes the DSL loads the configuration file at compile-time, iterates over the fields, and emits ordinary C functions with external linkage that implement requests made inside the JSON.

[**special.json**](examples/special/special.json)  
```json
{
  "sin" : {
    "f"          : "sin(x)",
    "df"         : "cos(x)",
    "series"     : [ 0, 1, 0, -0.1666666, 0, -0.0083333 ]
  },    
  
  "tanh" : {
    "f"          : "tanh(x)",
    "df"         : "1 - sq(tanh(x))",
    "series"     : [ 0, 1, 0, -0.3333333, 0, 0.1333333 ]
  },    
  
  "sigmoid" : {
    "f"          : "1 / (1 + exp(-x))",
    "df"         : "sigmoid(x) * (1 - sigmoid(x))",
    "series"     : [ 0.5, 0.25, 0, -0.02083333, 0, 0.00208333 ]
  }, 
  
  "factorial" : {
    "statements" : "double y = 1; while(x > 1) y *= x--; return y;",
    "integer"    : null,
    "comment"    : "Use a key with null value as a flag"
  },

  "E1" : {
    "f"          : "sq(x) * exp(x) / sq(exp(x) - 1)",
    "series"     : [ 1, 0, -0.08333333, 0, 0.00416666, 0, -0.000165344 ],
    "note"       : "first Einstein special function",
    "comment"    : "Note the differing number of coefficients--JSON is a schemaless formats"
  }
}
```
The top-most entity is an unordered object of key-value pairs. For each pair, the key is the name of the function. This is not necessarily the name of the exported function, but rather a parameter that the Circle code uses when executing the DSL.

The value of each top-level pair is a collection of optional fields. The ones recognized by the Circle code implenting the DSL are:
* `f` - A C++ expression providing the definition of the function. This is useful for functions that are clearly expressed in a single statement. The variable `x` in this expression holds the value of the argument.
* `statements` - A sequence of C++ statements providing the definition of the function. This includes the function's return statement. The variable `x` holds the value of the argument.
* `series` - An array of constants holding the coefficients of a Taylor series to approximate the function. The number of coefficients is not fixed; the Circle code implementing the DSL will evaluate the Taylor series using all the coefficients provided.
* `integer` - A flag indicating that the input argument must be a positive integer. If this flag is asserted in the JSON, the Circle implementation emits code to test if the argument `x` is a positive integer, and if not, produces an error. The decision to emit this test is made at compile time. The test itself is made at runtime, when the value of the argument is available.
* `note` - Since we're generating functions programmatically, it's useful to log the functions as they are defined by the implementation of the DSL. The `note` field is added to the log when its associated function is generated to help the developer make sense of the build process.

The comment and derivative fields don't yet have corresponding implementations in the Circle implementation of the DSL. This is one of the strengths of separating code (C++/Circle) from logic (the JSON). You can version the logic and code independently of one another. Adding fields to the JSON won't break the code. Adding support for new fields in the code won't break the JSON, unless those fields are mandatory.

[**special.cxx**](examples/special/special.cxx)  
```cpp
// Header-only JSON parser.
#include <json.hpp>

// Parse the JSON file and keep it open in j.
using nlohmann::json;
@meta json j;

// Open a json file.
@meta std::ifstream json_file("special.json");

// Parse the file into the json object j.
@meta json_file>> j;
```
```
$ circle special.cxx
Generating code for function 'E1'
  note: first Einstein special function
  Injecting expression 'sq(x) * exp(x) / sq(exp(x) - 1)'
  Injecting Taylor series
Generating code for function 'factorial'
  Injecting statements 'double y = 1; while(x > 1) y *= x--; return y;'
Generating code for function 'sigmoid'
  Injecting expression '1 / (1 + exp(-x))'
  Injecting Taylor series
Generating code for function 'sin'
  Injecting expression 'sin(x)'
  Injecting Taylor series
Generating code for function 'tanh'
  Injecting expression 'tanh(x)'
  Injecting Taylor series

$ nm special | egrep "f_|series_"
00000000004007f0 T f_E1
00000000004008a0 T f_factorial
0000000000400920 T f_sigmoid
00000000004009a0 T f_sin
00000000004009f0 T f_tanh
0000000000400850 T series_E1
0000000000400950 T series_sigmoid
00000000004009b0 T series_sin
0000000000400a00 T series_tanh
```
To open the JSON file at compile time, we use [this popular JSON parser](https://github.com/nlohmann/json). It's header-only, which makes building a breeze. Meta objects for the top-level JSON object and the file stream are created in the translation unit. These objects have the storage duration of the translation unit (that's static storage duration for meta objects); they are destructed in the opposite order in which they're constructed before the compiler exits.

Let's look at the output of the compile process before getting to the rest of the code. As we compile, the status of the five generated functions are logged. These logs are normal `printf` statements coming from inside the translation unit. Each of the five functions from the JSON are logged, but in alphabetical order according to their names. The note for the function `E1` is printed, because `note` is a recognized field in the example's DSL. Functions defined with expressions have those expressions printed. Functions defined with statements have those statements printed. Finally, if the function has a `series` field in the configuration, the log informs when a Taylor series function is generated.

After the executable is built, the command line tool `nm` reveals the exported functions. The functions generated from expressions or statements have "f_" prepended to their JSON names. The Taylor series have "series_" prepended to their JSON names.

Let's break the the code-generation algorithm down into a recipe:
1. `@meta for` over each item in the JSON configuration.
1. `@meta printf` to the log message to the terminal. `@meta if` there's a `note`, print that too.
1. Declare a function with the name f_{name}. 
    * Use string concatenation to join the substrings together. The [dynamic name operator `@()`](#dynamic-names) turns the name string into an identifier token.
    * Mark the function with the `extern "C"` _linkage-specifier_ to turn off function name mangling. After all, we want our output binary to be usable by script engines and other clients that don't implement C++ ABI name mangling.  
1. Generate the function definition.
    * `@meta if` the flag `integer` is found in the item's JSON, emit a runtime test that confirms that the argument variable is a positive integer.
    * `@meta if` there is an `f` field in the JSON, turn the string value of that field into tokens and parse as a _primary-expression_ with the `@expression` extension. Return that result object. Remember to @meta printf the expression for our log.
    * `else @meta if` there is a `statements` field, turn the string value of that field into tokens and parse as a _statements_ production with the `@statements` extension. The _return-statement_ is already part of the `statements` text, so we don't have to return anything. @meta printf the statements for the logic.
    * `else` issue a `static_assert`, complaining that there's no definition for this function. In Circle, `static_assert` takes any meta string, so we can use some text formatting to produce a prettier error.  
1. `@meta if` there is a `series` field, declare a function called series_{name}.
    * `@meta printf` the series to the log.
    * Declare a real variable `xn` that holds the argument `x` raised to the current power.
    * Declare a real variable `y` that serves as an accumulator for the Taylor series.
    * `@meta for` each coefficient in the `series` array.
        * Accumulate `xn * c` into `y`.
        * Advance `xn` to the next power with `xn *= x`.

[**special.cxx**](examples/special/special.cxx)  
```cpp
// Loop over each json item and define the corresponding functions.
@meta for(auto& item : j.items()) {
  @meta json& value = item.value();

  // Extract and print the name.
  @meta std::string name = item.key();
  @meta printf("Generating code for function '%s'\n", name.c_str());

  // Print the note if one exists.
  @meta+ if(value.count("note"))
    printf("  note: %s\n", value["note"].get<std::string>().c_str());

  // Define the function as f_XXX.
  extern "C" double @("f_" + name)(double x) {
    @meta if(value.count("integer")) {
      // If the function has an integer flag, the incoming argument must be
      // a positive integer. Do a runtime check and fail if it isn't.
      if(roundf(x) != x || x < 1.0) {
        printf("[ERROR]: Argument to '%s' must be a positive integer.\n",
          @string(name));
        abort();
      }
    }

    @meta if(value.count("f")) {
      // The "f" field has an expression which we want to evaluate and 
      // return.
      @meta std::string f = value["f"].get<std::string>();
      @meta printf("  Injecting expression '%s'\n", f.c_str());

      // Inject the expression tokens and parse as primary-expression. Returns
      // the expression.
      return @expression(f);

    } else @meta if(value.count("statements")) {
      // The "statements" field has a sequence of statements. We'll execute 
      // these and let it take care of its own return statement.
      @meta std::string s = value["statements"].get<std::string>();
      @meta printf("  Injecting statements '%s'\n", s.c_str());

      @statements(s, "special.json:" + name);

    } else {
      // Circle allows static_assert to work on const char* and std::string
      // objects that are known at compile time.
      static_assert(false,  
        "\'" + name + "\'" + " must have 'f' or 'statements' field");
    }
  }

  // Define the series as series_XXX if a Taylor series exists.
  @meta if(value.count("series")) {
    extern "C" double @("series_" + name)(double x) {
      @meta printf("  Injecting Taylor series\n");

      double xn = 1;
      double y = 0;
      @meta json& terms = value["series"];
      @meta for(double c : terms) {
        @meta if(c)
          y += xn * c;
        xn *= x;
      }

      return y;
    }
  }
}
```
The Circle code that implements our little DSL is not really longer than the recipe when written out in bullet points. Just as there's no API for programmatically declaring data members, there's no API for declaring functions and objects; we just write the thing out with a normal declaration statement:

* `extern "C" double @("f_" + name)(double x)`

The dynamic name operator `@()` turns std::string into an identifier token, and that's the only extension required for this part of code generation. Most languages offering reflection do so at runtime through very elaborate and [fine-grained APIs](https://docs.microsoft.com/en-us/dotnet/api/system.reflection.emit.typebuilder.definemethod?view=netframework-4.7.2). The mantra is _if you know C++, you already know Circle_. With very few language additions for the programmer to learn (although pretty big additions to the compiler itself), it's possible to implement a DSL that traverses a configuration file and generates a shared object based on the fields in that file. 

## Errors in configuration files
```json
{
  "sin" : {
    "f"          : "sin(x)",
    "df"         : "cos(x)",
    "series"     : [ 0, 1, 0, -0.1666666, 0, -.0083333 ],
    "comment"    : "See the broken numeric literal -.0083333."
  }
}
```
Involving external resources like JSON adds a real twist into the compilation process: how do we deal with errors that originate not inside C++/Circle code, but inside the resource itself?

We'll take the [**special.cxx**](examples/special/special.cxx) file and break it subtly. C++ allows floating-point literals in the form -.0083333, but JSON does not: it requires an integer 0 between the _-_ and the _._, like so: -0.0083333. We can't expect an error from the C++ tokenizer, because C++ doesn't tokenize this text--that's done by the JSON parser.

The Circle compiler has no specific response to this kind of error, so what happens when we try to compile **special** using this configuration file?
```
$ circle special.cxx
[ERROR]: Uncaught exception:
[json.exception.parse_error.101] parse error at line 5, column 47: syntax error while parsing value - invalid number; expected digit after '-'; last read: '-.'
```
We get a nicely-formatted error that provides the exact line and column of the problem along with a nice description.

When the JSON library JSON library encounters a syntax error, it throws a parse error exception object. Since we're running the parser at compile time, this becomes a compile-time exception. The translation unit's source code does not catch this exception, although it could with `@meta try/catch`. Instead, the exception unwinds all the way through the translation unit until it is caught by a generic _handler_ in the `main` function of the compiler itself.

The JSON exception object is part of a class hierarchy that inherits [`std::exception`](https://en.cppreference.com/w/cpp/error/exception). Circle's uncaught exception handler is able to make a successful type cast to `exception`, so it's able to call [`what`](https://en.cppreference.com/w/cpp/error/exception/what) to retrieve the error string.

Here's the rub: `what` is a virtual function. Its implementation was defined by the JSON library, _inside the interpreter_. The concrete type that implements this function adds the function pointer to a vtable that exists _inside the interpreter_. The Circle uncaught exception handler is running compiled code, and wants to make a virtual function call into a function implemented in the interpreter. Is Circle able to simplify call `what` and have the function execute by the interpreter?

Yes, it is. Functions that are defined in the translation unit and called from inside the interpreter don't require a foreign function interface: you simply copy the arguments and start interpreting the AST for the function definition. But what happens if we store a function pointer somewhere? We have to store a 64-bit address that is callable like any real, compiled function.

Circle uses libffi to generate a foreign function "closure." This is a small allocation of memory that's marked executable. If you call the address like a function pointer, execution goes to a libffi-provided trampoline, which forwards a handle for the function and pointers to each argument to a user-provided (in this case, interpreter-provided) handler function. The handler can then push the arguments to the interpreter's call stack and execute the function from AST. There's a similar mechanism for returning result objects.

This is a crucial bit of plumbing. You never have to worry about serializing function call arguments between the interpreter and compiled code.
* All calls are intra-process, so we don't have address space concerns.
* The interpreter and compiled codes agree on the convention for class object layout and RTTI, because each implement the C++ ABI.

If we used a JSON parser that came in binary form (i.e. packaged as a shared object) and it threw an exception inside compiled code, that exception would completely unwind the interpreter, then unwind the Circle compiler process itself until it was handled by the uncaught exception handler. In this case, as in the case above, the `what` function on the exception handler is cleanly invoked. This time, however, a direct call is made to the `what` function, because the exception was implemented by compiled code (in a different shared object, sure, but in the same process); there's no call through a trampoline function.

Circle's integrated interpreter features smooth interoperability with compiled code. No markup is ever required to import functions or objects that are provided by compiled code; just `#include` the appropriate header and use it as you normally would. 

## Kernel parameterization

Consider the task of kernel development. A kernel can be any function that runs on streams of data. The field encompasses high-performance computing, data science, structural engineering, fluid mechanics, chemistry, &c &c. These critical functions are often dependent on many identifiable parameters. Drivers of this parameterization involve performance characteristics of the hardware, the size of the input, the type of input, the distribution of the input and accuracy and performance requirements. Examples of kernels are:

* Sorts, reductions and histograms for GPU computing.
* Dense matrix operations for deep learning.
* Finite difference models for climate research.
* Finite element models for engineering.
* Sequence aligners for bioinformatics.
* Radiative transfer integrators for atmospheric science.
* Electrostatic force fields for molecular dynamics simulations.
* Shader programs for realtime and professional graphics.

To reduce cost and development time, it's critical to understand how a change in parameterization affects the output of your kernel. Separating the code from the _logic_ is at the heart of the configuration-oriented programming paradigm.

[**params.hxx**](examples/kernel/params.hxx)
```cpp
enum class kernel_flag_t {
  ldg,
  ftz,
  fast_math
};

struct kernel_key_t {
  int sm;
  std::string type;
};

struct params_t {
  int bytes_per_lane;
  int lanes_per_thread;
  std::vector<kernel_flag_t> flags;
};
```

For this toy problem, consider a kernel that has two template parameters: 
1. sm - A shader model number describing the generation of GPU hardware.
1. type - The type of data being processed, eg double, short or float3_t.

A realistic example would also parameterize the _kind of calculation_ being performed, and attempt to move the brains of the calculation itself into the configuration.

When the kernel's function template is instantiated, it packages its template parameters into the struct `kernel_key_t`, hands it to an interface that abstracts the configuration, and receives a struct `params_t` with the parameters that will be used to guide compilation.

Circle's interpreter will allow us to host the configuration. Circle's introspection will help the abstraction layer automatically convert the `kernel_key_t` type, a C++ structure, to the dynamic type expected by the configuration, and then convert the result object into the `params_t` type. We aim to completely abstract the kernel from the source of the configuration.

## Querying JSON

[**kernel1.cxx**](examples/kernel/kernel1.cxx) [(output)](examples/kernel/output1.txt)
```cpp
// Load a JSON file.
@meta json kernel_json;
@meta std::ifstream json_file("kernel.json");
@meta json_file>> kernel_json;

template<int sm, typename type_t>
void fake_kernel(const type_t* input, type_t* output, size_t count) {
  // Look for a JSON item with the sm and typename keys.
  @meta kernel_key_t key { sm, @type_name(type_t) };
  @meta cirprint("Compiling kernel %:\n", key);

  // At compile-time, find the JSON item for key and read all the members
  // of params_t. If anything unexpected happens, we'll see a message.
  @meta params_t params = find_json_value<params_t>(kernel_json, key);

  // Print the kernel parameters if they were found.
  @meta cirprint("  Params for kernel: %\n\n", params);
  
  // Off to the races--generate your kernel with these parameters.
}
```

```
$ circle kernel1.cxx
Compiling kernel { sm : 52, type : float }:
  Params for kernel: { bytes_per_lane : 8, lanes_per_thread : 5, flags : [ ldg, ftz ] }

Compiling kernel { sm : 52, type : double }:
  Params for kernel: { bytes_per_lane : 16, lanes_per_thread : 5, flags : [ ftz, fast_math ] }

Compiling kernel { sm : 70, type : float }:
  Params for kernel: { bytes_per_lane : 8, lanes_per_thread : 7, flags : [ ldg ] }

Compiling kernel { sm : 70, type : short }:
  Params for kernel: { bytes_per_lane : 16, lanes_per_thread : 10, flags : [ ] }

Compiling kernel { sm : 35, type : int }:
  Params for kernel: { bytes_per_lane : 20, lanes_per_thread : 11, flags : [ ] }
```


## Querying Lua

[**kernel.lua**](examples/kernel/kernel.lua)
```lua
print("lua locked and loaded")

function printf(...)
  io.write(string.format(...))
end

type_sizes = {
  char = 1,
  short = 2,
  int = 4,
  long = 8,
  float = 4,
  double = 8
}

function is_float(type)
  return "float" == type or "double" == type
end

function kernel_params(key)
  -- This function has your kernel's special sauce. It runs each time the
  -- kernel function template is instantiated, and key has fields
  --   int sm
  --   string type
  -- describing the template parameters.

  -- This file is not distributed with the resulting executable. It and the
  -- Lua interpreter are used only at compile-time. However, the luacir.hxx
  -- Circle/Lua bindings work at both compile-time (inside the interpreter)
  -- and runtime.

  printf("  **Lua gets key { %d, %s }\n", key.sm, key.type)
  params = { }
  params.flags = { }
  if is_float(key.type) and key.sm > 52 then
    params.flags[1] = "fast_math"
  end

  if key.type == "short" then
    params.bytes_per_lane = 8
    if key.sm < 50 then
      params.lanes_per_thread = 2
    else
      params.lanes_per_thread = 4
    end

  elseif key.type == "float" then
    params.bytes_per_lane = 16
    if key.sm < 50 then
      params.lanes_per_thread = 4
    else
      params.lanes_per_thread = 8
    end
    params.flags[#params.flags + 1] = "ftz"

  else
    params.bytes_per_lane = 24
    params.lanes_per_thread = 32 // type_sizes[key.type]
    params.flags[#params.flags + 1] = "ldg"

  end
  return params
end
```

[**kernel2.cxx**](examples/kernel/kernel2.cxx) [(output)](examples/kernel/output2.txt)
```cpp
// Load and execute a Lua script.
@meta lua_engine_t kernel_lua;
@meta kernel_lua.file("kernel.lua");

template<int sm, typename type_t>
void fake_kernel(const type_t* input, type_t* output, size_t count) {
  // Look for a JSON item with the sm and typename keys.
  @meta kernel_key_t key { sm, @type_name(type_t) };
  @meta cirprint("Compiling kernel %:\n", key);

  // At compile-time, call the lua function kernel_params and pass our key.
  // If anything unexpected happens, we'll see a message.
  @meta params_t params = kernel_lua.call<params_t>("kernel_params", key);

  // Print the kernel parameters if they were found.
  @meta cirprint("  Params for kernel: %\n\n", params);
  
  // Off to the races--generate your kernel with these parameters.
}
```
The source of the kernel differs only on one line from the `kernel1.cxx`'s kernel: now we call the Lua function "kernel_params" on the `kernel_lua` object. What magic can we expect when the kernel is instantiated?
```
$ circle kernel2.cxx 
../include/luacir.hxx:41:11: error: 
  ip: function 'luaL_newstate' is undefined
```
Drat. Trying to compile the example results in an unusual kind of error: an undefined error, not during linking but during translation. The interpreter, "ip," complains that the function "luaL_newstate" is undefined. This function is declared in the Lua auxiliary header file `lauxlib.h` and called from `lua_engine_t`'s constructor to initialize the Lua interpreter. In order to host Lua, we'll need to _dynamically link_ to its shared object not at runtime but inside Circle at compile time. Use the -M command line argument to order Circle to add `liblua5.3.so` to the list of imported shared objects. All the exported symbols in -M files are scanned so the interpreter can resolve functions and objects when no definition is available in the source. If the symbol is found in a shared object, the interpreter makes an FFI call to the function and execution continues.

[**output2.txt**](examples/kernel/output2.txt)
```
$ circle -M /usr/lib/x86_64-linux-gnu/liblua5.3.so kernel2.cxx
lua locked and loaded
Compiling kernel { sm : 52, type : float }:
  **Lua gets key { 52, float }
  Params for kernel: { bytes_per_lane : 16, lanes_per_thread : 8, flags : [ ftz ] }

Compiling kernel { sm : 52, type : double }:
  **Lua gets key { 52, double }
  Params for kernel: { bytes_per_lane : 24, lanes_per_thread : 4, flags : [ ldg ] }

Compiling kernel { sm : 70, type : float }:
  **Lua gets key { 70, float }
  Params for kernel: { bytes_per_lane : 16, lanes_per_thread : 8, flags : [ fast_math, ftz ] }

Compiling kernel { sm : 70, type : short }:
  **Lua gets key { 70, short }
  Params for kernel: { bytes_per_lane : 8, lanes_per_thread : 4, flags : [ ] }

Compiling kernel { sm : 35, type : int }:
  **Lua gets key { 35, int }
  Params for kernel: { bytes_per_lane : 24, lanes_per_thread : 8, flags : [ ldg ] }
```
We point Circle to the Lua shared object and try again. This time, everything goes through. The Lua function is invoked once for each kernel instantiation. The function is fed a Lua table, which is automatically built using Circle's introspection. It returns a Lua table to the `lua_engine_t` class, which uses introspection to convert the dynamic object to an instance of the `params_t` class. This time, let's go into the details.

[**luacir.hxx**](examples/include/luacir.hxx)
```cpp
template<typename result_t, typename... args_t>
result_t lua_engine_t::call(const char* fname, const args_t&... args) {
  // Push the function to the stack.
  lua_getglobal(state, fname);

  const size_t num_args = sizeof...(args_t);
  @meta for(int i = 0; i < num_args; ++i)
    // Push each argument to the stack.
    push(args...[i]);

  // Call the function.
  lua_call(state, num_args, LUA_MULTRET);

  if constexpr(!std::is_void<result_t>::value) {
    result_t ret { };
    if(auto value = get_value<result_t>(fname))
      ret = std::move(*value);
    return ret;
  }
}
```
`lua_engine_t::call` is called with the function name "kernel_params" and one argument, the 'kernel_key_t' object. A meta for iterates over each function template parameter, calling `push` on each of them. What does `push` do?

[**luacir.hxx**](examples/include/luacir.hxx)
```cpp
template<typename arg_t>
void lua_engine_t::push(const arg_t& arg) {
  if constexpr(std::is_enum<arg_t>::value) {
    lua_pushstring(state, name_from_enum(arg));

  } else if constexpr(@sfinae(arg.c_str())) {
    lua_pushstring(state, arg.c_str());

  } else if constexpr(@sfinae(std::string(arg))) {
    lua_pushstring(state, arg);

  } else if constexpr(std::is_integral<arg_t>::value) {
    lua_pushinteger(state, arg);

  } else if constexpr(std::is_floating_point<arg_t>::value) {
    lua_pushnumber(state, arg);

  } else if constexpr(std::is_array<arg_t>::value) {
    push_array(arg, std::extent<arg_t>::value);

  } else if constexpr(is_spec_t<std::vector, arg_t>::value) {
    push_array(arg.data(), arg.size());

  } else {
    static_assert(std::is_class<arg_t>::value, "expected class type");
    push_object(arg);
  }
}

template<typename arg_t>
void lua_engine_t::push_array(const arg_t* data, size_t count) {
  lua_createtable(state, count, 0);
  for(size_t i = 0; count; ++i) {
    // Push the value.
    push(data[i]);

    // Insert the item at t[i + 1].
    lua_seti(state, -2, i + 1);
  }
}

template<typename arg_t>
void lua_engine_t::push_object(const arg_t& object) {
  lua_createtable(state, 0, @member_count(arg_t));
  @meta for(size_t i = 0; i < @member_count(arg_t); ++i) {
    // Push the data member.
    push(@member_ref(object, i));

    // Insert the item at t[member-name].
    lua_setfield(state, -2, @member_name(arg_t, i));
  }
}
```
As in the [Better object serialization](#better-object-serialization) example, we've got recursive code that switches over different kinds of data members. `push_object` receives a class object and metafors over data member, calling `push` on each of them. 

If the member is an enum, `name_from_enum` converts the enumerator to a string and pushes it to the Lua stack. This allows the C++ side to use enums freely, and the Lua side to treat them as strings (since it doesn't know our enumerators' values). If the member is a string, a string type is pushed; for an integral type, an integer is pushed, and so on. 

If the member is an array or `std::vector`, `push_array` creates yet another Lua table and recursively invokes `push` on each element. In Lua, both arrays and objects/maps are implemented with tables. For arrays, the keys are 1-indexed integers, and for objects, the keys are strings. When `push` is done it returns to `push_object`, which uses Circle introspection to push the _name_ of the data member to the Lua stack and calls `lua_settfield` to set the key/value pair in the table. This builds a Lua table that semantically matches the C++ structure passed to the function template.

When the Lua function returns, the result object is passed to `get_value`, which uses similar techniques to deserialize the value from a Lua table into a `params_t` instance.

The Circle-powered Lua interface is a convenience for the kernel that relies on Lua for parameterization. However, the kernel is free to talk to the Lua interpreter at a lower level, even providing C functions that can be called from Lua. This mechanism is supported by Circle's interpreter--whenever the address of a function is taken in the interpreter, a libfffi closure is generated which returns a callable address and establishes a trampoline mechanism to allow calls from native code to re-enter the interpreter, which executes the function's definition.


## Template metaprogramming

* `...[index]` - Subscript a type, non-type, template or function parameter pack.
* `@sfinae(expr)` - Evaluates true if expression substitutes, false if there's a failure.
* `@mtype(type)` - An eight-byte builtin type that holds a type.
* `@dynamic_type(type)` - Box a type into an @mtype.
* `@static_type(expr)` - Unbox a type from an @mtype.
* `@pack_type(mtype_array, count)` - Type parameter pack from @mtype array.
* `@pack_nontype(object)` - Value parameter pack from class object.

```cpp
template<typename... types_t>
struct tuple_t {
  @meta for(int i = 0; i < sizeof...(types_t); ++i)
    types_t...[i] @(i); 
};
```

## SFINAE

In the context of function template argument deduction, ill-formed types created from argument substitution do not result in an error. Rather, the function template is removed from the candidate set. If no functions are viable, overload resolution fails, and that breaks compilation and results in a diagnostic.

This mechanism is exploited by the `std::enable_if` class template by forming a broken function template default argument, causing unwanted overloads to be filtered out of the candidate set. This is the most common deliberate usage of the Substitution Failure Is Not An Error behavior. It is becoming common to see SFINAE being used as an _expression_ to test if some capability is valid under some choice of template parameters.

```cpp
template<typename _Tp, typename _Up>
  class __is_assignable_helper
  {
    template<typename _Tp1, typename _Up1,
       typename = decltype(declval<_Tp1>() = declval<_Up1>())>
static true_type
__test(int);

    template<typename, typename>
static false_type
__test(...);

  public:
    typedef decltype(__test<_Tp, _Up>(0)) type;
  };
```

This class template is representative of dozens of similar uses in `<type_traits>` which intend to yield a simple true or false value depending on if some operation will compile. In this case, it tests if you can assign an object of type `_Up1` to an object of type `_Tp1`. The decltype expression is an unevaluated context, but it's not one where failures don't result in errors. 

To prevent an error on substitution failure, the programmer has to put the compiler into the context of overload resolution by simulating a function call. That's the point of the typedef at the bottom. The overload in which the assignment expression is valid returns `true_type`, and the other overload returns `false_type`. 

If both candidates are valid, the first candidate is preferred, becasue functions with typed arguments are preferred over functions with C-style ellipsis arguments. In this case, the decltype in the typedef yields `true_type`, and the class template provides useful information. _Because we're in the context of overload resolution_, if the assignment is invalid, the first candidate is removed from the candidate set and no error is issued. The ellipsis function being the only viable function, it is chosen by overload resolution and the typedef sets `type` to `false_type`.

The pattern is so tedious as to discourage its use by language experts and so confusing as to put it out-of-reach for everyone else. Circle provides a simple and efficient operator which exposes SFINAE as an expression:

> `@sfinae(expr)` - Evaluates true if expression substitutes, false if there's a failure.

This operator makes expression SFINAE very easy, and combined with meta control flow or if constexpr, allows for more dynamic code generation.

[**sfinae.cxx**](examples/sfinae/sfinae.cxx) [(output)](examples/sfinae/output.txt)  
```cpp
template<typename type_t>
void go(type_t& obj) {
  // Try to set obj.x = 1.
  if constexpr(@sfinae(obj.x = 1)) {
    printf("Setting %s obj.x = 1.\n", @type_name(type_t));
    obj.x = 1;
  }

  // Try to call obj.y().
  if constexpr(@sfinae(obj.y())) {
    obj.y();
  }

  // Try to use type_t::big_endian as a value.
  if constexpr(@sfinae((bool)type_t::big_endian)) {
    printf("%s is big endian.\n", @type_name(type_t));
  }
}

struct a_t {
  int x;
  enum { 
    // Declare a list of flags to detect here. The values of the enums
    // don't matter, only if they are present or not.
    big_endian,
    zext,
  };
};

struct b_t {
  // obj.x = 1 is invalid, so it should fail during @sfinae.
  void x() { }

  void y() { 
    printf("b_t::y() called.\n");
  }
};

int main() {
  a_t a; go(a);
  b_t b; go(b);
  return 0;
}
```
```
$ circle sfinae.cxx 
$ ./sfinae 
Setting a_t obj.x = 1.
a_t is big endian.
b_t::y() called.
```

## Typed enums

A common template metaprogramming problem involves manipulation of type lists. There is no standard way to represent type lists. Often they are passed in through type parameter packs, manipulated with recursion, and expanded back into arguments to be bound to other template parameter packs.

Circle extends the ordinary enum by allowing you to associate a type with each enumerator. How do we enable this? Use `typename` immediately after the `enum` keyword, right before the `class` or `struct` keywords that indicate a scoped enum:

```cpp
enum typename [class | struct] my_typed_enum [: underlying-type] {
  [identifier = ] type-id, [identifier = ] type-id
};
```

You can make scoped or unscoped typed enums. You can make a typed enum with or without a fixed underlying type. Each enumerator has an _optional_ identifier and a _mandatory_ _type-id_. If you cannot afford an identifier, one will be provided for you. (Circle follows the dynamic-name convention and provides identifiers `_0`, `_1` and so on.)

Why is the name of the enumerator optional? Often you simply want to use a the typed enum as a list of types, and if those types come from a template parameter pack, there's no name to begin with.

In most contexts, a typed enum functions exactly like an ordinary enum. Each enumerator is assigned an implicit integral value, starting at 0 and incrementing with each new declaration. This value maps to the implicit name of the enumerator. The associated type is just that, _associated_. So how do we access this associated type?

* `@enum_type(type, index)` - Type associated with i'th enumerator.

What can we do with this? Define variant classes, for one thing.

```cpp
template<typename... types_t>
struct variant_t {

  struct none_t { };

  enum typename class tag_t {
    none = none_t,
    @meta for(int i = 0; i < sizeof...(types_t); ++i) 
      types_t...[i];
  };

  tag_t tag = tag_t::none;

  union {
    // Declare a variant member for each enumerator.
    @meta for(int i = 0; i < @enum_count(tag_t); ++i)
      @enum_type(tag_t, i) @(i);
  };
  
  // We need an empty constructor to tell the compiler we can be trusted
  // with the union over non-trivial types.
  variant_t() { }

  // Use switches over tag with case statements to destruct, copy construct,
  // move construct, copy assign and move assign variant members.
  ~variant_t();

  // Other members here.
};
```

The foundation for the Circle variant is the typed enum. It holds an enumerator named "none" that has the associated empty class `none_t`. This is what the variant holds by default. We then expand the type parameter pack into the typed enum as unnamed declarations. This is the same pattern as used in the [tuple implementation](#hello-world), but now in an enum context.

The variant holds an instance of the typed enum called `tag`. If the value of tag is `tag_t::none`, the variant is empty. Otherwise the active variant member is the type associated with the enum.

To define the union, we loop over all enumerators in the enum, and for each one, instantiate a variant member of type `enum_type(type_t, i)` with the name `@(i)`.

Because the types are associated with enumerators, and enumerators are already iterable, it becomes trivial to programmatically generate the member functions required to support the variant: pseudo-destructors are called on each variant member to handle destruction; assignment is called on each variant member to handle assignment; move construction is used on each variant member to handle move construction; and so on.

```cpp
struct none_t { };

template<typename tag_t>
struct variant2_t {
  // tag_t must have a none enumerator.
  tag_t tag = tag_t::none;

  union {
    @meta for(int i = 0; i < @enum_count(tag_t); ++i)
      @enum_type(tag_t, i) @(@enum_name(tag_t, i));
  };

  // Member functions go here.
};

// The variant members of the anonymous union are given the same names as
// these enumerators.
enum typename class my_types_t {
  none = none_t,
  us = unsigned short,
  d = double,
  s = std::string,
  m = std::map<std::string, double>,
};

// Instantiate the variant over the typed enum rather than over a parameter 
// pack.
variant2_t<my_types_t> my_variant;
```
We can also define the typed enum by hand rather than automatically through a parameter pack. This allows us to provide convenient names for each enumerator, names which will be transfered to the variant members. Now, in addition to using the standard `get` and accessor on the variant, the user can access variant members by the names they provided in the typed enum.

```cpp

// Instantiate the variant over the typed enum. Now each variant member of the
// anonymous union uses the name of the corresponding enumerator. This is the 
// same as manually writing:

struct variant2_t {
  my_types_t tag = my_types_t::none;

  union {
    none_t none;
    unsigned short us;
    double d;
    std::string s;
    std::map<std::string, double> m;
  };

  // Member functions go here.
};
```
If the above instantiation `variant2_t<my_types_t>` were written by hand, it would look exactly like this. The generative code adheres very closely to the hand-written ideal. We use the enum introspection keywords and metafor over each enumerator, depositing a member-specifier into the anonymous union with each iteration.

## case-typename

For added convenience, Circle introduces a _case-typename_ statement. This bit of porcelain allows the user to specify a _type-id_ in a _case-statement_ rather than specifying an integral _constant-expression_. The _type-id_ is then automatically mapped to the corresponding enumerator in the typed-enum which is used in the predicate of the enclosing _switch-statement_.

* `case typename _type-id_:` - Special case statement compatible with switches over typed enums.

```cpp
enum typename class my_enum_t {
  Int = int,
  Double = double,
  Void = void,
  Char = char,
};

my_enum_t e = my_enum_t::Double;
switch(e) {
  case typename int:
    // case typename replaces the type-id with the corresponding enumerator
    // of the enum in the switch predicate.
    printf("It's an int\n");
    break;

  case typename double:
    printf("It's a double\n");
    break;

  case my_enum_t::Void:
    // We can still use normal case statements in this switch.
    printf("It's a void\n");
    break;

  case typename char:
    printf("It's a char\n");
    break;
}
```
This construct makes writing execution alternatives for each member of a variant class very natural. The underlying switch mechanism remains exactly the same, because the translation from _type-id_ to typed enumerator is performed at compile-time. 

## Variant

Full variant examples coming soon.

## Generic dispatch

A common task is to switch over an entity know at runtime, such as an enum, and call a function corresponding to that variable. C++ introduced virtual functions during the object-oriented programming push to address this common task: the runtime variable becomes, in essence, the virtual pointer that is implicitly created with the object.

But virtual functions cannot be templated, so the dispatch problem remains. One of Circle's goals is to reduce boilerplate code, and dispatches are a big contributor of boilerplate.

What we want is a single function with all of these properties:
1. It takes generic arguments.
1. It returns a generic return type.
1. It works for any arity of composition.
1. It doesn't require any special markup on the types being dispatched.

The third point deserves comment. We could switch over a single enum and call an associated function. That's an arity-1 dispatch. What if we have multiple variables to switch over? Do we need to provide an associated function for each possible combination of them?

```cpp
template<typename type1_t, typename type2_t>
struct uberfunction_t {
  type1_t obj1;
  type2_t obj2;
  double go(double x);
};
```
To keep things organized, let's introduce a class template, which for any arity-N dispatch, has N template parameters. The `go` function draws on functionality from the members instantiated from each of its types. If special behavior is required for a specific type parameter, the uber function gets a partial specialization for that type. If special behavior is required for a specific combination of types, an explicit specialization is provided.

There are a number of challenges in front of us:
1. Create a list of types to substitute into each template parameter.
1. Instantiate the class template and call its function for each combination of types. That is, instantiate the class template for each element of the N-rank outer product of the type lists.
1. Associate an enum or string (or something else we can manipulate at runtime) with each of the types in each of the type lists. It has to be something we use nested switches on, to generate the N-arity dispatch.

What if we choose a typed enum as both the runtime type and the language feature that serves as a type list?
1. An enumeration is a _set_. The set can name the types in each of the type lists.
1. Each enumerator is given a human- and program-readable identifier name. For example, "Int" for type `int` and "ntbs" for the null-terminated byte string type `const char*`.
1. Each enumerator has an associated type which can be extracted at compile-time with the `@enum_type` operator.

[**dispatch.cxx**](examples/dispatch/dispatch.cxx) [(output)](examples/dispatch/output.txt)
```cpp
template<
  size_t I,
  template<typename...> class client_temp, 
  typename... types_t, 
  typename... enums_t, 
  typename... args_t
>
auto dispatch_inner(tuple_t<enums_t...> e, args_t&&... args) {
  if constexpr(I == sizeof...(enums_t)) {
    // Instantiate the client class template.
    return client_temp<types_t...>().go(std::forward<args_t>(args)...);

  } else {
    switch(get<I>(e)) {
      static_assert(std::is_enum<enums_t...[I]>::value);

      // Forward to the next level.
      @meta for(int i = 0; i < @enum_count(enums_t...[I]); ++i) 
        case @enum_value(enums_t...[I], i):
          return dispatch_inner<
            I + 1, 
            client_temp, 
            types_t...,                     // Expand the old types
            @enum_type(enums_t...[I], i)    // Add this as a new type
          >(e, std::forward<args_t>(args)...);
    }
  }
}

template<
  template<typename...> class client_temp,
  typename... enums_t, 
  typename... args_t
>
auto dispatch(tuple_t<enums_t...> e, args_t&&... args) {
  return dispatch_inner<0, client_temp>(e, std::forward<args_t>(args)...);
}
```
The generic dispatch function uses enum introspection in a creative way to greatly simply template metaprogramming. The public-facing function `dispatch` takes an N-arity tuple filled with the enum for each rank we wish to expand, and the arguments to forward as an rvalue reference parameter pack. Each iteration generates a switch statement over the I'th enum, where I runs from 0 to N. The metafor loops over all enumerators in the I'th enum type, extracts the type associated with the i'th enumerator, and appends the type as the I'th type in the `types_t` parameter pack.

Let's consider an example usage:

```cpp
struct circle   { double val() const { return 10; } };
struct square   { double val() const { return 20; } };
struct octagon  { double val() const { return 30; } };

enum typename class shapes_t {
  circle = circle,
  square = square,
  octagon = octagon,
};

struct red      { double val() const { return 1; } };
struct green    { double val() const { return 2; } };
struct yellow   { double val() const { return 3; } };

enum typename class colors_t {
  red = red,
  green = green,
  yellow = yellow,
};

struct solid    { double val() const { return .1; } };
struct hatch    { double val() const { return .2; } };
struct halftone { double val() const { return .3; } };

enum typename class fills_t {
  solid = solid,
  hatch = hatch,
  halftone = halftone,
};

template<typename shape_obj_t, typename color_obj_t, 
  typename fill_obj_t>
struct shape_computer_t {
  shape_obj_t shape;
  color_obj_t color;
  fill_obj_t fill;

  @meta printf("Instantiating { %s, %s, %s }\n", @type_name(shape_obj_t),
    @type_name(color_obj_t), @type_name(fill_obj_t));

  double go(double x) { 
    return (x * shape.val() + color.val()) * fill.val(); 
  }
};

int main(int argc, char** argv) {
  if(5 != argc) {
    printf("Usage: dispatch shape-name color-name fill-name x\n");
    exit(1);
  }

  // Map the enumerator names to enums.
  shapes_t shape = enum_from_name_error<shapes_t>(argv[1]);
  colors_t color = enum_from_name_error<colors_t>(argv[2]);
  fills_t fill = enum_from_name_error<fills_t>(argv[3]);
  double x = atof(argv[4]);

  // Use our tuple to hold the runtime enums.
  tuple_t<shapes_t, colors_t, fills_t> key { shape, color, fill };

  // Provide the enum tuple to select the object to instantiate and the
  // numeric argument.
  double y = dispatch<shape_computer_t>(key, x);

  printf("The dispatch result is %f\n", y);
  return 0;
}
```
The main function parses command line arguments and turns them into enums. The're then packed into a tuple and passed to the generic dispatch, which automatically generates the nested switch statements and instantiates `shape_computer_t` over every one of the 27-elements in the 3x3x3 outer product of types.

[(output)](examples/dispatch/output.txt)  
```
$ circle dispatch.cxx 
Instantiating { circle, red, solid }
Instantiating { circle, red, hatch }
Instantiating { circle, red, halftone }
Instantiating { circle, green, solid }
Instantiating { circle, green, hatch }
Instantiating { circle, green, halftone }
Instantiating { circle, yellow, solid }
Instantiating { circle, yellow, hatch }
Instantiating { circle, yellow, halftone }
Instantiating { square, red, solid }
Instantiating { square, red, hatch }
Instantiating { square, red, halftone }
Instantiating { square, green, solid }
Instantiating { square, green, hatch }
Instantiating { square, green, halftone }
Instantiating { square, yellow, solid }
Instantiating { square, yellow, hatch }
Instantiating { square, yellow, halftone }
Instantiating { octagon, red, solid }
Instantiating { octagon, red, hatch }
Instantiating { octagon, red, halftone }
Instantiating { octagon, green, solid }
Instantiating { octagon, green, hatch }
Instantiating { octagon, green, halftone }
Instantiating { octagon, yellow, solid }
Instantiating { octagon, yellow, hatch }
Instantiating { octagon, yellow, halftone }

$ ./dispatch square green halftone 7
The dispatch result is 42.600000
$ ./dispatch circle red hatch 3
The dispatch result is 6.200000
```
The story gets better. Recall same-language reflection as the mechanism for [dynamic generation of enums](https://github.com/seanbaxter/circle#automating-enums)? Your application can define a stock collection of composable types, and the enums that serve as type sets for generic dispatch can be programmatically generated from configuration files. If you want it done quick and dirty, just include the comma-separated list of types in your configuration and employ `@statements` to inject all those type-naming tokens in one go.