---
layout: post
title: Testing Code
---

This post is just a test of source code formatting. OK, So source code formatting works fine inside github but not inside the actual blog.. not sure how to sort that.

Here is a useful thing called the "Fresh" monad:

```haskell
{-# LANGUAGE GeneralizedNewtypeDeriving #-}

module Fresh (
 Fresh(..), runFresh, fresh
 ) where

import Control.Monad.State

newtype Fresh s a = Fresh { unFresh :: State [s] a }
 deriving Monad

runFresh m s = evalState (unFresh m) s

fresh = do (x:xs) <- Fresh get ; Fresh (put xs) ; return x

```

you give it an infinite stream of distinct values and it lets you get out a fresh one every single time.

Just for fun here is the same sort of thing in the Eff language:

```
type fresh = effect
 operation gensym : unit -> int
end

let run s = handler
 | val y -> (fun s -> y)
 | s#gensym () k -> (fun s -> k s (s+1))
 | finally f -> f 0;;

let g = new fresh in with run g handle
    g#gensym () :: g#gensym () :: g#gensym () :: [] ;;
```
