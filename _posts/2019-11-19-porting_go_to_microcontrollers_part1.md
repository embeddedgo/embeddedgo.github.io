---
layout: post
title: Porting Go to microcontrollers (part 1)
tags: mcu go embeddedgo
---

![Gopher and noos lollipop]({{ site.baseur }}/images/gopher/gopher-noos.jpg)

<!--more-->

In this article, I would like to briefly describe the process of porting Go to a new architecture, along with some details about my port to the bare metal ARMv7-M ISA.

#### Starting point

The starting point is easy but requires a lot of work. Make Go recognize your new GOOS/GOARCH (in my case linux/thumb) and implement it using some existing port (in my case linux/arm). At the effect you obtain two ports that generate identical code.

To do this you need to find all architecture dependent code. In case of compiler, assembler and linker the *find* command with `-exec grep` option is your best friend. In case of the runtime and the standard library you should search for all \*_GOARCH.\* files and copy them with your NEWGOARCH name. You should also search for all files with `// +build.*GOARCH` directive and add your NEWGOARCH to it.

#### The Assembler

The next step is probably also universal, although maybe someone would choose a different path. You need to open the documentation of your ISA, cd to the [goroot/src/cmd/internal/obj/NEWGOARCH](https://github.com/embeddedgo/go/tree/embedded/src/cmd/internal/obj/thumb) directory and don't leave that directory for the next few weeks.

The Go assembler has many instructions which are common to all ports, e.g. RET, JMP, CALL, ADD, SUB so there's a good chance that at the beginning you won't have to change anything in [goroot/src/cmd/asm/internal](https://github.com/embeddedgo/go/tree/embedded/src/cmd/asm/internal). However, it will come the time that you will also need to look into this directory.

After implementing a few first instructions you should definitely implement them also in the disassembler. From now every new instruction should be implemented in both ones because it practically doesn't increase your effort and gives you additional control over the generated code. After implementing each new instruction in both assembler and disassembler I checked the resulting code using this simple script:

```
#!/bin/sh

o="$(basename $1 .s).o"

GOARCH=thumb go tool asm $1 && go tool objdump $o
rm -f $o
```
You can found the disassemler source code in: [goroot/src/cmd/objdump](https://github.com/embeddedgo/go/tree/embedded/src/cmd/objdump) and [goroot/src/cmd/vendor/golang.org/x/arch](https://github.com/embeddedgo/go/tree/embedded/src/cmd/vendor/golang.org/x/arch).

Let's move now to my specific case: linux/thumb.

#### The brief history of Thumb

The thumb is the most important digit of the human hand. Losing a thumb is more disturbing than losing any finger.

![Do gophers have thumbs?]({{ site.baseur }}/images/gopher/gopher-thumb.jpg)

A long time ago, engineers at ARM decided to shorten the instruction words of
their microprocessor. This was dictated by the needs of resource constrained
embedded systems. So they created a new set of 16-bit instructions called Thumb.
Comparing a thumb alone with a whole arm well illustrates the capabilities of the new instruction set.

This new heavily depleted instruction set has been added to the existing one and the decision which set will be used has been left to the developers. The assumption of the engineers was that one program could use both instruction sets for different pieces of code. Unfortunately, they came across a significant problem: the lack of space in the current instruction set encoding for an additional one. The solution to the problem was brilliant although not without some flaws. They decided to use the least significant bit of the address used for indirect branches. If this bit is set the CPU decodes the following instructions in the Thumb mode, if it's clered the instruction decoder switches to the ARM mode.

In practice it turned out that using a thumb instead of the whole arm and switching between them isn't very comfortable. If you have a second fully functional arm you use it all the time and you don't bother with an extra thumb unless you really need it. Engineers at ARM finally recognized this problem and decided to sew the arm back to the thumb. It wasn't the original arm but the result turned out to be surprisingly good. This is how the *Thumb2* instruction set was made. In my personal opinion it's better than the original ARM instruction set. The original one has purely academic background, the Thumb2 is the result of years of engineering experience. The switching bit (Thumb bit) in a branch address remained, even in case of the ARMv7-M that is a Thumb2-only ISA.

#### Convert the ARM assembler into the Thumb2 assembler

So I have working assembler for GOARCH=thumb but it still generates ARM code. It would seem that the conversion should be simple: the same user registers,
mostly the same addressing modes and instruction names. The original arm assembler helped much but the thumb one diverges far away from it. Here are the reasons:

1. Many Thumb2 instructions have two available encodings: 16 or 32 bit. One of my design choices was that the code generator should use the shortest possible encoding. This complicates thigs much compared to the original ARM assembler.

2. The instruction encoding isn't as regular as in ARM.

3. Every ARM instruction has the condition code field which makes it conditional. In Thumb2 only branch instructions are conditional. You can make up to four consecutive instructions conditional preceding them with an IT (if then) instruction but only opposite conditions are allowed in IT block.

4. The another design choise was an automatic generation of IT instructions so in many cases ARM assembly source code can be used unmodified.

5. In ARM all basic arithmetic instructions have the S bit that decides if condition flags will be updated. In Thumb2 only some 32-bit arithmetic instructions have such S bit. Others do it depending on whether they're in the IT block or not.

6. Thumb2 immediates encoding, although more flexible, are mishmash in comparison with ARM immediates encoding.

7. The GOARCH=thumb means the ARMv7-M ISA. It differs much from the ARMv7-A when it comes to the special registers.

#### The Linker

If you have a working assembler your next setp should be porting the linker because the Go asemmbler and linker are thightly coupled and you can't fully test them separately.

For typical architecture, porting the linker should be fairly straightforward. The whole linker source is in the [goroot/src/cmd/link](https://github.com/embeddedgo/go/tree/embedded/src/cmd/link) directory. Your work will consist of:

1. Implement all types of [relocations](https://www.altoros.com/blog/golang-internals-part-3-the-linker-object-files-and-relocations/) (you started this in assembler, which generates relocations for the linker).

2. Implement the executable binary format for your GOOS/ARCH. If it uses common executable format like ELF your work will be limited to making small changes in an existing implementation.

A confirmation of the linker operation will be the ability to assemble and link a simple assembly-only program. It should contain the `_rt0_GOARCH_GOOS` function which is the default entry point. In this program you should test all types of reclocations by loading global variables and directly/indirectly calling other functions. You need a debugger with support for your architecture.

Let's get to the things specific for bare-metal programming. In this case the linker must also handle some device specific things:

1. Memory map.

2. Interrupt handling model.

3. Boot process.

In case of ARMv7-M ISA the memory map is [fixed](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0203h/BEIFDEEB.html) but many manufacturers often like to add something custom. So I add `-M` option to the linker which allows you to specify available memory blocks. For now only two RAM blocks are supported. The first one is the main memory block where all global variables and a heap are placed. The second one, if specified, is considered non-DMA capable and used only by runtime (mainly by memory allocator) for its internal structures.

The start of the text segment, which usually coincides with the beginning of Flash memory, can be specified using `-T` option.

The ARMv7-M ISA defines something like a [vector table](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0552a/BABIFJFG.html) that contains addresses of interrupt handlers. On system reset, the vector table is fixed at address 0x00000000 which usually coincides with the beginning of the text segment and Flash memory. Some manufacturers maps this region also at different adddress (eg. ST maps it to 0x8000000). The first two words of vector table are special: the first one is used as initial value of the main stack pointer, the second one (reset handler) is the address of the entry point, in my case the `_rt0_thumb_noos` function.

In case of the noos/thumb port the linker is responsible for creating this vector table. It search the code for all available interrupt handlers and creates table large enough to accommodate all of them. The main stack is arranged at the beggining of the main RAM block, before the DATA, BSS and HEAP segments. This helps to catch any stack overflow caused by interrupt handlers.

The linker must also handle properly the Thumb bit in function call addresses.

#### The Compiler

You start porting the compiler by creating new ops and rule files in the [goroot/src/cmd/compile/internal/ssa/gen](https://github.com/embeddedgo/go/tree/embedded/src/cmd/compile/internal/ssa/gen) directory. You probably did it in the first step by coping files from an other architecture. Now you should write new ones using istructions implemented by your new assembler.

There are of course many other places in [goroot/src/cmd/compile](https://github.com/embeddedgo/go/tree/embedded/src/cmd/compile) where you should add something. Just look what is made there for the other architectures.

The noos/thumb port adds one new thing to the compiler, the `//go:interrupthandler` directive that allows to write interrupt handlers in Go. It also implements a couple of functions from [embedded/mmio](https://github.com/embeddedgo/go/tree/embedded/src/embedded/mmio) package as intrinsics to ensure memory access ordering and to avoid function call overhead.

#### Summary

In this article I briefly described the first phase of porting Go to the new architecture. 
It concerned code generation. In the second part we will deal with the Go runtime.
