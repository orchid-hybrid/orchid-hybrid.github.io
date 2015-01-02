---
layout: post
title: Compiling Regexes
---


I decided to implement a regex->asm compiler for practicing at emitting assembly code and writing compilers.
At first I thought I would create an NFA graph and optimize it and transform it into a DFA before emitting assembly code where each DFA state would have its own label and code.
Some friends pointed me towards this site though: [Regular Expression Matching: the Virtual Machine Approach](http://swtch.com/~rsc/regexp/regexp2.html) which covers a much more direct way to compile regexes not involving any graphs!


Here is the full implementation of the compiler

```
module Regex where

import Control.Monad.Trans
import Control.Monad.Writer
import Data.Monoid

import Fresh
import X86

data Regex
 = Char Char
 | Regex :^ Regex
 | Regex :| Regex
 | Star Regex

data Bytecode
 = CHAR Char
 | LABEL Label
 | SPLIT Label Label
 | JUMP Label

compile :: Regex -> WriterT [Bytecode] (Fresh Label) ()
compile (Char c) = tell [CHAR c]
compile (r1 :^ r2) = do compile r1 ; compile r2
compile (r1 :| r2) = do
 l1 <- lift $ fresh ; l2 <- lift $ fresh ; l3 <- lift $ fresh
 tell [SPLIT l1 l2]
 tell [LABEL l1] ; compile r1 ; tell [JUMP l3]
 tell [LABEL l2] ; compile r2
 tell [LABEL l3]
compile (Star r) = do
 l1 <- lift $ fresh ; l2 <- lift $ fresh ; l3 <- lift $ fresh
 tell [LABEL l1, SPLIT l2 l3]
 tell [LABEL l2] ; compile r ; tell [JUMP l1]
 tell [LABEL l3]

-- rax is the buffer pointer
-- rbx is the end
elab :: Bytecode -> [X86]
elab (CHAR c) = 
 [ Cmp Nothing (R RAX) (R RBX)
 , Jge ".fail"
 , Cmp (Just Byte) (DR RAX) (C c)
 , Jne ".fail"
 , Inc (R RAX)
 ]
elab (LABEL l1) = [Label l1]
elab (SPLIT l1 l2) =
 [ Push (R RAX)
 , Push (V l2)
 , Jmp l1
 ]
elab (JUMP l1) = [Jmp l1]

runCompile regexp = (snd $ runFresh (runWriterT $ compile regexp) labels) >>= (s . elab)
 where labels = map ((".l"++).show) [0..]
```

This takes a regular expression and translates it into the bytecode instructions that the page described. Then I do a little post processing to turn that into assembly code.
This creates a bit of code that I can paste into my regex.asm file to get a program that acts like "grep" but for a single regex.

Matching regexes is done by the x86 code by moving a pointer through a buffer as it checks each character. The way that I implement SPLIT is to push the second address and pointer into the string onto the stack. If a character match fails it will check the stack to see if it can backtrack to parse the string in a different way. 

Here's an example of compiling a regex with this:

```
*Regex> putStr . runCompile $ (Char 'm' :^ (Star (Char 'o') :| (Char 'u' :^ Char '!')))
  cmp  rax,rbx
  jge  .fail
  cmp  byte  [rax],'m'
  jne  .fail
  inc  rax
  push rax
  push .l1
  jmp  .l0
.l0:
.l3:
  push rax
  push .l5
  jmp  .l4
.l4:
  cmp  rax,rbx
  jge  .fail
  cmp  byte  [rax],'o'
  jne  .fail
  inc  rax
  jmp  .l3
.l5:
  jmp  .l2
.l1:
  cmp  rax,rbx
  jge  .fail
  cmp  byte  [rax],'u'
  jne  .fail
  inc  rax
  cmp  rax,rbx
  jge  .fail
  cmp  byte  [rax],'!'
  jne  .fail
  inc  rax
.l2:
````

And the host file or "runtime" that enables this to run: https://github.com/orchid-hybrid/x86-asm-practice/blob/master/64/regex.asm
