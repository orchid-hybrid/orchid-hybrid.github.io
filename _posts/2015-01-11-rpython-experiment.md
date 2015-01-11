---
layout: post
title: RPython experiment
---

There is this project called pypy which implements a python interpreter in using the RPython translation toolchain which compiles it down into C code with a JIT - A Just In Time tracing compiler that tries to identify hot loops and then watches the interpreter executing them so that it can try to specialize the traces into more efficient compiled versions. You can use the same tools to turn any bytecode interpreter into a JIT based interpreter (I've heard that the tools work much better on bytecode than an AST based interpreter).

You can get a lot of really good technicaly information about how this is done from the following two links <a href="http://www.aosabook.org/en/pypy.html">The Architecture of Open Source Applications</a>,
<a href="http://tratt.net/laurie/blog/entries/fast_enough_vms_in_fast_enough_time">Fast Enough VMs in Fast Enough Time</a>. There's a good tutorial for beginners on the main pypy blog here: <a href="http://morepypy.blogspot.co.uk/2011/04/tutorial-writing-interpreter-with-pypy.html">Writing an Interpreter with PyPy</a> and some more nice example code on bitbucket <a href="https://bitbucket.org/pypy/example-interpreter">Kermit</a>.

I had heard this 'generating a jit compiler from an interpreter' thing was based on the 3rd futamura projection but im a little skeptical about that, I think that a tracing JIT specializer is a slightly different thing. I am probably wrong or being too specific about terms but it's not too important: pypy is about 6x faster than cpython - that's what matters.

So I wanted to try this stuff out - I thought it would be really interesting to apply it to the UM virtual machine from ICFP 2006. My plain C implementation takes 20.8 seconds to run (when compiled with -O3, at first I did the comparison without -O3 since I forgot and the results were incredible.. you have to be careful when doing "science"), the RPython version takes 16.9 seconds. It is faster but not by too much. I think an RPython expert could probably make this a lot faster I don't really know how to.

