---
layout: post_page
title: Running append backwards
---

I just watched [minikanren uncourse #6](https://www.youtube.com/watch?v=cILE9j9rNdY) today and Will Byrd showed something absolutely incredible.

The first time I was really amazed at Prolog was from the following (standard example):

```
append(X,[],X).
append([X|Xs],Y,[X|Zs]) :- append(Xs,Y,Zs).
```

it's just the standard recursive definition of `append/3`, and it does append lists but it also can split the 'output' list into two inputs!

```
?- append(X,Y,[a,b,c,d,e,f]).

X = [],
Y = [a, b, c, d, e, f] ;

X = [a],
Y = [b, c, d, e, f] ;

X = [a, b],
Y = [c, d, e, f] ;

X = [a, b, c],
Y = [d, e, f] ;

X = [a, b, c, d],
Y = [e, f] ;

X = [a, b, c, d, e],
Y = [f] ;

X = [a, b, c, d, e, f],
Y = [].
```

A bit of background first. In the last uncourse hangout we saw a relational intepreter generate quines (programs whose input and output is the same): Will Byrd showed how to turn a simple little lambda calculus interpreter into a kanren relation. This time we added some operators cons, car, cdr, if to it and then...

It's kind of impressive that the relational intepreter in minikanren can handle Y combinator based recursive functions like append:

```
(define (my-append-call x y)
  `((((lambda (f)
        ((lambda (x)
           (f (x x)))
         (lambda (x)
           (lambda (y) ((f (x x)) y)))))
      (lambda (my-append)
        (lambda (l)
          (lambda (s)
            (if (null? l)
                s
                (cons (car l) ((my-append (cdr l)) s)))))))
     (quote ,x))
    (quote ,y)))
```

but what really stunned me was that minikanren is able to run the scheme append function backwards:

```
> (for-each print (run* (q) (fresh (x y) (== q (list x y)) (eval-expo (my-append-call x y) '() '(a b c d e f)))))
(() (a b c d e f))
((a) (b c d e f))
((a b) (c d e f))
((a b c) (d e f))
((a b c d) (e f))
((a b c d e) (f))
((a b c d e f) ())

```

The code (edited a little bit to work in chicken scheme) is [here](https://github.com/orchid-hybrid/miniKanren-uncourse/blob/master/append.scm).
