---
title: "Compile-Time Function Evaluation"
date: 2023-03-04T10:19:00+02:00
tags: ["compilers", "c++", "zig", "ast"]
summary: "Brief overview of two possible implementations of compile-time function evaluation: recursive walks over the AST and byte-code interpretation."
---

In this post I try to briefly outline possible approaches when implementing compile-time
function evaluation in compilers. This is a blog-post version of a short presentation I gave
in a compiler class.

## What Is Compile-Time Function Evaluation?

*Compile-time function evaluation* (or CTFE, for short) is a mechanism for evaluation functions and
expressions at compile-time (duh). This means that instead of emitting instructions and executing them
at runtime, the compiler evaluates expressions during the compilation process and replaces the expressions
with the results. It can be seen as a generalization of the 2 optimizations called <cite>constant propagation and
constant folding[^1]</cite>. It is present in some languages, e.g. Zig (`comptime`), C++ (`constexpr`),
Rust (`const`), etc. <cite>Note that Go has supports only simple constant expressions evaluation, not function evaluation[^2]</cite>.

## What Is CTFE Good For?

CTFE has two advantages:

1. *Performance*: by evaluating expressions during compilation, less instructions are to be executed during runtime.
2. *Robustness*: by evaluating expressions during compilation, certain conditions and assertions can be checked at
   compile time. If such conditions do not hold, the program does not compile.

Point 1 is straightforward, what's interesting to me is the second point. "Side-effect" of using CTFE means
that the programmer can write more reliable code. Certain bugs can be caught during compilation, not during
the runtime, which is neat. Additionally, compiler must check for undefined behavior during CTFE. If undefined
behavior is met (e.g. integer overflow or memory leak), the compiler must emit an error and exit (this is the case at
least for C++ and Zig).

## Limitations of CTFE

If there were no limitations to CTFE, what would stop us from evaluating the entire program during compilation
and just emit instructions returning the desired result? This would be the ideal case. Well, it's the runtime
dependencies that limit us. We can quickly see that not all things can be evaluated at compile-time. Worse
we can evaluate only a very small subset of all possible expressions during the compilation. Such limitations
depend on the specification of the language. If you wish to see all the limitations for the C++ `constexpr`,
see <cite>the language doc specifying what's a *constant expressions* in C++[^3]</cite>. Generally, an expression
is constant if it depends only on other constant expressions and constant variables.

In the compiler architecture, CTFE is a strictly front-end (or a middle-end) thing. This means that CTFE happens
*before* linking. In C++ world, this means that we cannot evaluate `constexpr` functions that will be linked
into our program, since we do not know the function body during the compilation phase! This makes sense,
     but consider the C++ following code:

```c++
constexpr int evaluate_me(int number);

int main() {
    constexpr int result = evaluate_me(10);
}

constexpr int evaluate_me(int number) {
    return number + 32;
}
```

This does not compile! Even though the function body is available in the same source file where the invocation
happens, the CTFE cannot evaluate `evaluate_me(10)` since the function body would be parsed only after
the invocation has been encoutered and "CTFE"-ed.

## 2 Possible Implementation Approaches

There are two possible (and kinda neat and simple) approaches when implementing CTFE: *recursive AST evaluation*
and *translation of AST to some IR and subsequent interpretation*.

### Recursivelly Walking & Evaluating the AST

This is the straightforward solution. During the build of the AST, check whether a node is marked as a
constant expression/function. If the node is marked as constant, then recursivelly, and in a depth-first search manner,
evaluate the abstract syntax subtree rooted at the current node. If some of the descendant nodes cannot be
evaluated, emit a compile-time error and fail. Otherwise, return the value up the tree until reaching the
node where we began.

<cite>This is the current implementation in Clang[^4]</cite>. It has two neat properties: it is easy to understand and implement.
If you check the Clang source responsible for CTFE, you'll be greeted with a file containing 16k LOC
(`AST/ExprConstant.cpp`). This is daunting, but the entrypoint function <cite>`Expr::Evaluate`[^5]</cite> is really simple,
here's the simplified outline of the function:

```cpp
static bool Evaluate(APValue &Result, EvalInfo &Info, const Expr *E) {
    QualType T = E->getType();
    if (E->isGLValue() || T->isFunctionType()) {
        LValue LV;
        if (!EvaluateLValue(E, LV, Info))
            return false;
        LV.moveInto(Result);
    } else if ( ... ) {
        // Same pattern goes on for other types of expressions.
    }
    return true;
}
```

The function does the following: get the type of the expression, recursivelly call the associated
function which computes the value of the expression, check whether it was computed successfully and store it.
That's it.

### <cite>Interpreting Byte-Code[^6]</cite>

Recursivelly walking and evaluating AST can be REALLY expensive, especially when evaluating the same constant
expression multiple times (invoking single `constexpr` function multiple times) or evaluating cycles with lots
of iterations.

Different implementation of CTFE can mend this. First, when encoutering a `constexpr` function,
translate the function body to some intermediate representation or byte-code. Subsequently, the
byte-code is interpreted by some interpreter, when such function is invoked and a result is returned.

Clang offers an experimental *constant interpreter* (hidden behind the `-fexperimental-new-constant-interpreter` flag),
which does exactly this. The interpreter is a simple stack machine and the byte-code is human readable.
Here's a simple example demonstrating the power of the interpreter. Suppose the following line of code in C++
(and let's imagine it's a body of a cycle summing array into an accumulator variable):

```cpp
acc = acc + array[i];
```

This is the corresponding byte-code:

```text
GetLocalSint32 acc
GetParamPtr array
GetLocalSint32 i
AddOffsetSint32
LoadSint32
AddSint32
SetLocalSint32 acc
```

And these are the steps of the interpreter:

```text
// Stack              , selected instruction and operands
1. []                 , GetLocalSint32 acc
2. [acc]              , GetParamPtr array
3. [acc, *array]      , GetLocalSint32 i
4. [acc, *array, i]   , AddOffsetSint32
5. [acc, (*array + i)], LoadSint32
6. [acc, value]       , AddSint32
7. [(acc + value)]    , SetLocalSint32 acc
```

Note the instruction at step 4, `AddOffsetSint32` - it takes number (an offset) and a pointer and moves the pointer
as specified by the offset, resulting in a value `(*array + i)` at the top of the stack at step 5.
Also note the `LoadSint32` instruction at step 5, taking the pointer at the top of the stack and loading
a value pointed to by the pointer. Effectively, steps 4 and 5 get the i-th element in the `array`.

Few important notes:

1. Translating AST to byte-code and interpreting is MUCH more efficient,
2. multiple invocations of a single function result in a single compilation,
3. byte-code is optimizable!

> Note: <cite>The work on constant interpreter was picked up again[^7]</cite>

### CTFE and Undefined Behavior

I've talked about CTFE and UB: if UB occurs, compiler must emit an error and exit. This is very helpful for the
programmer, since they can catch more errors at compile time, but this has a pretty significant impact on the
CTFE mechanism implementation; it now must detect a wide variety of errors which are not always easy to detect.
For example, CTFE in C++ must implement a subset of a memory sanitizer -- checking invalid memory access etc.

### Zig

Zig comes with a powerful CTFE support. It implements CTFE via an interpreter, interpreting untyped ZIR
(Zig internal interpretation). What's interesting is that parametric polymorphism is implemented via the same
mechanism - all type variables in a function declaration are marked as `comptime` and values passed in
at the call site must be know at compile time.

2 things stand out to me when it comes to Zig CTFE. First, functions do not have to be explicitely marked
with `comptime` in order to be invoked at compile-time! Single function can be invoked during runtime
AND compile-time (if the function body can be evaluated at compile-time). We can mark the function call with `comptime`:

```zig
const std = @import("std");

pub fn main() void {
    const comptime_result = comptime add(32, 10);
    const runtime_result = add(32, 10);

    std.log.info("comptime result: {}", .{.comptime_result});
    std.log.info("runtime result: {}", .{.runtime_result});
}

fn add(fst: u8, snd: u8) u8 {
    return fst + snd;
}
```

Second, whole blocks can be marked as `comptime`. Suppose we wish to check whether all characters in a string
are upper case at compile-time:

```zig
pub fn main() void {
    // Do some computation ...
    comptime {
        var i = 0;
        while (i < text.len) : (i += 1) {
            if (text[i] >= 'a' and text[i] <= 'z') {
                @compileError("text must be uppercase!");
            }
        }
    }

    // Continue with the computation.
}
```

Zig offers a builtin <cite>`@embedFile`[^8]</cite> function, taking a `comptime` path and returning a string. This builtin
combined with user-defined `comptime` functions working with the embedded file can be pretty powerful.


## Drawbacks of CTFE

CTFE is not all glitz-and-glamour, it also comes with its own drawbacks. For one, by having to evaluate functions
at compile time, the compile times get longer.

Another drawback, invisible to the programmer, is the impact on the actual compiler infrastructure. Implementing
CTFE, especially via constant interpreter, bloats the compiler, making it harder to maintain. But what does not?!

[^1]: [Constant folding & propagation on Wikipedia](https://en.wikipedia.org/wiki/Constant_folding)
[^2]: [Go language specification](https://go.dev/ref/spec#Constant_expressions)
[^3]: [C++ constant expressions language design doc](https://en.cppreference.com/w/cpp/language/constant_expression).
      Also see the [`constexpr` specifier doc](https://en.cppreference.com/w/cpp/language/constexpr).
[^4]: [High-level documentation of the constant interpreter in Clang](https://clang.llvm.org/docs/ConstantInterpreter.html)
[^5]: [`Expr::Evaluate` function in the `AST/ExprConstant.cpp` file](https://clang.llvm.org/doxygen/ExprConstant_8cpp_source.html#l14979)
[^6]: [AMAZING talk by Nandor Licker, initial author of the C++ constant interpreter](https://www.youtube.com/watch?v=LgrgYD4aibg).
      This talk is a must-see, Nandor gives awesome introduction to the work he has done.
[^7]: [Nicely written blogpost on the C++ constant interpreter](https://www.redhat.com/en/blog/new-constant-expression-interpreter-clang)
      by Timm Baeder who is continuing the development in the interpreter.
[^8]: [Zig's `@embedFile` builtin function documentation](https://ziglang.org/documentation/0.10.1/#embedFile)
