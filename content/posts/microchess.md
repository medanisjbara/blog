+++
date = '2026-01-30T13:55:41+01:00'
draft = false
title = 'Microchess'
+++

# Improving Microchess Portability

Microchess was the first commercial chess engine for the 6502-powered KIM-1. It is a brilliant and facinating piece of engineering that I honestly do not fully understand yet.

With that said, wouldn't it be nice if it can be executed on any 6502 machine ?

Disclaimer: A seasoned 6502 assembly dev might see this and say "What is this filth he brought upon this cursed land". I'm still learning and am not claiming to be an expert. I'm just sharing an experience I found interesting with some personal opinion as a blog post, I'm not making a wiki here. If you want to see (and hopefully improve) an actual wiki, checkout my [wiki](coming-soon)

My initial goal for this project was to port it for sim65 emulator to be able to run it locally as i found that the retro commuity mostly runs it on an emulator on the arduino (emulating on computer seems to be doable but more difficult).

The problem with the 6502 programs of the 70s is how tightly coupled a program is to the hardware it is running on. addresses for io ports, ram, rom, and other stuff need to be hardcoded. These days, we take the linker's job in a compilation for granted. A lot of people (me included) got into the tech world well after standards of these kinds were well standardised and dynamic linking is something most people don't even think about. in othrr words, gen-z devs' intuition says "if it's binary for that cpu architecture, why doesn't it work ?" which (for a dev from the 70s) is just stupid.

Luckily, several attempts at addressing this have been made. Some of which are truly surreal. LUNIX for C64 is an insane one that supports a filesystem and patches binaries in ram to make up for the lack of EMMU hardware required for dynamic linking. That my fellows is true art.

That (as cool as it sounds) is not what we'll be using. I might attempt this one day though.

we'll be using static linking instead, and the cc65 is the perfect toolchain for this. we can even use the C library that comes along.

Ever thought about how you'd hello world literally from scatch ? python/lua make this easy, you just `print("hello world")` where I come from, people take this for granted (one could argue that modern tech people are too oblivious these days, but I find that a bless rather than a curse). But even if you do on modern systems it's considered very standarised. You just put stuff in registers and use `call 0x00` to make a syscall, the system does the rest. on a microprocessor like the MOS 6502 you kinda need to know the hardware you're talking to even if you know the assembly for the processor. It is kinda like relearning/redefining your hello world every time you're working on a new computer brand. Ben eater makes a very nice video about this for his hardware btw.

The cc65 toolchain solves this in an elegant way and even supports writing in C for the 6502. having a C library and an abstraction layer for that mess is as awesome as it can get. Now "all we need is to swap out old coupled IO interactions with the new standard ones" one one say. welp, not so easy as we'll find out.

I believe the first thing anyone would do when starting out is to take the raw code and try to run it through the assembler `ca65` so it can spit out an error we can look at to know where to start.
```
microchess.s(71): Error: Unexpected trailing garbage characters
microchess.s(73): Error: ':' expected
microchess.s(84): Error: ':' expected
microchess.s(88): Error: ':' expected
```
Apparently every label in the original doesn't end with a column and `ca65` expects all assembly labels to have a column at the end. That's an easy one. Step 1 complete.

```
microchess.s(71): Error: Unexpected trailing garbage characters
```

But then comes those weird `*=$1000` which seems to be there to force the binary into specific RAM locations, (the thing that people these days don't think about cuz we've got ELF and EMMU to cover us). And the assembler complains that it doesn't even know what those are and calls them garbage (and trust me, that's a good thing, even if it means more work for us). The cc65 toolchain solves this in an elegant way. You define `.segment` for different segments of the binary. And since each machine is going to have different RAM addresses available (remember that on a 6502 computer, RAM is just an address space, and so is ROM and other stuff, It's not like you can roam around free in the address space and put whatever you want wherever you want). the cc65 toolchaine defines config files for each machine. Ben Eater's videos do dig deep into the details of these and they are really awesome, but since we're not making a new computer, we don't really care about their content or the "how to write one yourself", in fact, a cool byproduct of trying to port this the right way is the fact that we will get a microchess codebase that works on every supported computer that has a cfg file in the toolchaine.

On nixos, you can get a list of all supported machines with this command `ls -1 $(nix-build -A cc65 --no-out-link '<nixpkgs>')/share/cc65/cfg | xargs -n 1 basename -s .cfg` and if you happened to try it. yes, we ported microchess from being something that works on one 6502 based machine to something that works on 63 more machines now (assuming this works). But my goal is still to get this to work on sim65, while making it compile for one definitively means it'll compile for the others (and likely work fine). I haven't tested any 6502 machine other than sim65 (not even sim65c02 which btw should be identical since 65c02 is just a 6502 with some added instructions to make devs' lives easier).


With that said, it's time to get to work.
```diff
@@ -68,20 +71,22 @@
 prompt		= $FC			; prompt character, '?' or ' '
 reverse		= $FD			; which way round is the display board
 
-		*=$1000			; load into RAM @ $1000 onwards
-
-CHESS:
+.segment    "CODE"
+.proc _main: near
 	CLD				; INITIALIZE
 	LDX	#$FF			; TWO STACKS
-	TXS	
+	TXS
 	LDX	#$C8
 	STX	SP2
+	JMP	OUT
+.endproc
 
 ;	ROUTINES TO LIGHT LED
 ;	DISPLAY AND GET KEY
 ;	FROM KEYBOARD
 
 OUT:
+CHESS:
 	JSR	DrawBoard		; draw board
 	LDA	#'?'			; prompt character
 	STA	prompt			; save it
```
I've moved CHESS label to out (so they're now the same thing) and defined a `_main` instead of CHESS. The toolchain needs this to know what to call. And other than CODE segment, there's the RODATA segment.
```diff
@@ -1060,6 +1087,8 @@
 
 ; text descriptions for the byte in DIS1
 
+.segment "RODATA"
+
 PieceName:
 	.byte	"King    "
 	.byte	"Queen   "
```

All this should feel pretty standard if you've worked with the cc65 or with modern assembly (kinda, not the .segment syntax but everyone who ever used `objdump` knows those segments exist in a binary). But to the uninitiated, I still strongly advise you checkout Ben Eater's videos. Particularly the one where he played around with porting MSBasic to his machine.


AND VOILA!! Just like that, the assembler is happy. No errors. No "garbage" warnings.

**But here’s the catch:** A program that compiles is just a program that hasn't crashed *yet*. The compiler is no longer babysitting us, but the IO is still broken. Microchess thinks it's talking to a KIM-1 teletype, but `sim65` is waiting for standard POSIX-style input.

**To be continued...** Next week, in Part 2, we tackle the real headache: mapping 50-year-old IO routines to a modern terminal.
