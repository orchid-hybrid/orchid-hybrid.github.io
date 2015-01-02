---
layout: post_page
title: Register Allocation and Graph Coloring
---

Introduction
===========

This blog post is about [Ella](https://github.com/orchid-hybrid/bee/tree/master/Bee)- the code I wrote to try compiling arithmetic expressions to x86 assembly.

To represent arithmetic expressions I define an abstract syntax tree in `Language.hs`:

```haskell
data Op = Add | Sub | Mul
data E = N Int32 | O Op E E
```

Using this `1 + (2 * 3) - 5` is represented by `O Add (N 1) (O Sub (O Mul (N 2) (N 3)) (N 5))`. It might seem silly to try to compile arithmetic expressions directly because we could just evaluate them in haskell (and a good compiler would do this), but the same ideas apply when you have variables (e.g. function parameters) in the AST too. Being able to compile this is the first step to being able to functions that take some inputs and do arithmetic with it.

In assembly the instructions are opcodes with operands, the operands can be numbers, registers or memory cells but instructions only support certain combinations. You can `mov` from a register to a register, but not from a cell to a cell. Since you can't usually operate on memory cells alone you have to have one of the operands a register but there are only a small number of them in 32-bit x86 so you have to make use of them well.

When I decided to program this I came up with some vague ideas on how to do it but overall was worried that some expressions might end up needing a move from cell to cell (which isn't allowed) and there might be no free registers left. So I wanted a way that would always work and that I could optimize afterwards.

One Note Samba
===============

Surprisingly it's possible to compile arithmetic evaluating using lots of memory cells and only only register! After that you can optimize it by replacing some of the cells with registers - that will make the compiled code faster and better because it uses less resources.

The technique to compile arithmetic with only a single register is quite simple (from Ella.hs):

```haskell
ella :: E -> It () -> WriterT [A ()] (Fresh Int) ()
ella (N i) r = tell [AMov r (Num i)]
ella (O op x y) r = do
 a <- lift $ fresh
 ella y (Cell a)
 ella x (Register ())
 case op of
  Add -> tell [AAdd (Register ()) (Cell a)]
  Sub -> tell [ASub (Register ()) (Cell a)]
  Mul -> tell [AMul (Cell a)]
 if r == Register ()
    then return ()
    else tell [AMov r (Register ())]

runElla exp = snd $ runFresh (runWriterT $ ella exp (Register ())) [0..]
````

The code says that to compile a number `(N i)` into place `r` you should do the instruction `[Mov r (Num i)]`.

Then to compile a complex expression `O op x y` (where `op` might be +, - or *) get a fresh/unsued memory cell `a` and compute `y` into it, then compute `x` into the one special register. Now you can perform the operation (add, sub or mul) with the register and the cell which is an acceptable combination of operands. After that it might be required to move the result out of the register.

The reason this works is that `y` and even the computation of `x` can use the register all they want, it only matters that it holds the result of computing `x` at the end - and once the operation is done we no longer need the register.

x86 is a complex instruction-set architecture (as opposed to a reduced one) so it has a lot of complexity, for one thing mul works different to add and sub. it always works with the EAX register (which makes us use that register as our special register) and it also overwrites EDX with the result (because multiplication of two small numbers can result in something really huge, the result comes out in two registers rather than one).

Life, Death and Register Coloring
=================================

I tested this by comparing the results of an evaluator with the result of compiling and executing the assembly code this algorithm generates on lots of randomly generated expressions (from Main.hs) and it works! Of course we were extremely greedy in allocating as many memory cells as we wanted. To compile some small expressions could take hundreds of words of memory.

So a lot of memory would be saved if we could be economical and reuse memory cells when there is no conflict. You can consider the moment a value is put into a cell (or register) for the first time as it's birth - and the last time it is accessed as its death. Here are two short examples with their lifetimes diagrammed:

```
test1 =  (O Add (O Add (O Add (N 1) (N 2)) (N 3)) (N 4))
-- AMov (Cell 0) (Num 4)       |
-- AMov (Cell 1) (Num 3)       | |
-- AMov (Cell 2) (Num 2)       | | |
-- AMov (Register ()) (Num 1)  | | |
-- AAdd (Register ()) (Cell 2) | | |
-- AAdd (Register ()) (Cell 1) | |
-- AAdd (Register ()) (Cell 0) |

test2 =  (O Add (N 1) (O Add (N 2) (O Add (N 3) (N 4))))
-- AMov (Cell 2) (Num 4)        |
-- AMov (Register ()) (Num 3)   |
-- AAdd (Register ()) (Cell 2)  |
-- AMov (Cell 1) (Register ())   |
-- AMov (Register ()) (Num 2)    |
-- AAdd (Register ()) (Cell 1)   |
-- AMov (Cell 0) (Register ())    |
-- AMov (Register ()) (Num 1)     |
-- AAdd (Register ()) (Cell 0)    |
```

So the first one needs all 3 cells, but the second expression could be computed only using a single cell since there's no overlap of the lifetimes! To write a program that figures this out we need a data structure that we can encode whether or not two cells lifetimes overlap: A graph whose vertices are cells, with an edge connecting every two cells that have overlapping lives would be perfect!  Here are the two examples turned into graphs:

<p align="center">
  <img src="http://i.imgur.com/jO2nhvY.png" />
</p>


Graph Coloring
==============

This part is a bit like the four color theorem, although we might need more than four!

<p align="center">
  <img src="http://i.imgur.com/Zdth5is.gif" />
</p>

You have to color in the graph so that no two neighbours have the same color - and we want to do it with a small number of colors. It's really hard to compute the smallest number of colors, but it's not too hard to color it in with a relatively small number of colors using a heuristic.

What my code (in Coloring.hs) does is look at the vertices in order starting with the one which has the most neighbours - just coloring in each node with the first color none of its neighbours use - that's a greedy algorithm. There are much better algorithms but they're more complex! Once the graph has been colored we can use this information to optimize the assembly code - it will only use as many registers and memory cells as there are colors. Before this it wastefully used one cell per vertex of the graph!

Since registers are the fastest to work with we turn the memory cells that correspond to the most commonly used colors into registers. After doing this I ran lots more random tests to check it worked and got some really good results:

```
  ;; exp: O Sub (O Mul (O Mul (O Mul (O Mul (N 4) (N (-12))) (O Add (N 11) (N 13))) (O Add (O Mul (N 16) (N (-10))) (O Mul (N (-30)) (N 17)))) (O Mul (O Mul (O Mul (N 17) (N (-7))) (O Add (N 15) (N 19))) (O Mul (O Add (N (-29)) (N 23)) (O Add (N (-22)) (N (-16)))))) (O Mul (O Add (O Sub (O Mul (N (-12)) (N 22)) (O Sub (N 7) (N (-4)))) (O Mul (O Add (N 15) (N (-23))) (O Sub (N 13) (N (-28))))) (O Mul (O Add (O Mul (N (-18)) (N 27)) (O Add (N 9) (N 26))) (O Mul (O Mul (N (-8)) (N (-30))) (O Mul (N 9) (N (-8))))))
  ;; result: 1355813760
  ;; naive spills: 31
  ;; optimized spills: 3
```

This big complicated expression got compiled by Ella into instructions that needed 31 memory cells, but with register allocation it only uses 3!
