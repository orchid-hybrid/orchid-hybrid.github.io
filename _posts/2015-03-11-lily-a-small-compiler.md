---
layout: post
title: Lily - a small compiler
---

# Introduction

I worked with my friends to create a small compiler just for learning, it's called lily after a terrible funny anime ripoff of alien: https://github.com/tca/lily

Here is the definition of the language:

```
<program> ::= <definition> ...

<definition> ::= (define (<name> <var> ...) <statement> ...)

<statement> ::= (begin <statement> ...)
              | (display "string")
              | (display <expression>)
              | (if <expression> <statement> <statement>)
              | (set! <var> <expression>)
              | (return <expression>)

<expresson> ::= <var>
              | (<op> <expresson> <expresson>)
              | (<name> <expression> ...) ;; procedure call

<op> ::= + | - | * | % | / | =
```

and here's an example program:

```
(define (main)
    (return (fizzbuzz 0)))

(define (fizz) (display "fizz"))

(define (buzz) (display "buzz"))

(define (fizzbuzz x)
    (if (= x 100)
        (return 0)
        (begin
            (if (= (% x 3) 0)
                (call (fizz))
                (if (= (% x 5) 0)
                    (call (buzz))
                    (if (= (% x 15) 0)
                        (begin (call (fizz)) (call (buzz)))
                        (display x))))
            (newline)
            (return (fizzbuzz (+ x 1))))))
```

lily compiles this to x86 64-bit assembly code which you can assemble with nasm and run!

The language is very simple. You can't heap allocate anything, the only variables you have are those stack allocated when calling a procedure. The only data type is a 64 bit integer.

# Purpose

The primary purpose of writing this compiler was to learn about managing stack frames.

It's quite tricky to do stack frames, you have to design your stack frames and invent a calling convention - the first design we came up with was pretty bad.. but as soon as we noticed that we improved it. I also read gcc output a bit to learn from how they do it. Here are the notes on that: https://github.com/tca/lily/edit/master/NOTES2.md

# Implementation

To write this compiler the first thing I did was write a parser or recognizer for the language definition. I'm programming in scheme I can use s-expressions to save a lot of work, but I still wanted to ensure that the s-expressions fitted the language definition.

I used my own pattern matcher for that (the scheme language doesn't have a pattern matcher, so I added that to it with a macro) and in the processes of writing lily I developed my matcher further. This was a really good experience.

* https://github.com/tca/lily/blob/master/lily/recognize.scm

Once you know the input program is (at least mostly) well formed you can actually compile it. To write the compiler I made a copy of the recognizer and then changed it to emit "sequences" of what I call s-instructions. For example:

```
   (mov rax 0)
   (mov rbx 7)
   (add rax rbx)
   (push rax)
```

There is a script called asm.scm which translates this into proper assembly syntax which nasm can compile. It's much much easier to produce s-expressions than directly producing assembly (or in something like haskell or ocaml, you would produce constructors of a data type). I've had the same thing happen in previous compilers: When emitting C code you really need something like [c-expressions](https://github.com/orchid-hybrid/c-exprs/) or the code will become hard to work with.

* https://github.com/tca/lily/blob/master/lily/compile.scm

The compiler itself is very simple: It just recurses down the input program producing assembly code. It doesn't make good use of registers, it only really calculates with rax and pushing things onto the stack. I've experimented with register coloring before and it would be good to integrate it with this compiler..

# Results

A nice thing about this compiler is that it is easy to read and understand, and it's only about 500 lines if you don't count the code for my pattern matcher (which is also very short).

The main things I learned from doing this is:

* How do to stack frames
* Even a compiler that handles procedures, calls, conditionals, arithmetic and produces assembly can be short and simple.
* It's so important to have a nice way to produce code. (s-instructions, c-exprs, etc.)
* It helps a lot to create tools that make programming easier (e.g. pattern matcher, the sequences tool)
