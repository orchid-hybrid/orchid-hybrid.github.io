---
layout: post
title: tokenization
---

# Prereqs

You should understand regex derivatives to read this post.

* http://matt.might.net/articles/implementation-of-regular-expression-matching-in-scheme-with-derivatives/

# Tokenization

Tokenization is splitting a string of symbols

e.g.

```
123213+x*y+32122
```

into a list of tokens like

```
(number 123213)
(plus +)
(var x)
(times *)
(var y)
(plus +)
(number 32122)
```

now you can do parsing on tokens rather than on characters.

One nice advantage of this is that you can use something like an "LR(1)" parser (a parser with only one token/symbol lookahead) even if your language has keywords like 'else'. Without tokenization this might require 4 symbols of lookahead.

# Languages

So that the notion of a token is simple and doesn't depend on context we can describe them using regular expressions.

Here are the regular expressions that describes the tokens in the simple calculator language at the start of this post:

```
(define r-dig '(or (symbol #\0)
                   (or (symbol #\1)
                       (or (symbol #\2)
                           (or (symbol #\3)
                               (empty))))))
(define r-num `(seq ,r-dig (kleene ,r-dig)))

(define r-plus `(symbol #\+))

(define r-times `(symbol #\*))

(define r-var `(or (symbol #\x) (symbol #\y)))

(define alphabet (list #\0 #\1 #\2 #\3 #\+ #\* #\x #\y))
```

# Compilation by differentiation

You can make a very simple regular expression matcher (an interpreter) by differentiating the regex by each character you see as you go through the string you are testing against.

Brzozowski proved in his seminal (paywalled) paper that (as long as you simplify them a little) a regex only has finitely many derivatives.

We can use this to precompute the graph of every possible state the regex matching process can go through: regular expressions as nodes and characters/symbols as edges. This process lets us write a compiler which optimizes away all the symbolic processing of derivatives and simplification.

Since I want to handle different sorts of tokens I actually work with vectors of regular expressions. The vector `(list r-num r-plus r-times r-var)` gets compiled into the following graph:

(the format is: node1, node1, edges between them. And I renamed some of the regex vectors so it's easier to read)

```
;; start
;; num
;; (0 1 2 3)

;; start
;; ((empty) (epsilon) (empty) (empty))
;; (+)

;; start
;; ((empty) (empty) (epsilon) (empty))
;; (*)

;; start
;; ((empty) (empty) (empty) (epsilon))
;; (x y)

;; num
;; num
;; (0 1 2 3)
```

# Generated code

From that graph it's easy to generate scheme code that implements the state transitions.

Here is code generated my regvec compiler (it renamed the states as ints):

```
(define (token state char)
  (case state
    ((2) (case char
           ((#\0 #\1 #\2 #\3) '1)
           ((#\+) '3)
           ((#\*) '4)
           ((#\x #\y) '5)
           (else #f)))
    ((1) (case char
           ((#\0 #\1 #\2 #\3) '1)
           (else #f)))
    (else #f)))

(define (accepting? state)
  (case state
    ((1) number-token)
    ((2) #f)
    ((3) plus-token)
    ((4) times-token)
    ((5) var-token)
    (else #f)))

(define start-state 2)
```

# Notes

This could be a very useful tool in a wide range of places (anywhere you need a tokenizer). I just generated scheme code but you could easily target other languages.

The process of computing the graph is a kind of abstract interpretation of the regex(es). We're really precomputing every possible path the interpreter can take, "unrolling" it.

