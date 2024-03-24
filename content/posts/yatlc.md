---
title: "Writing a Compiler in Zig"
date: 2023-09-13
tags: ["compilers", "zig", "parsers", "llvm", "programming"]
summary: Short treatise on my experience writing my first compiler in Zig.
---

As a requirement to pass one of my classes, I was tasked to build a compiler.
Before taking on this project, I had only little experience with compilers - 2 unfinished compilers in Go
(finished only lexical analysis and some parsing), a JSON parser in C++,
messed around with ASTs in Go and gained some theoretical (although sometimes obsolete) knowledge
of how to build a compiler at university. Finally, this was my chance to take this all the way
and design and implement a language of my own.

The official assignment was very vague and open-ended, something along the lines of _"Just go,
think of something you'd like to try out and implement it, it does not have to be perfect.
It has to be a compiler (not an interpreter) and the features are up to you to choose."_
So we've decided to implement _imperative, statically typed,
[expression-oriented](https://en.wikipedia.org/wiki/Expression-oriented_programming_language),
syntactically scoped language_ (how exciting, I know...) with handy syntax for multi-dimensional arrays,
chaining of relational operators, ternary operators and all the classical stuff
(for, while and do-while loops, if statements, forward function declarations...).
Pretty similar to C. We've called this language **yatl**,
as in _yet-another-toy-language_. Here's [the repository](https://github.com/jsfpdn/yatlc) and
[a brief language documentation](https://github.com/jsfpdn/yatlc/blob/main/DOCS.md).

[**"If you don't design right from the beginning, every piece of the code you write is a patch"** -
Leslie Lamport](https://twitter.com/changelog/status/1698016182639436038). This should be etched in stone somewhere.

## The Compiler

Since my goal was to learn as many things as possible along the way, I've decided to write the compiler in Zig because
I figured that this was the right opportunity and was eyeing Zig up for some time now. What draw me to
Zig was the handy syntax, (err)defers and error handling, allocators and the build system.

The yatl compiler (yatlc) is completely hand-written, there are no generated units.
Since the design of the language allowed it, we've decided to make the compiler **single-pass**,
not building explicit abstract syntax tree but to emit the (intermediate) code directly.
We chose LLVM IR as the target code.

As usually, the main components of the compiler are the scanner, parser, symbol table, semantic analyzer and the
code generator. Since the compiler is single-pass, the semantic analysis and code generation are baked into the parser.
For example, there is no special semantic analysis unit to analyze the code, this is done directly by the parser.
There is a separate [code generation unit](https://github.com/jsfpdn/yatlc/blob/main/src/codegen.zig),
but it's more of a helper unit used by the parser. Additionally, there is a
[reporter](https://github.com/jsfpdn/yatlc/blob/main/src/reporter.zig)
unit to display "programmer-friendly" errors and give feedback to the programmer.
The compiler is not error resilient. If some error occurs, the compiler simply exits.

### Scanner and Tokens

Here's [the list of all the tokens](https://github.com/jsfpdn/yatlc/blob/main/src/token.zig#L29) in the language.
Tokens can carry additional information with them, for example the location in the source code and the actual symbol,
which can be later used for error reporting. For example, the string `"Hello!"` would be represented by the token

```zig
Token{
    .tokenType = TokenType.C_STRING,
    .bufferLoc = BufferLoc{ .start = 0, end = 7 },
    .sourceLoc = SourceLoc{ .line = 1, .column = 1 },
    .symbol = "\"Hello!\"",
};
```

To make it simple, yatlc reads a single input file and stores it in memory for the entire duration of the compilation,
there's no sliding window mechanism or anything similar implemented. Its cursor into the source file is just
advanced until either a token is fully recognized and can be returned or an error is encounterd.
Token of type `TokenType.ILLEGAL` is returned in that case.
The caller of the scanner (the parser) can either `peek` the current token without advancing and modifying the state,
or it can call `next` and advance one token forward.

[The scanner](https://github.com/jsfpdn/yatlc/blob/main/src/scanner.zig) was the easiest thing to implement.
There are two interesting things to note: First are the [`switchX(...)`](https://github.com/jsfpdn/yatlc/blob/main/src/scanner.zig#L107)
functions to simplify the scanning of short lexems starting with the same character. This was stolen from
the [Go scanner](https://github.com/golang/go/blob/6ecd5f750454665f789e3d557548bb5a65ad5c3a/src/go/scanner/scanner.go#L852) :)
Second is that types are of type `TokenType.IDENT`, the same as identifiers.
It's up to the parser to check whether the variable or function is not named after a type. The scanner
therefore has no knowledge of types.

You can invoke the compiler as `yatlc source.ytl --emittokens -o source` to spit out the tokens to `source.t` file.

### Symbols & Symbol Table

Each function and variable is represented via
[`Symbol`](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/symbols.zig#L9)
and stored in a
[`SymbolTable`](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/symbols.zig#L32).
Here's all the important information needed by the semantic analysis -- types of variables,
their names in the LLVM IR, whether they've been defined or only declared and the symbol's location in the source code.
Let's look at an example: `i32 answer = 42;`. In this case,
the following symbol would be stored in the symbol table:

```zig
Symbol{
    .name = "answer",
    .llvmName = "%answer.3",
    .location = Token{
        .tokenType = TokenType.IDENT,
        .bufferLoc = BufferLoc{ .start = 4, .end = 9 },
        .sourceLoc = SourceLoc{ .line = 1, .column = 5 },
        .symbol = "answer",
    },
    .t = *Type{ .simple = SimpleType.I32 },
    .defined = true,
}
```

Symbol table itself is a an arraylist (`std.Arraylist(std.StringHashMap(Symbol))`, actually) of maps,
but it's treated as a stack. When parser encounters `{`, lexical scope is opened
([`SymbolTable.open()`](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/symbols.zig#L57))
and a new, empty map is pushed to the top of the stack.
When a variable is declared, it is added to the map at the top of the stack. When parser encounters `}`,
lexical scope is closed
([`SymbolTable.close()`](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/symbols.zig#L63))
and a map is popped from the stack. All the symbols in the popped map are deallocated.

When a parser sees an identifier in an expression, the symbol table is consulted, traversing the stack
from top to bottom until the symbol is found, and the corresponding symbol is returned.
If no such symbol is found or the expression expected the symbol to have a different type,
the parser reports an error.

### Types

This brings us to [types](https://github.com/jsfpdn/yatlc/blob/main/src/types.zig). There are 12 atomic types
(signed and unsigned integers of various sizes, bool, float, double and unit, similar to C's void), an array
type and a function type. There are also 2 helper, internal, types for constants and pointers. All these
types are tucked in
[a fat pointer `Type`](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/types.zig#L171).

Functions and arrays are the only recursively defined types (although functions are not first class citizens
in yatl, function therefore cannot return or accept another function as an argument).

```zig
// Type of function `i32 main(i32 argc, [-][-]u8 argv)`
Type{
    .func = Func{
        .retT = *Type{ .simple = SimpleType.I32 },
        // .args is an array list that holds all arguments as symbols
        .args = std.ArrayList(Symbol),
        .namedParams = true,
    },
}
```

```zig
// Type of nested array `[-][-]u8`
Type{
    .array = Array{
        .dimensions = 1,
        .ofType = *Type{
            .array = Array {
                .dimensions = 1,
                .ofType = *Type{ .simple = SimpleType.U8 },
            },
        },
    },
}
```

```zig
// Type of two-dimensional array `[-,-]u8`
Type{
    .array = Array{
        .dimensions = 2,
        .ofType = *Type{ .simple = SimpleType.U8 },
    },
}
```

Since these types are recursively defined, they have to reference each other via pointers and
are therefore allocated on the heap. Each "subtype" (atomic type, array, or function) has its own,
specific, destructor. When calling
[`Type.destroy(...)`](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/types.zig#L184),
the destructor of the "subtype" is called and the pointer to `Type` is destroyed.

### Code Generation

LLVM IR is emitted directly without the use of any 3rd party library.
[`CodeGen`](https://github.com/jsfpdn/yatlc/blob/main/src/codegen.zig#L11)
struct is used for maintaining the IR in memory during the parsing.
The parser creates the string with the SSA instruction and appends it via `CodeGen.emit(...)`
when it sees fit (for example
[here](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L1236),
generating a store instruction when parsing assignments).
There are also some helper functions (e.g. [`llvmFloatOp(TokenType)`](https://github.com/jsfpdn/yatlc/blob/main/src/codegen.zig#L422))
that return the name of LLVM operation depending on the type
of the currently parsed expression and a token.

`CodeGen` also holds a set of
[predefined functions and declarations in the IR](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/codegen.zig#L207)
for I/O. These are the `@my_len`, `@my_print`, `@my_stringFree` and `@my_readLine` functions
(in reality, we just wrote some C code, translated it to LLVM IR and yanked the important parts).
These functions and declarations could be omitted from the generated IR if the programmer does not actually
use them to make the IR smaller, but clang will most probably optimize it away anyway.

Lastly, `CodeGen` is responsible for generating names for functions and variables to be used in the LLVM IR.
This name is then stored within the symbol in the symbol table. For local variables, the LLVM name is in the
form `%<identifier>.<id>` and `@<identifier>.<id>` for global variables. `id` is a unique increasing number.

You can invoke the compiler as `yatlc source.ytl --emitllvmir -o source` to spit out the LLVM IR to `source.ll` file.

### Parser

Parser is the meat of the compiler. It leads the whole compilation process, but from a 10,000-foot view it's fairly simple
-- it asks the scanner for more tokens, validates structure of the input program, adds symbols to the symbol table,
performs type conversions and type checking, emits the IR and reports errors to the programmer if there are any.

The recursive descent parser precisely follows the mutually-recursive non-terminals of the formal grammar. There are 11
non-terminals (of importance) in the grammar, each responsible for recognizing some class of expressions. Usually,
each function peeks the next token and decides whether it should try and parse the construct it's responsible
for or recursivelly call the next function. These are the main parsing functions in order of how they are called
in the recursive call chain:

1. control-flow statements\*
   ([ifs](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L694),
   [fors](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L787),
   [whiles](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L875),
   [do-whiles](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L940),
   [breaks](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L1102),
   [continues](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L1130) and
   [returns](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L1032)),
2. [declarations and assignments](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L1158),
3. [ternary operators `?:` and `??:`](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L1422),
4. [logical expressions](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L1697)
5. [unary `not`](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L2013),
6. [relational operators](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L2047),
7. [binary arithmetical operators `+`, `-`, `>>`, `<<`, `&`, `^` and `|`](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L2201),
8. [unary operators `-` and `!`](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L2285),
9. [binary arithmetical operators `*`, `/` and `%`](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L2398),
10. [array indexing, `--`, `++` and address-of operator `#`](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L2489),
11. [identifiers, (builtin) function calls, parenthesis and constants](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L2781).
    This function for example calls the very first function responsible for the control-flow statements.

> \* There's no such things as statement in yatl -- everything is designed as an expression, control-flow
> statements are expressions of type `unit`. This makes these "special expressions" assignable to variables of type unit,
> but since unit does not hold any value, those variables and assignments are not emitted to LLVM IR.
> This is why the language is [expression-oriented](https://en.wikipedia.org/wiki/Expression-oriented_programming_language).

## The Compiler Experience

There is always plenty of things that can be improved -- error resiliency, better error reporting...

I think implementing a single, small language feature across the whole compiler is the way to go.
I tried to implement whole compiler stages separately but I think that made it a lot harder
to test and see progress. This is doable only for the scanner part in my opinion since there's
not that much programming required and it's fairly straightforward and easily testable.
Also, having at least some subset of the language working early on would help me with my psyché.

Looking back, decomposition and testing are vital (duh!). The resulting parser is a mighty spaghetti beast
that's hard to read if you're not really familiar with it already. I think having explicit AST makes things easier since
you have a way to see what exactly the parser parsed. Without the AST, you have to rely a bit more on
the generated code which is really tedious.

## The Zig Experience

I have not done myself any favors by choosing Zig for the compiler. It was great experience and I think
I've learnt a ton, but implementing it in Zig made it harder than it could've been.

- Allocators are amazing. Even though I've spent non-trivial time chasing double-frees and memory leaks
  (and some are most definitely still there), it was still pretty good experience. If I were to write
  this in, say, C, I would not be bothered with memory leaks as much :wink:.
- I also learned to not bother with borrowing/sharing memory. The better option, in my opinion,
  is to just allocate new structure and return (pass) that when somethings must leave a function (or be passed to it).
- [Character ranges in switch statements are neat!](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/scanner.zig#L121)
- Errdefer, try and [switching on caught errors](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/parser.zig#L2435)
  are great quality-of-life-improvements. When error handling is nicely "tucked" away, code can be very concise and readable.
- I missed "proper" test suite like in Go --
  ["solved" with tables, and erredefer to log the name of the failed test](https://github.com/jsfpdn/yatlc/blob/7833e74d663ea1363613a5f55a7620299e6c3f5e/src/types_test.zig#L7).

There are a lot of things that are not implemented or designed well that should be refactored
and redesigned. I'm looking forward to implementing a different compiler in the (hopefully near)
future and doing this little bit more right.

## Future

What I'd like to try to implement in some future compiler:

* Language features: structs, higher-order functions, proper constant-expression evaluation at compile time,
  modules & imports, providing some standard library.
* Compiler features & tooling:
    * WASM as target code. I'd like to get faimiliar with WASM and the whole WASM ecosystem 
      and the possibility of compiling some language for the web seems cool.
    * Parse explicitly into AST to be able to write linters and formatters. 
    * DAP support for debugging.
    * Incremental parsing. This is probably the most ambitious idea of all of these, but the whole concept
      of incremental parsing and doing local updates in the AST seems awesome.
    * [SIMD during tokenization](https://www.openmymind.net/SIMD-With-Zig/). I don't think this is the right kind of
      optimization to start with since lexer is the most efficient unit of the compiler.
      I think much better optimization opportunities are in the design of the language itself to parse it easily,
      code generation, helper/internal data structures etc.
      [I still find it very interesting to lex via SIMD tho](https://stackoverflow.com/questions/64975030/can-i-use-simd-for-speeding-up-string-manipulation).

## In Closing

I always thought that writing a compiler will make you a better programmer. Now that I've written one,
do I feel that I'm a better programmer? I do not know. I do tend to think a lot more about compiler internals
when programming, but usually not because I'm trying to make my code more efficient,
but because I'm curious how particular feature of the language is implemented and how it works under the hood.
Other than that, it has greatly expanded my horizons how things work and how some problems can be attacked.

Even though I expected this project to be demanding, I was surprised how much time I spent on it.
Certainly, writing a compiler is complex process that won't be finished in a week (or a month or two
if you've got a full-time job, school to tend to and a life)
Also, learning a new (low-level) language (with manual memory management!) along the way did not help :)
With hindsight, I'm happy we've kept the language small, implemented only a handful of features
and were not too ambitious. What's funny is that before starting this project, I was not able to find
many resources talking about single-pass compilers. Just few weeks before writing this post,
[vgel](https://vgel.me) wrote
[this cool post about writing a single-pass compiler for subset of C in 500 lines of Python](https://vgel.me/posts/c500/),
supporting similar features as yatl has. yatlc has 5563 lines :)

Would I write the compiler in Zig again? Since I've done it once already and somewhat learned what Zig is about
along the way (as was my secondary goal of this exercise), no, I would not. If I had all the time in the world
and the motivation to squeeze every single drop of perfomence out of it, then maybe. Otherwise,
I would write it either in Go (since I'm the most familiar with it), Rust (to learn the language)
or some functional language (either Haskell or OCaml) to see what compiler development is like in purely
functional environment (and to get hands-on experience with monadic parser combinators).

Overall I'm very happy that I've managed to finish the compiler and also rediscovered my fondness
towards higher-level languages.
