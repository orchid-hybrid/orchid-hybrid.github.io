---
layout: post
title: Lisp OS - Mezzano
---

Out of nowhere a hacker named froggey dropped his Lisp OS project in [#lisp](http://tunes.org/~nef/logs/lisp/15.01.25) along with a virtualbox image so people could try it out it a virtual machine.

```
07:09:54 <froggey> hello, is this the right place for me to mention my Lisp OS?
07:10:08 <beach> Others mention theirs, so why not.
07:10:41 <froggey> oh good
07:10:46 --- quit: attila_lendvai (Quit: Leaving.)
07:11:13 <froggey> I've spent the past few years working on it, and I think it's about ready for other people to see
07:11:22 <beach> Wow.
07:11:29 <beach> You have been fairly quiet about it.
07:11:47 <froggey> I have
07:11:49 <froggey> here's a fairly recent screenshot: https://dl.dropboxusercontent.com/u/46753018/Screen%20Shot%202015-01-19%20at%2001.29.31.png
```

<img src="http://i.imgur.com/aSMaOam.png"></img>

Soon after he posted the source code: [https://github.com/froggey/Mezzano](https://github.com/froggey/Mezzano).

This is no small feat! He says he was hacking on this for the last five years, it's about 52k lines of code. A lot of people have dreamed about making a Lisp OS in the past (including myself) but I don't think anyone has done it, at least not since old lisp machine days. There have been a lot of hobby OS projects too and some are extremely good (for example TempleOS), Mezzano looks like a contender.

It has a file server and a basic TCP/IP stack that is used as part of the bootstrapping process. The Lisp OS contacts it via TCP and reads in all sorts of images fonts and data files as well as lisp code to compile.

It has its own common lisp implementation with compiler and runtime system, it may not be very optimized yet (froggey mentioned that it doesn't do register coloring) but it is able to compile many serious common lisp libraries like asdf, various image formats and of course the operating system itself.

and as you can see from the screenshot there is a graphical user interface desktop that you boot into with a lisp REPL, an IRC client, a rougelike game and telnet!

It's so inspiring to see people create amazing things like this. froggey was brave to spend that time working on this and to release the sources for people to play with and learn from.
