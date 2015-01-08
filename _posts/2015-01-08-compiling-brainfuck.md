---
layout: post
title: Compiling brainfuck
---

Brainfuck is a (ridiculously) cut down language with only really one control flow construct: `[ ... ]` which at `[` is skipped **if** the current cell is zero, and at `]` loops **while** the current cell is nonzero.

It's trivial to compile it to C by just replacing each command with some C, e.g. `[` turns into `while(*p) {` and `]` into `}`. In fact that's what the self hosting brainfuck to C compiler I wrote does:

```
--[------->++<]>-.[->+++<]>.+++++.-----------.+++++++++.+++++++++.+[->+++<]>++.+.--[--->+<]>-.--[->++<]>.--[->++<]>-.+.++[->+++<]>++.+++++.++++++.[->+++++<]>+++.+[--->+<]>+++.++[->+++<]>.>++++++++++.-[->++++<]>-.[->+++<]>.+++++.-----------.+++++++++.+++++++++.+[->+++<]>++.+.--[--->+<]>-.--[->++<]>.--[->++<]>-.+.++[->+++<]>++.+++++.+++++.++++++.[++>---<]>.+[--->+<]>+++.++[->+++<]>.>++++++++++.[--------->++<]>+.------------.+++++.++++++.[-->+<]>--.+[--->+++++<]>.[--->+<]>-.[---->+<]>+++.[->+++<]>+.------.+[-->+<]>+++.-.++.++.[------>+<]>-.[--->++<]>-.[->++<]>+.-[-->+++++<]>-.>--[-->+++<]>.-[++>---<]>+.[->++<]>-.------------.+++++.++++++.[-->+<]>--.+[--->+++++<]>.[--->+<]>-.--[++>---<]>-.--[--->++<]>.[-->+<]>+++++.+++[-->+++<]>+.+[------>+<]>.----[->++<]>-.------------.++++++++.+++++.++[++>---<]>.+.[->+++<]>.--------.++++[->+++<]>.[--->+<]>---.++[->+++<]>.[--->+<]>-.++[->+++<]>+.---[->+++<]>-.--[->+++<]>+.+.++[->+++<]>++.+++++++++++.++++++.-.[++>---<]>--.-----[->++<]>.+++++++.---------..[-->+<]>+++.-[-->+++<]>-.>+[>[-],>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+>[-]+<<<<<<<<<----------[>-<---------------------------------[>>-<<-[>>>-<<<-[>>>>-<<<<-[>>>>>-<<<<<--------------[>>>>>>-<<<<<<--[>>>>>>>-<<<<<<<-----------------------------[>>>>>>>>-<<<<<<<<--[>>>>>>>>>-<<<<<<<<<[-]]]]]]]]]]>[<<->>>>>>>>>>><[-]<[-]<[-]<[-]<[-]<[-]<[-]<[-]<[-]]<>>[>>>>>>>>++[------>+<]>..-.--[--->++<]>.[-->+<]>+++.<<<<[-]<[-]<[-]<[-]<[-]<[-]<[-]<[-]]<<>>>[>>>>>>>++[------>+<]>-.--[--->++<]>.[-->+<]>+++++.--[--->+<]>--.--.[--->+<]>---.++[->+++<]>+.+++++.-------.--[--->+<]>---.[--->+<]>++.+.-[-->+++<]>-.<<<<<<<<<<[-]<[-]<[-]<[-]<[-]<[-]<[-]]<<<>>>>[>>>>>>++[------>+<]>++..---.--[--->++<]>.[-->+<]>+++.<<<<[-]<[-]<[-]<[-]<[-]<[-]]<<<<>>>>>[>>>>>+[------->++<]>++.+++++.-.++[->+++<]>+.+++++.-------.--[--->+<]>---.[--->+<]>++.++.--[--->++<]>.[++>---<]>+.-[-->+++<]>-.<<<<<<<<[-]<[-]<[-]<[-]<[-]]<<<<<>>>>>>[>>>>++[------>+<]>++..--[----->+<]>+.[-->+<]>+++.<<<<[-]<[-]<[-]<[-]]<<<<<<>>>>>>>[>>>++[------>+<]>..[----->+<]>+.[-->+<]>+++.<<<<[-]<[-]<[-]]<<<<<<<>>>>>>>>[>>------[-->+++<]>.+[->+++<]>.+.+++.-------.--[->+++<]>-.++.--[--->++<]>.[++>---<]>+.[->+++<]>.<<<<<<<[-]<[-]]<<<<<<<<>>>>>>>>>[>--[-->+++<]>.<<[-]]<<<<<<<<<>>>>>>>>>>[-]>[-]>[-]>[-]>[-]>[-]>[-]>[-]>[-]>[-]><<<<<<<<<<<<<<<<<<<<<]--[-->+++<]>.>++++++++++.
```

This is sort of 'cheating' though or at least it's worth pointing out that we're piggybacking on C to avoid doing any real work: C handles the bracket matching for you.

A similar thing happens when you write an interpreter for a high level language in another high level language. You usually inherit things like evaluation order (unless you make that explicit using continuation passing) and you can piggy back on the host languages garbage collection.

So I decided to write a brainfuck compiler to the UM virtual machine in Ocaml (because I want to learn Ocaml better), because its instruction set is very low level I can't cheat: it doesn't have any bracketing or labels.

I forgot about the if aspect of `[ ... ]` and wrote most of the code, so I decided to leave that out: It's not proper brainfuck because of this but can still run some brainfuck programs.

So looking at the UM spec: http://boundvariable.org/um-spec.txt I made a sort of "assembler" that emits the binary code for each instruction:

```ocaml

open Core.Std

(* 8 registers *)

let orthography r v =
  ((0b1111 land 13) lsl (32-4)) lor
    ((0b111 land r) lsl (32-4-3)) lor
      (0x1FFFFFF land v)

let prepare op a b c =
  ((0b1111 land op) lsl (32-4)) lor
    ((0b111 land a) lsl 6) lor
      ((0b111 land b) lsl 3) lor
	(0b111 land c)
	  
let conditional_move a b c = prepare 0 a b c
let array_index a b c = prepare 1 a b c
let array_amendment a b c = prepare 2 a b c
let add a b c = prepare 3 a b c
let multiply a b c = prepare 4 a b c
let division a b c = prepare 5 a b c
let notand a b c = prepare 6 a b c

let halt = prepare 7 0 0 0
let allocation b c = prepare 8 0 b c
let abandonment c = prepare 9 0 0 c
let output c = prepare 10 0 0 c
			     
let load b c = prepare 12 0 b c
			     

let output_word ch wd =
  Out_channel.output_byte ch (0xFF land (wd lsr 24));
  Out_channel.output_byte ch (0xFF land (wd lsr 16));
  Out_channel.output_byte ch (0xFF land (wd lsr 8));
  Out_channel.output_byte ch (0xFF land (wd lsr 0))

```

Now I can use that module to write a little program that emits a .umz binary file for a brainfuck program.

```ocaml
open Core.Std

module Um = Emit
       
(* To compile: corebuild brain.byte *)
       
let position = ref 0
let places = Stack.create ()

let swap_in c = Um.output_word c (Um.array_amendment 1 3 0)
let swap_out c = Um.output_word c (Um.array_index 0 1 3)
let process_char c ch =
  match ch with
  | '+' -> Um.output_word c (Um.add 0 0 1); 1
  | '-' -> Um.output_word c (Um.add 0 0 2); 1
  | '>' -> swap_in c; Um.output_word c (Um.add 3 3 1); swap_out c; 3
  | '<' -> swap_in c; Um.output_word c (Um.add 3 3 2); swap_out c; 3
  | '.' -> Um.output_word c (Um.output 0); 1
  | '[' -> Stack.push places (!position); 0
  | ']' -> (match Stack.pop places with
	    | Some jump ->
	       (Um.output_word c (Um.orthography 6 (!position + 4));
		Um.output_word c (Um.orthography 5 jump); 
		Um.output_word c (Um.conditional_move 6 5 0);
		Um.output_word c (Um.load 4 6);
		4)
	    | None -> (* ERROR *) 0)
  | _ -> 0
	   
let () =
  let c = Out_channel.create "emit.umz" ~binary:true in
  
  (* set up the constants 1 [reg 1] and -1 [reg 2] *)
  Um.output_word c (Um.orthography 1 0x01);
  Um.output_word c (Um.orthography 2 0x10001);
  Um.output_word c (Um.orthography 3 0x0FFFF);
  Um.output_word c (Um.multiply 2 2 3);
  
  (* allocate a platter with lots of space for the cells *)
  Um.output_word c (Um.allocation 1 3);
  
  (* set register 3 to the start of that array *)
  Um.output_word c (Um.orthography 3 0);

  position := 6;
  
  let rec loop () =
    match In_channel.input_char In_channel.stdin with
    | None -> ()
    | Some ch -> position := !position + process_char c ch; loop ()
  in loop ()
```

The first thing it set up some registers with values like +1 and -1 inside them as well as allocating memory for the brainfuck tape. After that it emits code while reading input: This wouldn't be possible if I handled `[` correctly since knowing where to jump to (the corresponding `]`) you'd need to do this in two passes.

The main mechanic of this compiler is that it holds a stack of ccode positions of each open bracket, when it sees a close bracket it pops one off so it knows where to jump to.

My original idea was to make this a self hosting compiler to improve upon my ->C one, I wrote it in Ocaml to prototype it - I'm lucky I did this because of the mistake about missing out the if aspect there's no way this could self host. A different compilation strategy is required: for example an intermediate assembly language with labels would be perfect.

