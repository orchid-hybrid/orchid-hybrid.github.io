---
layout: post
title: Big Picture Notes
---

## Instruction Set Architectures

The Prolog book 'Clause and Effect' covers writing a simple compiler in prolog, it is roughly based on [Applied Logic and Its Use and Implementation As A Programming Tool](http://www.sri.com/work/publications/applied-logic-its-use-and-implementation-programming-tool) and it writes a compiler for the same language targetting 3 different instruction set architectures. In order of complexity/difficulty they are:

* targeting a stack machine architecture: this is easy since you just emit reverse polish notation
* targeting a one register (accumulator) instruction set: this is very simple (I used this idea in Ella)
* targeting a finite register machine: This is quite tricky! Of course a good optimizing compiler must make efficient use of registers.

This is in accordance with [TheSimplestPossibleCompiler](http://c2.com/cgi/wiki?TheSimplestPossibleCompiler) and [XpForOptimizingCompilers](http://c2.com/cgi/wiki?XpForOptimizingCompilers) which both mention that the simplest possible compilers target stack architectures.


## Parsing expression grammar

PEG is a very simple powerful language for recognizing languages as well as translating them into another format. There is an excellent literate program writeup by Kragen on his [bootstrapping PEG compiler](https://github.com/kragen/peg-bootstrap/blob/master/peg.md). There was also a great blog series on embedding PEG inside mercury - which really shows the strength of Mecury: [adventures in mercury:PEG](http://adventuresinmercury.blogspot.co.uk/search/label/PEGs).

I came across another [implementation in macropy](https://github.com/lihaoyi/macropy?files=1#macropeg-parser-combinators) which automatically generates good error messages. It would be nice to add similar automatic error message support to kragens PEG by bootstrapping the updated compiler..

A paper from the STEPS project [PEG-based transformer provides front, middle and back-end stages in a simple compiler](http://www.vpri.org/pdf/tr2010003_PEG.pdf) documents an experiment to use a slightly extended version of PEG (that handles s-expressions as well as strings of characters) to write a compiler. The entire compiler is written in 3 short PEGs, one for parsing the text into an s-expression, one of turning that into a stack based intermediate language, and finally one that emits assembly code from that.


## Pattern match compiler

The way that the STEPS PEG-based compiler processes s-exps is similar to a pattern matcher compiler that I wrote recently. It operates as a VM with some instructions (which are implemented using syntax-rules) processing a stack of s-expressions, one of the operations takes the top s-exprssion and when it's a list pushes each of its elements onto the stack.

The compiler itself was written in 3 steps, first I compiled a single pattern into a sequence of instructions that the macro could transform into code that would match against that pattern. Once this was working nicely I made it compile a whole list of patterns and then merge the lists of instructions into a trie. This allows the implementation to share some of the destructuring work.

Write-up and source of that [here](https://gist.github.com/orchid-hybrid/4901f7dd330be112d52e)


## Architecture

Writing a compiler is very hard so it's important to do things that reduce the difficulty. One of the best ideas is 'separation of concerns'. An example of this that I'm making use of is related to generating code:

Any compiler (written in lisp) that emits C code as output would have to print out all the brackets and semicolons and other C-syntax. This gets really really ugly as evidenced here [scm/c/c.scm](https://github.com/orchid-hybrid/scm/blob/master/c/c.scm), and it is also very difficult to work with.

A solution to that is [c-exprs](https://github.com/orchid-hybrid/c-exprs) which handles emitting C syntax. Its input is an s-expression language that represents C code and it's far easier to work with. To separate even further this is now a stand-alone program rather than just being a piece of code that a compiler would call upon.



## Attribute Grammars

Another technique which looks excellent for managing the complexity of programming a compiler is Attribute Grammars, this is a kind of aspect oriented programming that lets you decompose a program in a direction not normally possible. UHC uses this and the slides on [The Architecture of the Utrecht Haskell Compiler](http://foswiki.cs.uu.nl/foswiki/Ehc/TheArchitectureOfTheUtrechtHaskellCompiler) give a rough overview of how that is done. Sadly the compiler itself does not work on my computer. There is a good introduction to attribute grammars in [Why Attribute Grammars Matter by Wouter Swierstra](https://www.haskell.org/haskellwiki/The_Monad.Reader/Issue4/Why_Attribute_Grammars_Matter).
