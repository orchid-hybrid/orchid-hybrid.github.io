---
layout: post
title: Flattening Trees
---

Introduction
===========

I've been reading the final chapter of SICP which covers register machines (this is like assembly/machine code), in particular they reimplement the explicit control evaluator as a register machine and then use those ideas to create a lisp compiler. In order to create my own compiler I'm planning on following alongn with SICP but using x86 rather than the register machine simulator from the book. On the other hand x86 programming is scary and I'm not very good at it!

One of the important parts of running lisp in a register machine is "lisp structured memory", the ability to cons, car and cdr. So for practice I've written a trivial compiler that takes a tree and builds a sequence of instructions that build it on the stack - then implement the following recursive function in x86:

    (define (print-tree tree)
      (if (number? tree)
          (display tree)
          (begin (display "(")
                 (print-tree (car tree))
                 (display " . ")
                 (print-tree (cdr tree))
                 (display ")"))))

Here's an example of running this:

    #;> (print-tree '((#x66 . #x77) . (#x8989 . #x4141)))
    ((102 . 119) . (35209 . 16705))

To flatten a tree of cons cells onto the stack we will need a representation that uses pointers. Number are represented as a single tagged qword (64 bits), and conses are a two qwords (the car and cdr pointers), the car is tagged. Tagging is needed to tell what type a piece of data on the stack is. We reserve the four lowest bits of a qword to use for the tag. 0000 for a cons and 0001 for a number.

To tag a number in assembly you need to bitshift it left 4 bits then or it with 0b0001, for example if our number was 1945

    mov rax,1945
    shl rax,4
    or  rax,0b001

to test the tag you bitmask it using `and`, and to get the data out once you looked at the tag you can use `shr`.

Here's an example of allocating the following structure on the stack: `(#x66 . #x77)`

    address         value
    0x7fffffffe618: 0x0007fffffffe6300
    0x7fffffffe620: 0x00007fffffffe628
    0x7fffffffe628: 0x0000000000000771
    0x7fffffffe630: 0x0000000000000661

the two numbers (tagged with a 1 nybble) are at the bottom of the stack because they were allocated first (note that the stack grows by subtracting the stack pointer so the address are lower the higher up on the stack you are). At the top there is a 0 tagged pointer, and then an untagged pointer - these two qwords represent a cons cell.

Here is the source for the little compiler that takes a tree and allocates it onto the stack:

    (define base-pointer
      ;; this counter is used to keep track of where we
      ;; are, relative to the base-pointer (that points
      ;; to the bottom of the stack frame)
      (let ((rsp 0))
        (lambda ()
          (set! rsp (+ rsp 8))
          rsp)))
    
    (define (push-number n)
      ;; to allocate a number just tag it and push it onto the stack
      ;; return its location relative to the base pointer
      (emit `(mov rax ,n))
      (emit `(shl rax 4))
      (emit `(or rax 0b0001)) ; number tag is 0b0001
      (emit `(push rax))
      (base-pointer))
    
    (define (push-cons kar kdr)
      ;; to allocate a cons cell, gives the addresses of its car and cdr
      ;; we push the cdr, then push the tagged cdr - then increment the
      ;; stack pointer and return that value
      (emit `(mov rax rbp))
      (emit `(sub rax ,kdr))
      (emit `(push rax))
      (emit `(mov rax rbp))
      (emit `(sub rax ,kar))
      (emit `(shl rax 4))
      ;(emit `(or rax 0b000)) ; cons tag is 0, no need to or anything
      (emit `(push rax))
      (base-pointer)
      (base-pointer))
    
    (define (compile-tree tree)
      (if (number? tree)
          (push-number tree)
          (begin
            (let* ((kar (compile-tree (car tree)))
                   (kdr (compile-tree (cdr tree))))
              (push-cons kar kdr)))))

Here's the code it generates for the following invocation

    (let ((root (compile-tree '(#x66 . #x77))))
      (emit `(mov rax rbp))
      (emit `(sub rax ,root)))

produces

        mov rax,102
        shl rax,4
        or rax,0b0001
        push rax
        mov rax,119
        shl rax,4
        or rax,0b0001
        push rax
        mov rax,rbp
        sub rax,16
        push rax
        mov rax,rbp
        sub rax,8
        shl rax,4
        push rax
        mov rax,rbp
        sub rax,32

and that creates a stack just like we saw before. And finally here is the print-tree procedure which recursively walks over the tree printing it out:

    print_tree:
            mov     rbx,[rax]
            mov     rcx,rbx
            and     rbx,0xF
            test    rbx,rbx
    
            jnz     .number
    
            ;; cons
    
            mov     rbx,rcx
            mov     rcx,[rax+8]
            push    rcx
            push    rbx
            
            mov     rax,1
            mov     rdi,1
            mov     rsi,open_str
            mov     rdx,1
            syscall
    
            pop     rax
            shr     rax,4
            call    print_tree
    
            mov     rax,1
            mov     rdi,1
            mov     rsi,dot_str
            mov     rdx,3
            syscall
    
            pop     rax
            call    print_tree
    
            mov     rax,1
            mov     rdi,1
            mov     rsi,close_str
            mov     rdx,1
            syscall
    
            ret
    
    .number:
            mov     rax,rcx
            shr     rax,4
            call    print_number
    
            ret

To do all this it was really useful to be able to debug my code in GDB, in particular the following commands:

    break tree.asm:61 # to break at line 61
    info registers    # to see what the register values are
    ni                # next instruction
    layout asm        # to see the assembly code in gdb
    x/64xg $rbp-64    # to e'x'amine the top few 'g'iant qwords on the stack

