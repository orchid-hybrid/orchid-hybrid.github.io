Notes on Compilers
==================

I set myself a goal to write a bootstrapping scheme compiler. I had a rough idea how it was done from Marc Feeley's talk on writing a [scheme->c compiler in 90 mins](http://lambda-the-ultimate.org/node/349). I wrote up a parser quickly and made sure it was self-applicable. I didn't work on the compiler project until a few weeks after that.

One day I remembered about it and got hacking! Wrote this [scm](https://github.com/orchid-hybrid/scm) but it didn't quite make it. It was too inefficient and incomplete. When I tried to bootstrap it, it used up all possible memory quickly and crashed horribly. A really obvious reason for this was that I didn't have garbage collection orr tail call optimization both of which are extremely important in scheme! In fact looking back: I did not have optimizations.

One interesting thing I learned thinking about how to improve that attempt was that garbage collection would be almost worthless if I did not also do tail call elimination. The stack frame is the root for GC to find live data from and wasted stack frames lying around would block the GC from getting rid of a lot of environment variables that are no longer used.

So I got my friends to collab on it with me and tried again [moo](https://github.com/orchid-hybrid/moo). This time I implemented a garbage collector and a simple my stack based runtime with a trampoline in C. This was the first time I've ever implemented a GC so it was exciting to see it working in practice (A good way to test it is a recursive infinite loop). This compiler also has tail call optimization (since unlike the previous compiler it uses CPS tranformation, combining that with the trampoline gives you TCO for free).

Moo manages to successfully compile a bunch of interesting programs like binary trees, derivatives and the n-queens solver from SICP.. but it was having some huge code-blowups in the output it produces. So I [asked for some help on ltu](http://lambda-the-ultimate.org/node/5042) and thanks to Eric Bergstrome who noticed an explosion in the CPS code we got a big reduction! There were a few bugs in it and my friend did some incredible work pushing it further and further but it just would not take off.

So I tried again with a different approach - XP/test driven development, I wanted to make all the parts modular and independent and thoroughly tested. It was also based on building up the way instead of down: [Moloch](https://github.com/orchid-hybrid/Moloch). Development slowed and I started thinking about the big picture more than the details: I felt like it was silly to build my own stack data structure in C, and using a trampoline means the C stack itself would "vibrate" a lot. There is a very clever way to make use of the C stack effectively called Cheney on the MTA (Instead of vibrating with lots of tiny little jumps it waits and waits and does a gigantic leap "Off the empire state building") but I'm not using that advanced technique for this compiler! Also I was starting to feel a bit bad about spending so much time on it, I felt like I was neglecting the other parts of my life.

At this point I have learned that writing a scheme compiler is much harder than I first thought!

I studied the RABBIT and ORBIT papers more deeply 
* RABBIT: ftp://publications.ai.mit.edu/ai-publications/pdf/AITR-474.pdf
* ORBIT: repository.readscheme.org/ftp/papers/orbit-thesis.pdf
and I realized that I was missing a really important optimization: Closure analysis (where you decide whether or not you can stack allocate a closure). More generally I realized [this](http://i.imgur.com/S7hQDts.png) - A proper compiler is more than just a translator from one language to another, it must also optimize the code. I felt like I had been missing half the picture this whole time.

### Translation

* Parsing
There isn't much interesting to say about this, parsing lisp is easy compared to other languages.

* Macros: https://github.com/orchid-hybrid/syntactic-closures-macro-system
Bawden Rees syntactic-closures paper is excellent and the code from it works.
Add a little interpreter and you have a full blown macro system - that said, syntactic closures is not nice to use directly. This is implemented in scm0 which I haven't uploaded. I would rather implement syntax-rules in terms of it though - but what about CK macros?

* Closure Conversion
To compile a lambda term you close over all its free variables into an environment vector and pass that as an extra parameter.
It was very surprising to me that this was all you needed* to add lambda to C. (* not quite true: you also need GC - and there's basically no way to add this with the way C works. On the other hand with substructual types.. linear (for heap) or ordered types (for stack)).. This could make a fun demonstration.

* CPS transform
This is very interesting. It allows a trivial implementation of CWCC, since it's CPS semantics are in CPS form. This is not the case with SHIFT/RESET, they require a different implementation. It's not clear how.

A quote from Olin Shivers thesis:
> CPS-based compilers raise the stakes even further. Since all control and environment structures are represented in CPS Scheme by lambda expressions and their application, after CPS conversion, lambdas become not merely common, but ubiquitous. The compiler lives and dies by its ability to handle lambdas well. In short, the compiler has made an explicit commitment to compile lambda cheaply — which further encourages the programmer to use them explicitly in his code.
> Implementing such a powerful operator efficiently, however, is quite tricky. Thus, research in Scheme compilation has had, from the inception of the language, the theme of taming lambda: using compile-time analysis techniques to recognise and optimise the lambda expressions that do not need the fully-general closure implementation.


### Optimization

* Constant Elimination
In earlier compilers 'quoted data gets evaluated every time. This is pretty wasteful: pull them out to the toplevel.

* Closure Conversion
To compile lambda you need to perform closure conversion. Make a vector of all the free variables each lambda closes-over and pass it in as an extra parameter.

* Closure Analysis/Escape Analysis
Stack allocation is faster than heap allocation because anything that gets allocated on the heap cannot be removed until a full GC operation occurs, stack gets reclaimed just by popping (incrementing an integer). Heap allocation is the most general strategy but in many many situations it's possible to stack allocate a closure.
I think this is the most important thing that my second compiler is missing that stops it from being able to bootstrap. From the [Bigloo paper](http://www.classes.cs.uchicago.edu/archive/2006/spring/32630-1/papers/p118-serrano.pdf) "Our figures show that closure optimization leads to executable programs running 70% faster"
I started to study Olin Shivers and Matt Mights Theses (and [this post](http://matt.might.net/articles/implementation-of-kcfa-and-0cfa/)) about the 0-CFA algorithm but it's really really difficult.

* Register Coloring
One technique to minimize register usage is by graph coloring. I blogged about this as 'Ella' see minicompilers.
TODO: [Read Register Allocation By Model Transformer Semantics](http://arxiv.org/abs/1202.5539)

* Preserving registers (Stack optimization)
[SICP shows](http://mitpress.mit.edu/sicp/full-text/book/book-Z-H-35.html#%_sec_5.5) how to optimize stack usage when saving registers onto the stack. This is implemented in the lisp0 (lisp without lambda) practice compiler. Most of the code in that is taken from SICP.


### Books/Papers/Compilers

* Chicken scheme notes:
http://wiki.call-cc.org/chicken-compilation-process
http://wiki.call-cc.org/chicken-internal-structure

* Cheney on the M.T.A: 
chicken developers post about it: 
http://home.pipeline.com/~hbaker1/CheneyMTA.html
http://www.more-magic.net/posts/internals-gc.html

* Nanopass
Andy Keep explains about nanopass, A nice way to organize the parts of a compiler: https://www.youtube.com/watch?v=Os7FE3J-U5Q
Ghuloum explains how to build a compiler from assembly upwards: http://scheme2006.cs.uchicago.edu/11-ghuloum.pdf

* Compiling with Continuations
Marc Feeleys talk. Appel's book.

* Control Flow Analysis for Higher Order Languages, or Taming Lambda: http://www.ccs.neu.edu/home/shivers/papers/diss.pdf


### What else is out there

I tried reading other peoples code a bit to try to learn some stuff. It was interesting but I don't feel like it helped me much. Larger compiler projects are even harder to get into

* ichbins: https://github.com/darius/ichbins/
* maru: http://piumarta.com/software/maru/#original
* kashiwa: https://github.com/zick/kashiwa
* scheme_to_C: https://github.com/Johnicholas/Hello-Github/blob/master/scheme_to_C/compile.ccode.ss (did not bootstrap..)
* c of scheme: http://jozefg.bitbucket.org/posts/2014-06-07-c-of-scheme.html - interesting comments by cameron swords
* BONES: http://www.call-with-current-continuation.org/bones/
* ur-scheme: http://canonical.org/~kragen/sw/urscheme/ - a bunch of well written info about it!
* mincaml: a compiler for a cut down version of ocaml http://esumii.github.io/min-caml/index-e.html


### Mini-compilers

Since the big compiler was too hard on my first two tries, I decided to work on much smaller easier compiler projects. This would give me useful practice and get me closer to my goal.

* Ella: http://orchid-hybrid.github.io/2014/09/21/register-graph.html
I compiled arithmetic expressions down to assembly code. This was doing using a one register + memory cells IR then graph coloring to make it use a minimal number of registers and memory cells. This could be developed into a tiny imperative language?

* Tree: http://orchid-hybrid.github.io/2014/09/29/flattening-trees.html
I compiled trees into sequences of instructions that build those trees on the stack. This involved tagging the lowest nybble of the words that store the data with their type. This was good practice writing assembly inspecting the stack with gdb.

* Regex: http://orchid-hybrid.github.io/2014/10/24/compiling-regexes.html
I compiled regular expressions to assembly using a 'bytecode' IR. This was also really good practice programming in x86.

* Lisp0 (not finished): https://github.com/orchid-hybrid/lisp0
I started working on "lisp0", the idea was to have a very trivial lisp language without lambda. The compiler the "preserving" technique from chapter 10 of SICP to minimize the stack usage.

