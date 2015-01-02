---
layout: post_page
title: goto, considered
---

"If you need to go to somewhere, goto is the way to go" - Ken Thompson

For a long time I was strongly against goto. I felt like it was a bad language construct because it wasn't "compositional", you can't really break the program down into blocks which each have meaning on their own. So it makes reasoning about programs harder. I don't really feel like this anymore and it took me forever to really learn much on the topic since most discussions about it are unproductive.

## EWD 215: A Case against the GO TO Statement

* https://www.cs.utexas.edu/users/EWD/transcriptions/EWD02xx/EWD215.html
* https://www.cs.utexas.edu/users/EWD/ewd02xx/EWD215.PDF

In his paper Dijkstra talks about if/then/else, procedures and while - and in particular how complex they make expressing the current place in a program by means of an static index. It is key that the index is completely static, independent of the dynamic execution of the program because he starts out the paper saying that we are geared towards dealing with static relations and aren't as good at visualizing evolving processing. He also mentions the mathematical ideas that apply to reasoning about them (you'd use induction to deal with a while loop for example).

After this he explains that goto is a problem because you cannot have meaningful coordinates to mark the place in the program statically. You can do so dynamically but that's not what we're after. He adds that the statement is too primitive and makes a mess of your program.

There was a very good blog post about Dijkstra's opposition to "break" too, here: http://blog.plover.com/prog/Hoare-logic.html although I think this quote is important (he calls break an abortion clause I think):

> whatever clauses are suggested (e.g. abortion clauses) they should satisfy the requirement that a programmer independent coordinate system can be maintained to describe the process in a helpful and manageable way.

## Structured Programming Theorem

* http://en.wikipedia.org/wiki/Structured_program_theorem

There is this thing called the structured programming theorem which seems much more important than the attention it gets. Well it says that all you need to be turing complete is sequences, if/then/else and while loops (it seems to omit that you need some kind of unbounded data structure like arrays but whatever). The really interesting part to me is the Kosaraju hierarchy.

So when I was younger people would be using goto or arguing for it and I'd say you don't need it, it's no good. They'd show me some code from the linux kernel or whatever that uses goto for error handling and I'd rewrite it without goto (usually using procedures) and then really wonder why people would defend goto - I never understood it since I could always rewrite code to not use it without much difficulty. Looks like Dijkstra has had a similar experience too: https://www.cs.utexas.edu/users/EWD/transcriptions/EWD10xx/EWD1009.html

The Kosaraju hierarchy is interesting because it explains more about rewriting a program to not use goto. He constructed programs that are really hard to rewrite without using goto. In doing so you have to introduce new variables or do code duplication, and the programs always grow in size!

Now an extra point I want to raise, which isn't related to this post.. but: It's interesting that both languages are turing complete but the stricter one (no gotos) can't write some of the programs in as short a space (well.. you could go ahead and write an interpreter for a language with gotos if it was really really high on the Kosaraju hierarchy but that's cheating). Also I think this part of th reason that brainfuck is so difficult to program in, the wikipedia page links to P'' which is the same as brainfuck.

* http://soclab.ee.ntu.edu.tw/~soclab/course/Milestones_in_SW_Evolution/2-2-CACM-Nov-1975-p629-ledgard.pdf

![](http://i.imgur.com/ZztaBtJ.png)

I thought that this would change my mind finally about the goto thing and it did a bit but one of the conclusions the paper draws is that "From the programmer's viewpoint, theoretical results based on the conversion of one program form to another under restrictive conditions may not be of practical significance". This is a really good point to raise about the difference between theory and practice and how theoretical results actually apply to practice.

In summary, goto is something like as powerful being able to labelled your loops and do computed break/continue. That was an idea I had for a low level programming language instead of having jumps. I'm not sure what the benefit would be.

## Deriving programs, proving programs correct

* An Axiomatic Basis for Computer Programming - http://sunnyday.mit.edu/16.355/Hoare-CACM-69.pdf

My main reason for rejecting goto so much was my understanding of the Hoare logic approach of proving programs correct. Basically for each statement `S` you have `{ P } S { R }` which means that `P` is the precondition and `R` is the postcondition: Given that `P` holds and you do `S` then `R` will hold.

You have rules about doing this for various structured programming concepts and I've worked through some derivations and proofs from Dijkstras wonderful book A Discipline of Programming as well as from Hoare. I don't know how to do this when the language has goto (Although for the low low price of £35.64 Springer will let me read a paper by someone who figured it out!).

I believe that deriving programs and proving programs correct is very important. Part of why software is so hard to write and doesn't work that well when we do is that we don't prove it correct. On the other hand it is so difficult to prove programs correct that it's very rarely cost effective: I think it's done in train systems, flight systems, things like that.. but not much else. (It would be worth getting some facts on this)

All that said I don't prove programs correct much. I've tried to get into that in the past using Coq but that didn't work out well, and it applies to functional programs too - not imperative ones. So it's not really a good argument unless you're doing it in practice.

## Conclusion...

I have no idea what to think really. I think that the use of goto for error handling is bad and indicates that C is missing something important. It's best to be happy and laugh about things:

* https://github.com/igorw/retry/issues/3#issuecomment-56448334
* http://www.fortranlib.com/gotoless.htm "both the pro- and the anti-GOTO factions can agree. This construct is called the COME FROM statement"
