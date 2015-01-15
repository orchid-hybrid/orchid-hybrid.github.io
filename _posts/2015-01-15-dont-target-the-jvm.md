---
layout: post
title: Don't target the JVM.. and what's the point anyway?
---

There's a really nice paper by Graham Hutton, <a href="http://www.cs.nott.ac.uk/~gmh/ccc.pdf">Calculating Correct Compilers</a>. From a simple language with an eval function he derives a virtual machine and compiler simultaneously. One of the main techniques used is defunctionalization.

For lambda calculus the virtual machine that comes out is this:

```
data Code = HALT | PUSH Int Code | ADD Code | LOOKUP Int Code | ABS Code Code | RET | APP Code
```

which is *almost* a list, as you can see by rewriting it:

```
type Code = [Inst]
data Inst HALT | PUSH Int | ADD | LOOKUP Int | ABS Code | APP
```

but it's not! The `ABS` instruction has its own code in it too, so this is really a tree: We can't just write this out as a flat sequence of bytecode instructions.. On the other hand it is not hard to see how to flatten this out into linear array of instructions, just have the ABS instruction contain a code saying how long its code is. There are some more issues to resolve like how values contain code, but inspecting the semantics of the virtual machine you can see it's ok to just make that a pointer into the code array.

To make sure this is works as I imagine and there's no "cheating" involved I wanted to compile this down into a real VM that forces a flat instruction format on you. I chose the JVM which is stack based just like this VM and first compiled Huttons Razor (the language with just + and integers) into .class files using gnu.bytecode from the kawa scheme compiler.

Then I started to work on compiling `Code` above to the JVM, the language for lambda calculus. The JVM is very strict though: You have no computed jumps, every jump location must be statically known. I can handle that using <a href="http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html#jvms-6.5.tableswitch">tableswitch</a> - just pass around table indices instead of code pointers.

There is a real problem with that though, I wanted to use the jvm stack for this VMs stack but the JVM is very strict about validating bytecode. It checks a lot of things including doing a bit of dataflow analysis to make sure that the stack is the same no matter which code paths are taken!

>  If an instruction can be executed along several different execution paths, the operand stack must have the same depth (§2.6.2) prior to the execution of the instruction, regardless of the path taken.

from <a href="http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.9.2">structural constraints</a>

So if I were to use a large tableswitch for code pointers each case would have to do the same stack operations. Looking at `exec` (the semantics of the VM) there is no way that this will be the case! Sadly this means I have to manage my own stack and only use JVMs stack for scratch calculations. Realizing that I felt like there would not be much point in compiling to th JVM anymore, and wondered what's the point of compilation at all?

The final chapter of SICP covers this a bit, they turn their interpreter into a compiler and this removes the overhead of walking around the AST checking what type of node they are at and calling out to. In compiled code it would be able to just do the instruction in front of its nose and be told exactly where to jump when it has to. They are also able to use much fewer registers because of this.

For a more concrete example imagine implementing code for a VM by writing out the instructions as C macros: `PUSH(x); ADD(); RET();` vs writing them as C functions `push(x); add(); ret();`. In the functions version it will uselessly create a stack frame, jump to some other place in code, pop the stack frame and jump back.. 3 times (unless the compiler optimizes that, but that's not the point).

This is similar to specialization, things are statically known (i.e. the source code of the program being run) so you can do work on it at compile time and inline the results to reduce the overhead. That could give opportunities for more optimizations too. An interpreter + specialization can produce a compiler because it's capable of the two important things a compiler needs to do: translation (you can actually "port" code from one language to another automatically!) and optimization (specialization can inline all that interpretive overhead).. but the futamura projection is not all sunshine and daisies: I don't think it can translate to a very low level so it's only a compiler in the relaxed sense of the word (translating from one language to another) and it's only doing quite a basic optimization - you would want to implement more interesting ones yourself. So in an engineering sense I don't think this is a worthwhile technique to apply - I could very well be wrong though! I would love to see examples of this being pulled off.

* <a href="http://www.brics.dk/RS/03/13/BRICS-RS-03-13.pdf">A functional Correspondence between Evaluators and Abstract Machines</a>
