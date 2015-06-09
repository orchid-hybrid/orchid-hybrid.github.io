---
layout: post
title: Term rewriting
---

Earlier I found there was a bug in a simplifer regular expressions that I wrote. It didn't always simplify fully. This was causing an infinite loop in a program that needed them to be fully simplified.

It's easy to express this kind of algebraic simplification using term rewriting - so instead of fixing the buggy simplifier I chose to implement a term rewrite system.

It turns out that term rewriting is just a couple loops away from pattern matching. To perform term rewriting we just attempt to pattern match for a rewrite rule over each subexpression of the input term.. and do that in a loop until no more rewrites occur.

My pattern matcher [match-sagittarius](https://github.com/orchid-hybrid/match-sagittarius/tree/master/match) is built out of three modular components:

* An macro interpreter for pattern match "virtual machine" instructions.
* A compiler that turns patterns into sequences of match instructions.
* A trie data structure that is used to optimize away shared work between branches.

Because of this it's easy to change the pattern syntax for the matcher.

After changing the quasiquote pattern syntax to a simpler 'term rewrite' syntax, we made a macro that turns a bunch of term rewriting clauses into a pattern match worker function. Then we made a `define-rewrite-system` macro that locally defines the worker function and runs it in a couple of loops. Done!

```
(define-rewrite-system f
  ((d x x) --> (one))
  ((d x y) where (symbol? y) --> (zero))
  ((d x (+ u v)) --> (+ (d x u) (d x v)))
  ((d x (* u v)) --> (+ (* u (d x v))
			(* (d x u) v))))

(f '(d x (* x x)))) ;=> (+ (* x (one)) (* (one) x))
```

* [and-all-that](https://github.com/orchid-hybrid/and-all-that)
