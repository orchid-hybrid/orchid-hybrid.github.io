---
layout: post
title: Incy Wincy Spider
---

# Introduction and language overview

For a while I've been working on a reference counting based compiler from a pure lisp to C.

For reference counting to recover all garbage the language has to be pure otherwise you could create cycles e.g. `(set-cdr! c c)` - it also must be strict since you can create cycles in a lazy language by data recursion. The fact that reference counting works on a pure, strict language can be seen by thinking about the time or order objects are allocated in: If an object only contains objects older than itself then there cannot be any cycles and the pointers all form a tree.

The language is called `niea` and the frontend is [here](https://github.com/tca/NieA). This compiler validates a `.niea` file making sure it is syntactically correct and well scoped, then it performs closure conversion to erase lambda - emitting a first order `.ref` file. The language is quite basic: you can define (recursive) functions, use lambda freely and call procedures.

The `.ref` language is a pure\* first order language with automatic reference counting (\* there are side effect operations to print text out, but that does not cause any issues with refcounting). The compiler for this language (the backend) is called [spider](https://github.com/orchid-hybrid/spider) - it emits `.cexpr` files which is an s-expression syntax for C programs [c-exprs](https://github.com/orchid-hybrid/c-exprs) translates that into C which you can compile. It has some special builtins for creating vectors either from given elements or by calling a given "generator" procedure `n` times to create a length `n` vector.

# Reference counting calling convention

For a compiler to emit code that is consistent with itself each function has to use same calling convention. I designed a simple calling convention for reference counting that seems reasonable (not causing too many pointless incs and decs). A calling convention has to tell both the CALLER and the CALLEE of a procedure what rights and responsibilities they have.

## CALLER

### Parameters

* When passing a parameter into a procedure call the caller gives up its reference to that object.

### Return values

* When a value is returned by a procedure call the caller is given a reference to that object and is obligated to consume it (by decrementing the reference count after the object is not needed or by passing off to someone else).

### Example

Suppose a caller has references to some objects `x`, `y`, `z` and performs the following call `(f x y z)` then it has given up its references to all thoes objects. An implication of this is that if a caller wants to continue to use `y` after this function call it must increment the reference count of y before performing that call.

When the procedure call returns then x and z have been consumed (say) but some memory was probably allocated in building the result so the caller must use up the reference it was handed over.

## CALLEE

### Parameters

* The callee is given one reference to each of its parameters and must consume them.

### Return values

* The callee must hand over one reference in returning an object.

### Example

Inside the following procedure `(define (f x y z) x)` the callee `f` has one reference to each of `x`, `y` and `z`. To satisfy the calling convention the compiler should decrement the reference count of `y` and `z` but not `x`.

## Example

Almost every situation that arises comes in the following code:

```
(define (k x y)
  x)

(define (s x y z)
  ((x z)
   (y z)))
```

to satisfy the calling convention these would be compiled like this:

```
(define (k x y)
  (set! result x)
  (dec y)
  (return result))

(define (s x y z)
  (inc z)
  (set! t1 (x z))
  (set! t2 (y z))
  (set! result (t1 t2))
  (return result))
```

In `k` `y` is not used so has to be decremented. In `s` `z` is used twice so has to be incremented. The calls `(x z)` and `(y z)` consumes (one reference to) `x` and `z` as well as one reference to `y` and `z`, the call `(t1 t2)` consumes the result of those two calls and since the result of that is returned everything is balanced.

## If

Only supporting procedure calls is already enough to do conditionals but I wanted to directly implement `if` rather than using church encoding or a builtin that requires wrapping the then/else clauses in an extra lambda. Interestingly `if` is slightly difficult. The reason for this is because it is a control flow operator: in `(if b t e)`, dependening on `b` only one of the two branches will be followed.

What this means for reference counting is that we need to "equalize" the reference usage between branches. E.g. if one branch uses x twice and the other uses x once, we should add a decrement inside the second branch - so that now both use x twice. I chose to take the maximum number of usages of each variable between each branch and make the smaller one perform enough decs to equalize. There may be other strategies too.

# Results

Now for some results! If the HTML works you should be able to see rows code:graph.

## UFO2

<pre>
(define (count-change amount)
  (cc amount 5))

(define (or x y) (if x 1 y)) ;; strict!

(define (cc amount kinds-of-coins)
  (if (= amount 0) 1
      (if (or (&lt; amount 0) (= kinds-of-coins 0))
          0
          (+ (cc amount
                 (- kinds-of-coins 1))
             (cc (- amount
                    (first-denomination kinds-of-coins))
                 kinds-of-coins)))))

(define (first-denomination kinds-of-coins)
  (if (= kinds-of-coins 1) 1
      (if (= kinds-of-coins 2) 5
          (if (= kinds-of-coins 3) 10
              (if (= kinds-of-coins 4) 25
                  ;;(= kinds-of-coins 5)
                  50)))))

(define (scm-main)
  (print (count-change 100)))
</pre>


<img src="http://i.imgur.com/yj2gFUD.png">


## UFO15

<pre>
(define (begin a b) b)

(define (go n)
 (if n
     (go (- n 1))
     0))

(define (scm-main)
 (begin (go 10)
 (begin (go 50)
 (begin (go 100)
 (begin (go 10)
 (begin (go 10)
 (begin (go 20) 0)))))))
</pre>

<img src="http://i.imgur.com/Af9uqGA.png"></img>

## UFO14<

<pre>
(define (words)
 (cons "mother" (cons "in" (cons "law" (cons "woman"
  (cons "hitler" (cons "debit" (cons "card"
  (cons "bad" (cons "credit" (cons "school" (cons
  "master" (cons "the" (cons "classroom" (cons
  "eleven" (cons "plus" (cons "two" (cons "twelve"
  (cons "plus" (cons "one" (cons "dormitory"
  (cons "dirty" (cons "room" (cons "punishment"
  (cons "nine" (cons "thumps" (cons "the" (cons
  "morse" (cons "code" (cons "here" (cons "come"
  (cons "dots" (cons "a" (cons "decimal" (cons
  "point" (cons "im" (cons "a" (cons "dot" (cons
  "in" (cons "place" (cons "astronomer" (cons
  "moon" (cons "starer" (cons "the" (cons "eyes"
  (cons "they" (cons "see"
  0)))))))))))))))))))))))))))))))))))))))))))))))
 
 ;;;; The program is a bit long so not shown in full here
 ;;;; it allocates that big list of words
 ;;;; then builds a trie out of them and prints them in order

</pre>


<img src="http://i.imgur.com/FLtdEr3.png"></img>

## UFO11

<pre>
(define (cons x y)
 (lambda (selector)
  (selector x y)))
(define (car c)
 (c (lambda (x y) x)))
(define (cdr c)
 (c (lambda (x y) y)))

(define (begin a b) b)

(define (for-each f l)
  (if l
      (begin (f (car l))
             (for-each f (cdr l)))
      0))

(define (map f l)
  (if l
      (cons (f (car l))
            (map f (cdr l)))
      0))

(define (append a b)
  (if a
      (cons (car a) (append (cdr a)  b))
      b))

(define (loop acc spaces n)
  (if n
      (loop
       (append
        (map (lambda (x) (append spaces (append x spaces))) acc)
        (map (lambda (x) (append x (append (cons " " 0) x))) acc))
       (append spaces spaces)
       (- n 1))
      acc))

(define (sierpinski n)
  (for-each
   (lambda (x) (begin (for-each print x) (print "\n")))
   (loop (cons (cons "*" 0) 0) (cons " " 0) n)))

(define (scm-main)
  (sierpinski 4))

</pre>
<img src="http://i.imgur.com/ANZosvn.png"></img>


## UFO13

<pre>
(define (forward-backwards l-r k)
  (k l-r (reverse l-r)))

(define (foo l r n)
  (if n
      (forward-backwards
       (append l r)
       (lambda (fwd bwd)
         (begin (for-each (lambda (c) (for-each print c)) fwd)
                (begin (print "\n")
                       (foo (map (lambda (c) (cons "0" c)) fwd)
                            (map (lambda (c) (cons "1" c)) bwd)
                            (- n 1))))))
      (print-numbers (append l r) 0)))

(define (print-numbers l n)
  (if l
      (begin (print "\n")
             (begin (print n)
                    (begin (for-each print (car l))
                           (print-numbers (cdr l) (+ n 1)))))
      0))


(define (scm-main)
  (foo (cons (cons "0" 0) 0) (cons (cons "1" 0) 0) 4))

</pre>

<img src="http://i.imgur.com/2tqJKCj.png"></img>

