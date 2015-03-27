---
layout: post
title: Breaking Lambdabot
---

GHC - the glasgow haskell compiler - just hit [10000](https://ghc.haskell.org/trac/ghc/ticket/10000) tickets! And this 10 thousandth ticket is a very interesting one!

It's a vulnerability in haskell that resulted in lambdabot (An IRC bot which evaluates haskell code using the mueval library) being taken down for safety for a whole day. It's all been patched now, don't worry!

What is the vulnerability exactly and how could it be exploited? The scenario is that the IRC bot tries to only execute safe code, but we want to escalate it to execute (potentially dangerous) IO code.

Note: The exploit was created by Ørjan Johansen. I'm just doing a writeup because I had to dig pretty deep to understand it, I hope my notes help others understand this unusual kind of exploit too. I'm trying to make it that you don't need to know haskell to read this post (except for one section).

# Type safety and IO

(A) The Haskell type system is a static analysis done on programs before running them to try to signal bugs and stop harmful things from happening. The primary example of a type safety violation is interpreting one object as another.

(B) Another aspect of the type system is to separate programs which do IO (reading files, writing files, network access, spawning shell commands, etc.) from those which only do pure computations (like reversing a string, generating permutations of a list, flattening a tree - the sort of things which lambdabot is supposed to do).

The bug discovered was a very slight violation where GHC thought two different types were actually equal. It took a lot of cleverness to turn this into a completely generate coerce function (that can trick the compiler into thinking anything is anything else).

# Unsafety and SafeHaskell

But who needs a type safety bug! Haskell gives you `unsafeCoerce` which lets you interpret an object of one type as an object of any other type (A) and `unsafePerformIO` which lets you turn a program with IO type into a program that looks like it's pure (B) for free!

Why can't we just send lambdabot `unsafePerformIO (cat "/etc/passwd")`? It turns out someone already thought of this and there is a whole system of tagging code as Safe or Unsafe, so if we only import safe code we shouldn't be break the bot.

# Going from a bug in the type system to unsafeCoerce

Let's look at the code from bug itself:

* https://ghc.haskell.org/trac/ghc/ticket/10000
* https://ghc.haskell.org/trac/ghc/ticket/9858 (related issue)

The real core of this exploit is very technical and specific to some advanced type system features. Let's just look at what it enables someone to do:

```
uc :: a -> b
```

By taking advantage of the type system bug he was able to implement the `unsafeCoerce` function using only SafeHaskell so GHC considers it safe - turning that bug into a type safety vulnerability.

This is bad because (A) can easily be used to cause crashes but we don't exactly have IO code execution yet (that could be used to take control of the computer the computer or VM the bot runs in), we need to implement `unsafePerformIO` using `unsafeCoerce`. It turns out this is well known to the right people - so the danger was understood at this point by people familiar with GHC internals.

# From A to B

## More on UnsafeCoerce

First off I want to go over unsafeCoerce and its relation to unsafePerformIO a little more. I installed GHC and experimented with it a bit:

In haskell every value has a type and everything is supposed to strictly adhere to this type system.. but someone added an escape hatch where you can cast one type of value into any other, similar to casting a value from one type to another in (invalid) C: `*((foo*)&p)`.

```
> :m + Unsafe.Coerce
> unsafeCoerce True :: Int
-4611685468992169172
> unsafeCoerce False :: Int
140655415996216
> unsafeCoerce (-4611685468992169172 :: Int) :: Bool
False
> unsafeCoerce "abc" :: Int
4035225815559036378
```

and `unsafeCoerce "foo" :: Integer` just about took my computer down before an out of memory exception.

## unsafeCoerce from unsafePerformIO

The following code by Jonathan Cast shows how to implement unsafeCoerce using unsafePerformIO:

```
unsafeCoerce :: a -> b
unsafeCoerce a = unsafePerformIO $ do
  let ref = unsafePerformIO $ newIORef undefined
  ref `writeIORef` a
  readIORef ref
```

The mutable cell `ref` has a polymorphic type. This code write in a value at one type and reads it out with another. The <a href="http://mlton.org/ValueRestriction">value restriction</a> is what keeps ML safe from this sort of thing.

* https://www.mail-archive.com/haskell-cafe@haskell.org/msg30573.html
* http://stackoverflow.com/questions/22114757/unsafecoerce-implementation-in-haskell

This shows that different kinds of unsafe functions are related in some way, but to get RCE we really need to go from unsafeCoerce to unsafePerformIO:

## unsafePerformIO to unsafeCoerce

This subsection is more technical so I summarize it at the end if you want to skip.

I looked into how GHC implemented the "IO monad" which talks about the representation and implementation of IO.

* http://git.haskell.org/packages/base.git/blob/HEAD:/GHC/IO.hs

In particuar it's just an instance of the ST monad. Which we can test:

```
Prelude> :m + Control.Monad.ST Unsafe.Coerce
Prelude Control.Monad.ST Unsafe.Coerce> runST (unsafeCoerce (do print "hi" ; return 3) :: ST s Int)
"hi"
3
```

An ST s a is just a newtype wrapper `for s -> (# s, a #)`. That means in terms of implementation they are the same: newtypes don't exist as runtime - they're only used to help make type errors when programmers make mistakes. We can unsafeCoerce from one to the other.

With that in mind here's my attempt:

```
{-# LANGUAGE RankNTypes, MagicHash, UnboxedTuples #-}

-- :set -fobject-code

import Unsafe.Coerce

performIO m = b (a m ()) where
 a :: IO a -> rw -> (# rw, a #)
 a = unsafeCoerce
 b :: (# rw, a #) -> a
 b (# _, x #) = x
```

and it works.. but you get a segfault afterwards, I'm not sure why:

```
Prelude> :l Play
[1 of 1] Compiling Main             ( Play.hs, interpreted )
Error: bytecode compiler can't handle unboxed tuples.
  Possibly due to foreign import/export decls in source.
  Workaround: use -fobject-code, or compile this module to .o separately.
> :set -fobject-code
> :l Play
[1 of 1] Compiling Main             ( Play.hs, Play.o )
Ok, modules loaded: Main.
Prelude Main> performIO (do print "hello world!" ; getLine)
""hello world!"
^CSegmentation fault (core dumped)
```

Notice the two double quotes at the start of the string? It is attempting to return the value before the computation has finished.

I searched a bit and discovered someone else (credit to int-e) has implemented it in a different way. Using a really clever trick:

```
unsafePerformIO :: IO a -> a
unsafePerformIO a =
    let {-# NOINLINE (>>>=) #-}
        infixl 1 >>>=
        (>>>=) :: IO a -> (a -> IO b) -> IO b
        (>>>=) = (>>=)
    in  unsafeCoerce (a >>>= unsafeCoerce . const) ()
```

The inner "unsafeCoerce" is used to turn `rw -> (# rw, a #)` into `() -> (# rw, a #)` and applying that gets you `a` somehow

It's more stable for some reason, but the really important thing is that while these are both SafeHaskell you could type this one into lambdabot .. you couldn't do that with my attempt.

* http://lpaste.net/74498
* http://ircbrowse.net/day/haskell/2013/09/01

# Conclusion

In summary: You can go from the `uc` function that coerces one object to another to a "SafeHaskell" version of unsafePerfomIO. Chaining this with the type safety violation gives you IO code execution on lambdabot and for this reason the bot was removed until this bug was fixed. Secondly - these implementations of unsafePerformIO are a little shakey, it may not fully execute your IO programs and it will often crash afterwards. I think the execution is reliable enough to be dangerous though.

What to take from this? (Trying to analyze the attack and see how we can improve security..)

If you like to think of "types as propositions" as values as proofs then you can see that an exploit literally is a constructive proof of some type of computation that you didn't expect (e.g. arbitrary coercion "a -> b" or making IO code look like pure code "IO a -> a").

There is a strong assumption that haskell code is type safe and it's relied on. I think the security of haskell programs would improve if uses of unsafe operators was replaced with safe versions of the common patterns they are used in.

Something to take more seriously in modern 'strongly typed' languages is type safety. We should try to produce progress and preservation proofs for any type system features we want to add to a language. Haskell and Rust do not do this but working out the mathematics like this could really help them improve their languages and design.

Related: Just recently [False was proved in Coq](https://github.com/clarus/falso) based on the vm_compute tactic. This does let you write `unsafeCoerce : forall (A : Set) (B : Set), A -> B` but I don't think you can get "code execution" from Coq using that it doesn't have an IO monad. What is really gives you is 'theory escalation' where you go from the (exepected) Coq theory to a stronger theory where everything can be proved.

And some related links:

* http://joyoftypes.blogspot.co.uk/2012/08/generalizednewtypederiving-is.html
* https://existentialtype.wordpress.com/2012/08/14/haskell-is-exceptionally-unsafe/
