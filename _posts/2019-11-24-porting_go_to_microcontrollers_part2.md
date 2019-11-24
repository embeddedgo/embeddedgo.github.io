---
layout: post
title: Porting Go to microcontrollers (part 2)
tags: mcu go embeddedgo
---

![Gopher and noos lollipop]({{ site.baseur }}/images/gopher/gopher-noos.jpg)

<!--more-->

In the [previous part]({{ site.baseur }}/2019/11/19/porting_go_to_microcontrollers_part1.html) of this article I briefly described the process of porting Go language to a new architecture with little focus of my particular case: linux/thumb and noos/thumb ports. The description concerned tools related to the code generation: assembler, compiler, linker, disassembler. In this part I will deal with the second part of the Go language: the runtime. This part will be also less general because porting the runtime to a bare metal system differs much from porting it to any operating system.

My work on the noos/thumb port took place in two stages:

1. linux/arm &rarr; linux/thumb

2. linux/thumb &rarr; noos/thumb

The first stage touched the runtime slightly, mainly because of the Thumb bit in a function call address.

The second stage was mostly runtime because in case of GOOS=noos we don't have any operationg system but the runtime work is mostly based on the cooperation with the OS.

An operating system provides the Go runtime the following things: 

- memory,

- threads,

- synchronization,

- time.

In case of the noos target the runtime has to provide it all by itself.

#### Memory

The noos port introduces a simple memory allocator that implements the interface expected by runtime. It works on memory blocks described at link time in -M option.
The noos allocator reserves some memory space for persistent allocation need by runtime and gives the whole remaining part to the Go memory allocator as the initial heap arena.

The Go allocator and garbage collector remained almost unmodified. Their rich set of configuration parameters allowed to adapt them to the system with <1MB RAM. It is really a good piece of code that turned out to be scalable form fraction of a megabyte to many gigabytes. The set of parameters I have chosen is definitely not the optimal one, the allocator and GC not tested much, but the current effect is promising in particular when it comes to memory fragmentation which a headache in case of embedded systems.

There is a [test program](https://github.com/embeddedgo/stm32/blob/master/devboard/f4-discovery/examples/gctest/main.go) which I used to force the allocator and GC to do some work. It contains two gorutines that communicate over a channnel. The first one vigorously allocate random blocks of memory and send them to the second one. The second one simply receives the blocks and discards them. This simple program can work for days on the border of memory capacity (tested with GOMAXPROCS set to 1 and 2).

#### Threads

I had couple of ideas on how to address the problem of threads needed by the Go scheduler. The one was to don't implement them at all and schedule all gorutines on one thread. However, it turned out that the design of the whole runtime is so strongly dependent on the concept of the OS thread. It uses them heavily for so many things that the necessary changes would require enormous work with a vague effect.

The second idea was to implement simple thread scheduler, well separated from the runtime, who does everything a decent RTOS does. I have some experience in this topic so the first implementation was created fairly quickly and it turned out immediately how many things are redundant.

Eventually I implemented a thread scheduler that is tightly coupled to the Go scheduler. It provides a syscall interface required by Go scheduler but works directly on M structs and uses Go scheduler decisions to schedule M's on available cores (P's). If you don't know what these G, M, P mean read the [description](https://github.com/embeddedgo/go/blob/embedded/src/runtime/HACKING.md).

#### Synchronization

The thread scheduler provides futex like synchronization primitive. It allow to implement all runtime synchronization mechanisms but also allow to handle communication between interrupt handlers and goroutines (see [rtos.Note](https://github.com/embeddedgo/go/blob/embedded/src/embedded/rtos/note.go) type) which is essential for an efficient bare-metal programming.

#### Time

The Go runtime needs to measure time for some of its algorithms. The thread scheduler also requires time for its own needs. But none of them has a built-in time source. The user application is responsible for provide the time source for both. It does this at startup using the [rtos.SetSystemTimer](https://github.com/embeddedgo/go/blob/embedded/src/embedded/rtos/systim.go) function.

#### Summary

I'm aware that the description above is very cursory. If you would like to learn more, I encourage you to analyze the code and to ask questions on the [embeddedgo discussion group](https://groups.google.com/forum/#!forum/embeddedgo).