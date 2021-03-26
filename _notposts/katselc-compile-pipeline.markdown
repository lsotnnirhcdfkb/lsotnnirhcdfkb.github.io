---
title: The katselc Compile Pipeline
date: 2021-03-26 14:58:25 -0400
---

*At the time of writing most of this article, I mention that I haven't even finished implementing the parser. That is not true as of the time of publishing this article; I am now working on implementing Stage 3*

I recently made the decision to [rewrite katselc from scratch](https://github.com/lsotnnirhcdfkb/katsel/commit/ebada4c76581492a0958df5187b18d214a60a5f5) in Haskell instead of C++.
After putting a few functional sum datatypes in there (including the monstrous [Token datatype](https://github.com/lsotnnirhcdfkb/katsel/blob/92a96e3203e05e74fe30677c00e9ca13ebf42c5d/include/lex/token.h) and its [rewrite](https://github.com/lsotnnirhcdfkb/katsel/blob/oldcpp/src/lex/token.cpp)), and the fact that these datatypes made compile times extremely high, I felt it would just be cleaner to rewrite it instead in a language that properly supports them first-class.
I haven't added much yet (as of writing this I've only finished the lexer and some other things), so I plan to document the process of creating the compiler a lot more than I did before.
From here, you should expect more posts documenting the compiler more in detail, or maybe I might give up on this plan and just not document it at all.

I'm planning for the compiler to have a 4-stage pipeline, starting with lexing the source code.

---
# Stage 1: Lexing

The first stage transforms a sequence of characters into a stream of *tokens*.
A token is a small unit of meaning in a language.
katselc has 75 token types.
Some of them are symbols like '`OParen`' and '`BangEqual`', while others have more meaning like '`Identifier`' and '`IntLit`'.
Most of the rest of them of them are keywords, like '`While`' and '`If`'.

The lexer also handles the indentation tokens.
At the first character of each line (except for the first line) the indentation level of that line is counted.
That indentation level is compared to the one on the top of the indentation stack.

If it is:
- greater than the previous level, then emit an '`Indent`' token and push a new indentation frame onto the indent stack
- less than the previous level, then emit a '`Newline`' and as many '`Dedent`' tokens as there are indentation frames to pop
- equal to the previous level, then emit a '`Newline`' token

<sup>you might notice that this is reminiscent of the way Haskell's layout works</sup>

And then at the end, all the indentation frames are popped off the stack and enough dedent tokens are emitted.

If the top of the stack is an indentation-insensitive frame, then all whitespace is ignored until the '}' that closes that frame is reached

So, the source:

```
fun foo(n: uint32)
    n + 2
```

would be lexed into the sequence of tokens

```
[ Fun: 'fun' ] [ Identifier: 'foo' ] [ OParen: '(' ] [ Identifier: 'n' ] [ Colon: ':' ] [ Identifier: 'uint32' ] [ CParen: ')' ]
[ Indent: '' ] [ Identifier: 'n' ] [ Plus: '+' ] [ IntLit: '2' ] [ Newline: '\n' ]
[ Dedent: '' ]
[ EOF: '' ]
```

---
# Stage 2: Parsing

The parser transforms a flat list of tokens into an abstract syntax tree.
An abstract syntax tree represents the structure of the tokens, and how they interact with each other.

katselc uses a [*parsing expression grammar* (PEG) parser](https://en.wikipedia.org/wiki/Parsing_expression_grammar).
PEG parsers are very interesting, because they were designed to be directly convertible to backtracking recursive descent parsers, and also can be easily turned into [packrat parsers](https://pdos.csail.mit.edu/~baford/packrat/thesis/).
PEGs are very easy to design in Haskell, because all the basic PEG operations can be transformed into [parser combinator functions](https://en.wikipedia.org/wiki/Parser_combinator) (which are functions that take parsers as inputs and combine them into a new parser).

<sup>I might make another blog post about PEGs in the future</sup>

The AST is a tree structure that represents what the code means.
For example, the above sequence of tokens would be parsed into the following AST:

```
FunctionDecl
|-- name: 'foo'
|-- params:
|   `-- Param
|       |-- name: 'n'
|       `-- type: 'uint32'
`-- body: BlockExpr
          `-- stmts:
               `-- VarExpr 'n'
```

---
# Stage 3: IR

The IR is a different language that has a small subset of katsel's features.
It is built on a control-flow graph, which is a network of *basic blocks* that are all connected by branch instructions.
The idea behind a basic block is that all the instructions in the basic block are run from beginning to end, with no jumps.

Many primitive operations are <sup>(going to be)</sup> represented by *intrinsic functions*, functions that are builtin to the language.
For example, addition of two '`uint32`'s is <sup>(going to be)</sup> represented by a call to the intrinsic function '`__intrinsic_add_uint32`' (the function's name might not be exactly this however)

During transformation from AST to parsing, name resolution happens, and the type of all the values is determined.
Type-checking is a separate pass that happens on the fully constructed IR, unlike in the old C++ type-checker, which ran during the creation of each IR instruction.

The above AST would be transformed into the following IR:

<sup>(The first register is the return value register, and next few are for the parameters, so in this case '`#0`' is for the return place and '`#1`' is for the first parameter. Also, '`=>:`' indicates a branch instruction or a basic block terminating instruction)</sup>
```
fun foo(n: uint32): uint32 {
    a: uint32
    b: uint32

    entry(0) {
        (%0: uint32) = call __intrinsic_add_uint32 b 2;
        (%1: void) = store a %0;
        =>: gotobr(exir(1));
    }

    exit(1) {
        =>: return;
    }
}
```

Note that the textual form of the IR is entirely for human readers; the compiler only ever prints the textual form of the IR, but it does not parse it.
The IR is stored instead as a collection of objects (that can get rather complicated) in memory.

Some optimizations will happen here, but I haven't implemented any <sup>well I also at the time of writing this haven't even fully implemented the parser</sup>, and I can't think of any optimizations to apply to this.
But eventually, some optimizations will happen.

---
# Stage 4: Codegen

At this stage, the IR is transformed into the backend code. As of right now, I'm planning to have a C backend, so that IR would be transformed into the following C code:

```c
uint32_t _KF2Ns3fooP1T1Bs6uint32(uint32_t n) {
    uint32_t hashreg_0;
    uint32_t hashreg_1;

entry_0:
    uint32_t percentreg_0 = _KF3INTrs1yNs10add_uint32P2T1Bs6uint32T1Bs6uint32(hashreg_1, (uint32_t) 2);
    hashreg_0 = percentreg_0;
    notvoid_t percentreg_1 = notvoid;
    goto exit_1;

exit_1:
    return hashreg_0;
}
```

<sup>I can't wait to see how the C code is eventually going to end up looking like for really complex programs, especially with the name mangling! Also, those were a manually mangled names, so they might be a bit wrong, and I also haven't fully decided on a mangling scheme, so this is an approximate result. I might write another post about the name mangling, just to document it</sup>

And from there, that code (which hopefully isn't too too long) can be passed into a C compiler and that can be compiled to machine code from there.
I might make an LLVM backend though, if compilation times turn out to be unsatisfactory.

---
# Stage 5: Runtime (?)

And once there is an executable to run, then it can just be run.
The process of running a katsel program is complete.
