---
layout: post
title: A correctness proof in Coq
---

This post is about a function to test if two sorted lists of numbers have any overlap.

If you have unsorted lists it's quite inefficient since you have to look through the whole list to know if an element is in there. If you have two sorted lists you can walk down them both together and it's O(n) instead of O(n^2).

I thought it would be a good exercise to write a formal specification for this function and implement and prove it correct in Coq.

First I defined a predicate `P` that holds when two lists contain the same element.

```
Definition P {A} (u v : list A) := exists x, In x u /\ In x v.
```

Then I defined a specification for the function I want:

```
Definition Spec (f : list nat -> list nat -> bool) :=
  forall u v, StronglySorted (lt) u -> StronglySorted (lt) v ->
   if f u v then P u v else ~ P u v.
```

So `Spec f` states that, applied to sorted lists `u` and `v`, if `f u v` is true then the lists meet, otherwise they don't.

Next to implement the function itself, this is a bit tricky in a total language like Coq!

```
Require Import Recdef.

Definition totallen (uv : list nat * list nat) := length (fst uv) + length (snd uv).

Function intp (uv : list nat * list nat) {measure totallen} : bool :=
  match fst uv with
    | nil => false
    | x :: xs =>
      match snd uv with
        | nil => false
        | y :: ys =>
          match lt_eq_lt_dec x y with
            | inleft (left _) (* x < y *) => intp (xs, y :: ys)
            | inleft (right _) (* x = y *) => true
            | inright _ (* x > y *) => intp (x :: xs, ys)
          end
      end
  end.

unfold totallen; intros; destruct uv; simpl in *; subst; simpl.
omega.

unfold totallen; intros; destruct uv; simpl in *; subst; simpl.
omega.
Defined.
```

Overall it should be readable as a normal recursive function (given the knowledge that `lt_eq_lt_dec` checks if `x < y`, `x = y` or `x > y`). There's a few subtle things in it though, which I had to change around to get it all right: For example it takes a pair of lists rather than two parameters, I had to change it to be like that to get it accepted by the system.

The most important point is that the recursion pattern is a little bit complex, it takes two lists and when it does a recursive call one of the lists is smaller - but we don't know which. Coq accepts recursive functions where one known parameter always gets smaller. To be able to define this function I had to invent measurement that always gets smaller: The sum of the two lists. That's what `{measure totallen}` is about.

To be able to define a function isn't enough, you also need to be able to do a proof by induction - with just the right induction scheme - to be able to show the function is correct. A wonderful thing about defining a function using Coqs 'Function' command is that it tries to create induction schemes for you that fit the recursion pattern of the function. Without that special induction scheme you wouldn't be able to prove much about the function.

So I started the proof like this:

<code>
Theorem intp_Spec : Spec' intp.
Proof.
  unfold Spec'.

  intros uv.

  apply (intp_ind (fun uv b => StronglySorted lt (fst uv) -> StronglySorted lt (snd uv) -> if b then P (fst uv) (snd uv) else ~ P (fst uv) (snd uv)))
  ...
  ...
```

and it gives you one case that you have to prove for each line of the function, in the recursive ones you get an induction hypothesis that you need to make use of. The proof was very easy, just shuffling around lemmas and fitting them together, and once it was completed you get a nice Ocaml version of the function which you know works on sorted inputs!

```
Qed.

Extraction Language Ocaml.

Extraction intp.

(****

(** val intp : (nat list, nat list) prod -> bool **)

let rec intp uv =
  match fst uv with
  | Nil -> False
  | Cons (x, xs) ->
    (match snd uv with
     | Nil -> False
     | Cons (y, ys) ->
       (match lt_eq_lt_dec x y with
        | Inleft s ->
          (match s with
           | Left -> intp (Pair (xs, (Cons (y, ys))))
           | Right -> True)
        | Inright -> intp (Pair ((Cons (x, xs)), ys))))


 ***)
```
