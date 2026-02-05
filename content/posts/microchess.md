+++
date = '2026-01-30T13:55:41+01:00'
draft = true
title = 'Microchess'
+++

# Improving Microchess Portability

Microchess is the first commercial chess engine made for the 6502 powered KIM-1. It is a brilliant and facinating piece of engineering that I honestly do not fully understand yet.

With that said, wouldn't it be nice if it can be executed on any 6502 machine ?

Disclaimer: A seasoned 6502 assembly dev might see this and say "What is this mess he brought upon this cursed land". I'm still learning and am not claiming to be an expert. I'm just sharing an experience I found interesting with some personal opinion as a blog post, I'm not making a wiki here. If you want to see (and hopefully improve) an actual wiki, checkout my [wiki](coming-soon)

My initial goal for this project was to port it for sim65 emulator to be able to run it locally as i found that the retro commuity mostly runs it on an emulator on the arduino (emulating on computer seems to be doable but more difficult).

The problem with the 6502 programs of the 70s is how tightly coupled a program is to the hardware it is running on. addresses for io ports, ram, rom, and other stuff need to be hardcoded. These days, we take the linker's job in a compilation for granted. A lot of people (me included) got into the tech world well after standards of these kinds were well standardised and dynamic linking is something most people don't even think about. in othrr words, gen-z devs' intuition says "if it's binary for that cpu architecture, why doesn't it work ?" which (for a dev from the 70s) is just stupid.

Luckily, several attempts at addressing this have been made. Some of which are truly surreal. LUNIX for C64 is an insane one that supports a filesystem and patches binaries in ram to make up for the lack of EMMU hardware required for dynamic linking. That my fellows is true art.

That (as cool as it sounds) is not what we'll be using. I might attempt this one day though.

we'll be using static linking instead, and the cc65 is the perfect toolchain for this. we can even use the C library that comes along.

Ever thought about how you'd hello world literally from scatch ? python/lua make this easy, you just `print("hello world")` where I come from, people take this for granted (one could argue that modern tech people are too oblivious these days, but I find that a bless rather than a curse). But even if you do on modern systems it's considered very standarised. You just put stuff in registers and use `call 0x00` to make a syscall, the system does the rest. on a microprocessor like the MOS 6502 you kinda need to know the hardware you're talking to even if you know the assembly for the processor. It is kinda like relearning/redefining your hello world every time you're working on a new computer brand. Ben eater makes a very nice video about this for his hardware btw.

The cc65 toolchain solves this in an elegant way and even supports writing in C for the 6502. having a C library and an abstraction layer for that mess is as awesome as it can get. Now "all we need is to swap out old coupled IO interactions with the new standard ones" one one say. welp, not so easy as we'll find out.
