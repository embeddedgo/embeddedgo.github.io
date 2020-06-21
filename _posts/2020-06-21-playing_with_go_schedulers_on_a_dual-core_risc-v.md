---
layout: post
title: Playing with Go schedulers on a dual-core RISC-V
tags: go golang embeddedgo risc-v kendryte k210
image: images/playing_with_go_schedulers_on_a_dual-core_risc-v/dual-core_go.png
---

![Kendryte & Go]({{site.baseurl}}/images/playing_with_go_schedulers_on_a_dual-core_risc-v/dual-core_go.png)

<!--more-->

You can port the Go runtime to a system that doesn't implement threads. An example would be the current [WebAssembly](https://en.wikipedia.org/wiki/WebAssembly) port.

```go
func newosproc(mp *m) {
	panic("newosproc: not implemented")
}
```

But if you want to run a bare metal Go program on multiple cores the thread abstraction is a must, unless you are ready to implement a completely new goroutine scheduler.

The goroutine scheduler uses operating system threads as workhorses to execute its goroutines. The goal is to efficiently run thousands of goroutines using only a few OS threads. Threads are considered expensive. There is also much cheaper to access shared resources by multiple goroutines that run on the same thread. This is further optimized in Go by introducing the concept of a logical processor (called P) that has local cache of the most commonly used resources and can "execute" only one thread at a time. At the sane time there can be unlimited number of threads sleeping in the system calls.

You can set number of logical processors using [GOMAXPROCS](https://golang.org/pkg/runtime) environment variable or [runtime.GOMAXPROCS](https://golang.org/pkg/runtime/#GOMAXPROCS) function. The default GOMAXPROCS for [Kendryte K210]({{site.baseurl}}/2020/05/31/bare_metal_programming_risc-v_in_go.html) is 2.

#### Tasker

The Embedded Go implements a thread scheduler for GOOS=noos called [tasker](https://github.com/embeddedgo/go/blob/embedded/src/runtime/tasker_noos.go). Tasker was designed from the very beginning as a multi-core scheduler but the first multicore tests and bug fixes were done on K210 while working on *noos/riscv64* port.

Tasker is tightly coupled to the [goroutine scheduler](https://github.com/golang/go/blob/release-branch.go1.14/src/runtime/proc.go). It doesn't have it's own representation of thread. Instead it directly uses the [m structs](https://github.com/golang/go/blob/release-branch.go1.14/src/runtime/runtime2.go#L473) obtained from goroutine scheduler. The Go logical processor concept is taken seriously. Any available P is associated with a specific CPU using the following simple formula:

```go
cpuid = P.id % len(allcpus)
```

As you can see, when choosing a CPU for thread the tasker [relies on the goroutine scheduler decision](https://github.com/embeddedgo/go/blob/2db1fdb0d000ee94a737c9384e0b6bb8001a5c21/src/runtime/tasker_noos.go#L170).

CPU is the name used by the tasker for any independent hardware thread of execution, a *hart* in the [RISC-V](https://en.wikipedia.org/wiki/RISC-V) nomenclature.

The tasker threads are cheap.

#### Print hartid

Let's start playing with Go schedulers and two cores available in K210. The basic tool we need is a function that returns the current hart id which in case of K210 means the core id.

```go
package main

import _ "github.com/embeddedgo/kendryte/devboard/maixbit/board/init"

func hartid() int

func main() {
	for {
		print(hartid())
	}
}
```

As you can see the *hartid* function has no body. To define it we have to reach for [Go assembly](https://golang.org/doc/asm).


```
#include "textflag.h"

#define CSRR(CSR,RD) WORD $(0x2073 + RD<<7 + CSR<<20)
#define mhartid 0xF14
#define s0 8

// func hartid() int
TEXT ·hartid(SB),NOSPLIT|NOFRAME,$0
	CSRR  (mhartid, s0)
	MOV   S0, ret+0(FP)
	RET
```

The Go assembler doesn't recognize privileged instructions so we used macros to implement the *CSRR* instruction.

Let's use *GDB+OpenOCD* to load and run the compiled program. I recommend using the [modified version](https://github.com/sysprogs/openocd-kendryte/) of the [openocd-kendryte](https://github.com/kendryte/openocd-kendryte). You can use the [debug-oocd.sh](https://github.com/embeddedgo/scripts/blob/master/debug-oocd.sh) helper script as shown in the [maixbit example](https://github.com/embeddedgo/kendryte/blob/master/devboard/maixbit/examples/debug-oocd.sh). GDB isn't required to follow this article. You can use *kflash* utility instead as described in the [previous article]({{site.baseurl}}/2020/05/31/bare_metal_programming_risc-v_in_go.html).

```
Core [0] halted at 0x8000bb4c due to debug interrupt
Core [1] halted at 0x800093ea due to debug interrupt
(gdb) load
Loading section .text, size 0x62230 lma 0x80000000
Loading section .rodata, size 0x2c80f lma 0x80062240
Loading section .typelink, size 0x658 lma 0x8008ec20
Loading section .itablink, size 0x18 lma 0x8008f278
Loading section .gopclntab, size 0x3df15 lma 0x8008f2a0
Loading section .go.buildinfo, size 0x20 lma 0x800cd1b8
Loading section .noptrdata, size 0xf00 lma 0x800cd1d8
Loading section .data, size 0x3f0 lma 0x800ce0d8
Start address 0x80000000, load size 844500
Transfer rate: 64 KB/sec, 14313 bytes/write.
(gdb) c
Continuing.

Program received signal SIGTRAP, Trace/breakpoint trap.
runtime.defaultHandler ()
    at /home/michal/embeddedgo/go/src/runtime/tasker_noos_riscv64.s:388
(gdb)
```

As you can see our program has been stopped in `runtime.defaultHandler`. This function handles unsupported traps (there are still a lot of them). Let's see what happened.

```
(gdb) p $a0/8
$1 = 2
```

The A0 register contains a value of the *mcause* CSR saved at the trap entry (multiplied by 8). We can't rely on the current mcause value because the interrupts are enabled. Bu we can check if it's the same.

```
(gdb) p $mcause
$2 = 2
```

It seems there was no other traps in the meantime. The *mcause* CSR contains a code indicating the event that caused the trap. In our case it's *Illegal instruction exception*. Let's see what this illegal instruction is. The *mepc* register (return address from trap) was saved on the stack.

```
(gdb) x $sp+24
0x800d4820:     0x80062221
```

As before we can check does it's the same as the current one.

```
(gdb) p/x $mepc
$2 = 0x80062220
```

Almost the same (LSBit is used to save *fromThread* flag).

```
(gdb) list *0x80062220
0x80062220 is in main.hartid (/home/michal/embeddedgo/kendryte/devboard/maixbit/examples/multicore/asm.s:9).
4       #define mhartid 0xF14
5       #define s0 8
6
7       // func hartid() int
8       TEXT ·hartid(SB),NOSPLIT|NOFRAME,$0
9               CSRR  (mhartid, s0)
10              MOV   S0, ret+0(FP)
11              RET
12
13      // func  loop(n int)
```

All clear. Our program runs in the RISC-V *user mode*. We have no access to the *machine mode* CSRs. But there is a way to tackle this problem.

```go
func main() {
	runtime.LockOSThread()
	rtos.SetPrivLevel(0)
	for {
		print(hartid())
	}
}
```

The *rtos.SetPrivLevel* function can be used to change the privilege level for the current thread . As it affects the current thread only we must call *runtime.LockOSThread* first to wire our goroutine to its current thread (no other goroutine will execute in this thread). Now we can run our program.

{::nomarkdown}
<video width=640 height=480 controls preload=auto>
	<source src='{{site.baseurl}}/videos/playing_with_go_schedulers_on_a_dual-core_risc-v/zeros.mp4' type='video/mp4'>
	Sorry, your browser doesn't support embedded videos.
</video>
<p></p>
{:/}

As you can see our printing thread is locked to hart 0.

#### Multiple threads

Let's modify the previous code in a way that allows us to easily alter the number of threads.

```go
package main

import (
	"embedded/rtos"
	"runtime"

	_ "github.com/embeddedgo/kendryte/devboard/maixbit/board/init"
)

type report struct {
	tid, hartid int
}

var ch = make(chan report, 3)

func thread(tid int) {
	runtime.LockOSThread()
	rtos.SetPrivLevel(0)
	for {
		ch <- report{tid, hartid()}
	}
}

func main() {
	var lasthart [2]int
	for i := range lasthart {
		go thread(i)
	}
	runtime.LockOSThread()
	rtos.SetPrivLevel(0)
	for r := range ch {
		lasthart[r.tid] = r.hartid
		print(" ", hartid())
		for _, hid := range lasthart {
			print(" ", hid)
		}
		println()
	}
}

func hartid() int
```

Now the main function launches len(lasthart) goroutines and after that prints in a loop the hartid for itself and all other goroutines. Every launched goroutine periodically checks its hartid and sends report to the main goroutine.

Let's start with main + 2 goroutines.

{::nomarkdown}
<video width=640 height=480 controls preload=auto>
	<source src='{{site.baseurl}}/videos/playing_with_go_schedulers_on_a_dual-core_risc-v/main+2.mp4' type='video/mp4'>
	Sorry, your browser doesn't support embedded videos.
</video>
<p></p>
{:/}

You can see we have a stable state: the main goroutine runs on hart 1, the reporting goroutines run on hart 0. Let's add more goroutines.

{::nomarkdown}
<video width=640 height=480 controls preload=auto>
	<source src='{{site.baseurl}}/videos/playing_with_go_schedulers_on_a_dual-core_risc-v/main+30.mp4' type='video/mp4'>
	Sorry, your browser doesn't support embedded videos.
</video>
<p></p>
{:/}

The beginning looks interesting:

```
 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 0 0 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 1 0 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 1 1 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 1 1 1 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 1 1 1 1 0 0 0 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 0 0 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 0 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 0 0 0 0 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 0 0 0 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 0 0 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 0 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 0 0 0 0 0 1 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 0 0 0 0 1 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 0 0 0 1 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 0 0 1 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 0 1 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 1 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 0 0 0 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 0 0 1 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 0 1 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 1 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 0 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 1 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1
 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1
```

It seems that almost all reporting threads start on hart 0 but after a while they migrate to hart 1 and stay there.

Remember the goroutine scheduler can't run more than 2 goroutines at the same time. Our reporting goroutines don't do much. They spend most of their time sleeping on the full channel. It seems reasonable to gather them all on one P and give the other P for busy main thread.

Let's increase the number of logical processors by adding `runtime.GOMAXPROCS(4)` at the beginning of the main function.

{::nomarkdown}
<video width=640 height=480 controls preload=auto>
	<source src='{{site.baseurl}}/videos/playing_with_go_schedulers_on_a_dual-core_risc-v/main+30_4p.mp4' type='video/mp4'>
	Sorry, your browser doesn't support embedded videos.
</video>
<p></p>
{:/}

It seems the goroutine scheduler cannot reach a stable state. But we can see the hart id only. Can we see also the logical processor id? Yes, we can. Let's modify the hartid function to return both.

```
// func hartid() int
TEXT ·hartid(SB),NOSPLIT|NOFRAME,$0
	CSRR  (mhartid, s0)
	MOV   48(g), A0    // g.m
	MOV   160(A0), A0  // m.p
	MOVW  (A0), S1     // p.id
	SLL   $1, S1
	OR    S1, S0
	MOV   S0, ret+0(FP)
	RET
```

The `print(" ", hartid())` call has been changed to `print(hid>>1, hid&1)` to show both numbers next to each other.

{::nomarkdown}
<video width=640 height=480 controls preload=auto>
	<source src='{{site.baseurl}}/videos/playing_with_go_schedulers_on_a_dual-core_risc-v/main+30_4p_ph.mp4' type='video/mp4'>
	Sorry, your browser doesn't support embedded videos.
</video>
<p></p>
{:/}

As you can see the goroutine scheduler keeps the main goroutine on P=0,1 and reporting goroutines on P=2,3. Our simple rule that maps Ps to CPUs causes threads to jump between K210 cores.

Ending this article let's get back to two P's but let's give our reporting goroutines something to do. As we've already got some practice with Go assembly we will use it to write simple busy loop. Thanks to this we'll be sure the compiler won't optimize this code.

```
// func loop(n int)
TEXT ·loop(SB),NOSPLIT|NOFRAME,$0
	MOV  n+0(FP), S0
	BEQ  ZERO, S0, end
	ADD  $-1, S0
	BNE  ZERO, S0, -1(PC)
end:
	RET
```

You can find the full code for this last case on [github](https://github.com/embeddedgo/kendryte/tree/master/devboard/maixbit/examples/multicore). You can play with other things, like the channel length, the loop count, odd GOMAXPROCS values, etc.

{::nomarkdown}
<video width=640 height=480 controls preload=auto>
	<source src='{{site.baseurl}}/videos/playing_with_go_schedulers_on_a_dual-core_risc-v/main+30loop_ph.mp4' type='video/mp4'>
	Sorry, your browser doesn't support embedded videos.
</video>
<p></p>
{:/}

Workload disturbs the stable state from the second example. We can observe quite long periods when all goroutines run on the same logical processor which may be disturbing.

#### Summary

It's hard to draw any deeper conclusions from these superficial tests. It wasn't the purpose of this article. We have some fun with Go, RISC-V assembler, debugger and underlying hardware which is what you can expect from bare-metal programming. It seems the goroutine scheduler and tasker both work in harmony with each other. A more strict approach is needed to draw more definitive conclusions that can be used to improve one or the other.

*Michał Derkacz*