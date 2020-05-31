---
layout: post
title: Bare metal RISC-V programming in Go
tags: go embeddedgo risc-v kendryte k210
---

![Kendryte & Go]({{site.baseur}}/images/bare_metal_risc-v_programming_in_go/kendryte_go.png)

<!--more-->

For a couple of weeks I've worked on adding support for bare metal RISC-V programming to Go. As the noos/riscv64 port reached the usable level I've decided to write something about it.

Until now, the [Embeddeg Go](https://github.com/embeddedgo) have supported only ARM microcontrollers. I wanted to add another architecture as soon as possible to revise the design choices. From very beggining I've considered to add support for the very popular WiFi capable [ESP32](https://www.espressif.com/en/products/socs/esp32/overview) microcontroller and even finished studying the Xtensa LX6 ISA but in the meantime two things happened:

- the support for linux/riscv64 [landed in Go compiler](https://golang.org/doc/go1.14#riscv),

- the Siped [Maix Bit](https://maixduino.sipeed.com/en/hardware/board.html) landed in my hands:


![Sipeed Maix Bit]({{site.baseur}}/images/mcu/kendryte/maix_bit.jpg)

As I've already ported Go to a new architecture (ARMv7-M Thumb2 ISA) I know how much work it costs. The ready to use RISC-V port and the access to the real hardware left no choice.


#### Kendryte K210

If you want to play with real RISC-V hardware you don't have much choice today. I'm aware of three sources:

- [SiFive](https://www.sifive.com/), the RISC-V pioneer with they rather expensive [development boards](https://www.sifive.com/boards),

- [GigaDevice](https://www.gigadevice.com/), they provide very cheap RISC-V based [GD32V microcontrollers](https://www.gigadevice.com/products/microcontrollers/gd32/risc-v/) (development boards are provided by [SeedStudio](https://www.seeedstudio.com/SeeedStudio-GD32-RISC-V-Dev-Board-p-4302.html), [Sipeed](https://longan.sipeed.com/en/) and others),

- and finally [Kendryte](https://kendryte.com/) with their [K210](https://s3.cn-north-1.amazonaws.com.cn/dl.kendryte.com/documents/kendryte_datasheet_20181011163248_en.pdf) chip and development boards/modules provided by [Sipeed](https://www.sipeed.com/).

You can buy the Maix Bit board as seen above for about $13 or the whole kit with camera and LCD for about $21.

The heart of the Maix Bit is K210, an AI capable SOC from Kendryte. It's equipped with two [RV64G](https://en.wikipedia.org/wiki/RISC-V#ISA_base_and_extensions) cores and over a dozen peripherals of which the convolutional neural network accelerator (KPU) deserves special attention.

![Sipeed Maix Bit]({{site.baseur}}/images/mcu/kendryte/k210.png)

The good news is the 8 MB of integrated SRAM divided into 6 MB general purpose memory (plenty of space for most Go programs) and 2 MB dedicated to KPU. There is no integrated Flash. The program is loaded from external SPI Flash and runs from RAM.

#### Getting started with Maix Bit

The Maix Bit comes with the preinstaled [MicroPython](http://micropython.org/) port called [MaixPy](https://maixpy.sipeed.com/en/). A good starting point will be to blink an onboard LED using MaixPy and next do the same with Go.

Let's connect the board to the PC and paste the following code to MaixPy.

```python
from time import sleep
from Maix import GPIO

fm.register(board_info.LED_B, fm.fpioa.GPIO0)
led = GPIO(GPIO.GPIO0, GPIO.OUT)

while True:
	led.value(0)
	sleep(1)
	led.value(1)
	sleep(1)
```

{::nomarkdown}
<video width=640 height=480 controls preload=auto>
	<source src='{{site.baseur}}/videos/bare_metal_risc-v_programming_in_go/maixpy.mp4' type='video/mp4'>
	Sorry, your browser doesn't support embedded videos.
</video>
<p></p>
{:/}

As you can see the `board_info` assigns wrong pin to the blue LED but otherwise the code is working as expected.

Let's write a similar program in Go.

```go
package main

import (
	"time"
	"github.com/embeddedgo/kendryte/devboard/maixbit/board/leds"
)

func main() {
	led := leds.Blue
	for {
		led.SetOn()
		time.Sleep(time.Second)
		led.SetOff()
		time.Sleep(time.Second)
	}
}
```

Save it in *maix_blinky* directory as *main.go* file and try to compile.

```
$ cd maix_blinky
$ go mod init maix_blinky
$
$ GOOS=noos GOARCH=riscv64 go build -tags k210 -ldflags '-M 0x80000000:6M'
go: finding module for package github.com/embeddedgo/kendryte/devboard/maixbit/board/leds
go: downloading github.com/embeddedgo/kendryte v0.0.1
go: found github.com/embeddedgo/kendryte/devboard/maixbit/board/leds in github.com/embeddedgo/kendryte v0.0.1
$
$ ls -l
total 1188
-rw-r--r-- 1 michal michal      75 May 31 11:49 go.mod
-rw-r--r-- 1 michal michal     179 May 31 11:49 go.sum
-rw-r--r-- 1 michal michal     222 May 31 11:46 main.go
-rwxr-xr-x 1 michal michal 1200398 May 31 11:52 maix_blinky
```

Compilation succeeded! But to achieve this, you need to add support for noos/riscv64 pair to the vanilla Go compiler. Read [Getting started]({{ site.baseur }}/getting_started) and [github.com/embeddedgo/patch readme](https://github.com/embeddedgo/patch) for more information.

Let's flash our program to the Maix Bit. You need to install [kflash ISP utility](https://github.com/kendryte/kflash.py). The simplest way is to use `pip3 install kflash` command.

```
$ objcopy -O binary maix_blinky maix_blinky.bin
$ kflash -p /dev/ttyUSB0 -b 750000 -B bit_mic maix_blinky.bin
[INFO] COM Port Selected Manually:  /dev/ttyUSB0
[INFO] Default baudrate is 115200 , later it may be changed to the value you set.
[INFO] Trying to Enter the ISP Mode...
[INFO] Greeting Message Detected, Start Downloading ISP
Downloading ISP: |============================================| 100.0% 10kiB/s
[INFO] Booting From 0x80000000
[INFO] Wait For 0.1 second for ISP to Boot
[INFO] Boot to Flashmode Successfully
[INFO] Selected Baudrate:  750000
[INFO] Baudrate changed, greeting with ISP again ...
[INFO] Boot to Flashmode Successfully
[INFO] Selected Flash:  On-Board
[INFO] Initialization flash Successfully
Programming BIN: |============================================| 100.0% 38kiB/s
[INFO] Rebooting...
```

{::nomarkdown}
<video width=640 height=480 controls preload=auto>
	<source src='{{site.baseur}}/videos/bare_metal_risc-v_programming_in_go/blinky1.mp4' type='video/mp4'>
	Sorry, your browser doesn't support embedded videos.
</video>
<p></p>
{:/}

The developement of Go on Kendryte K210 takes place on [github.com/embeddedgo/kendryte](https://github.com/embeddedgo/kendryte). Your can clone the repository and try the other examples. Any help in developing this project is appreciated.

```
$ git clone https://github.com/embeddedgo/kendryte
remote: Enumerating objects: 229, done.
remote: Counting objects: 100% (229/229), done.
remote: Compressing objects: 100% (137/137), done.
remote: Total 229 (delta 48), reused 215 (delta 34), pack-reused 0
Receiving objects: 100% (229/229), 570.65 KiB | 28.00 KiB/s, done.
Resolving deltas: 100% (48/48), done.
$ cd kendryte/devboard/maixbit/examples
$ ls
blinky  build.sh  debug-oocd.sh  goroutines  load-kflash.sh
$ cd blinky
$ ../build.sh
$ ../load-kflash.sh
```

{::nomarkdown}
<video width=640 height=480 controls preload=auto>
	<source src='{{site.baseur}}/videos/bare_metal_risc-v_programming_in_go/blinky2.mp4' type='video/mp4'>
	Sorry, your browser doesn't support embedded videos.
</video>
<p></p>
{:/}

#### Sumary

The Kendryte K210 is impresive piece of hardware for it's price. It seems to be an ideal platform to play with Embedded Go.

At the end I will describe briefly the Maix Bit v2 development board.

![Maix Bit explained]({{site.baseur}}/images/bare_metal_risc-v_programming_in_go/maix_bit_num.jpg)

From the left tho the right:

**1** USB-C connector, connected to the CH552, can be used to power the board, program it and communicate with K210 through UART3,

**2** BOOT/USER button, pressed during reset starts the bootloader in programming mode, you can read its state in your code,

**3** RESET button,

**4** power LED,

**5** RGB LED,

**6** CH552 MCU, presents itself as FTDI FT2232H USB to UART converter, has ability to reset the K210 chip,

**7** MEMS microphone,

**8** CH552 UART Rx/Tx LEDs,

**9** DC/DC step-down PMU (RY1303A), provides all voltages need by K210 (0.9V, 1.8V, 3.3V),

**A** Kendryte K210 SOC,

**B** 16 MB SPI Flash,

**C** camera connector,

**D** LCD connector.

*Michał Derkacz*
