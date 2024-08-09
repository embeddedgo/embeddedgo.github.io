---
layout: page
title: Getting started
permalink: /getting_started
gover: 1.22.5
paver: 1.22.5
egrel: 1.22.5
eprev: 1.20.12.2
---

*Updated: 2024-08-09*

### Binary releases

Download a binary release suitable for your system.

| File | OS | Arch |
| ---- | -- | ---- |
| [embeddedgo-{{ page.egrel }}-darwin-amd64.tar.gz](https://www.lnet.pl/embeddedgo/embeddedgo-{{ page.egrel }}-darwin-amd64.tar.gz) | macOS   | x86-64 |
| [embeddedgo-{{ page.egrel }}-darwin-arm64.tar.gz](https://www.lnet.pl/embeddedgo/embeddedgo-{{ page.egrel }}-darwin-arm64.tar.gz) | macOS   | ARM64  |
| [embeddedgo-{{ page.egrel }}-linux-amd64.tar.gz](https://www.lnet.pl/embeddedgo/embeddedgo-{{ page.egrel }}-linux-amd64.tar.gz)   | Linux   | x86-64 |
| [embeddedgo-{{ page.egrel }}-linux-arm64.tar.gz](https://www.lnet.pl/embeddedgo/embeddedgo-{{ page.egrel }}-linux-arm64.tar.gz)   | Linux   | ARM64  |
| [embeddedgo-{{ page.egrel }}-windows-amd64.zip](https://www.lnet.pl/embeddedgo/embeddedgo-{{ page.egrel }}-windows-amd64.zip)     | Windows | x86-64 |

Previous release.

| File | OS | Arch |
| ---- | -- | ---- |
| [embeddedgo-{{ page.eprev }}-darwin-amd64.tar.gz](https://www.lnet.pl/embeddedgo/embeddedgo-{{ page.eprev }}-darwin-amd64.tar.gz) | macOS   | x86-64 |
| [embeddedgo-{{ page.eprev }}-darwin-arm64.tar.gz](https://www.lnet.pl/embeddedgo/embeddedgo-{{ page.eprev }}-darwin-arm64.tar.gz) | macOS   | ARM64  |
| [embeddedgo-{{ page.eprev }}-linux-amd64.tar.gz](https://www.lnet.pl/embeddedgo/embeddedgo-{{ page.eprev }}-linux-amd64.tar.gz)   | Linux   | x86-64 |
| [embeddedgo-{{ page.eprev }}-linux-arm64.tar.gz](https://www.lnet.pl/embeddedgo/embeddedgo-{{ page.eprev }}-linux-arm64.tar.gz)   | Linux   | ARM64  |
| [embeddedgo-{{ page.eprev }}-windows-amd64.zip](https://www.lnet.pl/embeddedgo/embeddedgo-{{ page.eprev }}-windows-amd64.zip)     | Windows | x86-64 |


### Instalation

Remove the previous Embedded Go instalation and any related changes in PATH.

Extract the archive you just downloaded.

Add the extracted directory to your PATH as described below.

#### Windows

Search and open *Advanced system settings*. Go to the *Advanced* tab and click *Environment Variables...* Edit the *Path* variable in *System Variables* or *User Variables*.

#### Unix (Linux, Mac)

Run `scripts/setup.sh` or manualy alter the PATH variable in your profile file (`.profile`, `.bash_profile`).

### Usage

Embedded Go is a superset of the original Go so you can use it the same way as the original one both for embedded and system programming.

The main tool of the original Go toolchain is the `go` command. Embedded Go obviously provides `go` too. It works the same as the original one. However, Embedded Go provides also a thin wrapper over `go` called `emgo`. Using `emgo` instead of `go` gives several benefits but the three most important are:

- Facilitate the use of Embedded Go for embedded programming in the presence of the original Go compiler used for system programming (different, more satable version, etc.).

- Allow to customize some of the Embedded Go environment variables using the `build.cfg` file. It's especially useful when you work at the same time on the multiple projects for different embedded targets. You can still use environment variables to configure Embedded Go if you want but variables set in `build.cfg` have priority.

- Allow to convert the produced ELF binary to the other formats like Intel HEX or raw BIN.

### Example (Linux)

Install

```
$ tar xf embeddedgo-{{ page.egrel }}-linux-amd64.tar.gz
$ ls embeddedgo-{{ page.egrel }}-linux-amd64
emgo  go  project_template  scripts
```

Alter the PATH environment variable

```
$ embeddedgo-{{ page.egrel }}-linux-amd64/scripts/setup.sh
Adding /home/test/embeddedgo-{{ page.egrel }}-linux-amd64 to the PATH in the
folowing profile files:

  /home/test/.profile

Changes take effect on next login. Please relogin.
```

Create a simple project for the [STM32F4-DISCOVERY](https://github.com/embeddedgo/stm32/blob/master/devboard/f4-discovery/doc/board.jpg) board (see also the [article about Teensy 4]({{ site.baseur }}/2024/01/01/go_http_server_on_teensy4.html))

```
$ mkdir blinky
$ cp embeddedgo-{{ page.egrel }}-linux-amd64/project_template/build.cfg blinky
$ cd blinky
$ emgo mod init blinky
go: creating new go.mod: module blinky
$ cat > main.go
package main

import (
	"time"

	"github.com/embeddedgo/stm32/devboard/f4-discovery/board/leds"
)

func main() {
	for {
		leds.Green.SetOff()
		leds.Orange.SetOn()
		time.Sleep(time.Second / 2)

		leds.Orange.SetOff()
		leds.Red.SetOn()
		time.Sleep(time.Second / 2)

		leds.Red.SetOff()
		leds.Blue.SetOn()
		time.Sleep(time.Second / 2)

		leds.Blue.SetOff()
		leds.Green.SetOn()
		time.Sleep(time.Second / 2)
	}
}
<Ctrl+D>
$
```

Compile

```
$ emgo mod tidy
go: downloading github.com/embeddedgo/stm32 v0.10.0
go: added github.com/embeddedgo/stm32 v0.10.0
$ emgo build
$ ls
blinky.elf  build.cfg  go.mod  go.sum  main.go
```

Program the board using [OpenOCD](https://openocd.org/)

```
$ cat > load.sh
#!/bin/sh

TARGET=stm32f4x
RESET=srst_only
TRACECLKIN=168000000

. $(emgo env GOROOT)/../scripts/load-oocd.sh
<Ctrl+D>
$ chmod a+x load.sh
$ ./load.sh
Bus 003 Device 008: ID 0483:3748 STMicroelectronics ST-LINK/V2
Open On-Chip Debugger 0.11.0+dev-00685-g830d70bfc (2022-05-17-14:21)
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
debug_level: 0

srst_only separate srst_nogate srst_open_drain connect_deassert_srst

TARGET: stm32f4x.cpu - Not halted
$ ./load.sh
Bus 003 Device 009: ID 0483:3748 STMicroelectronics ST-LINK/V2
Open On-Chip Debugger 0.11.0+dev-00685-g830d70bfc (2022-05-17-14:21)
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
debug_level: 0

srst_only separate srst_nogate srst_open_drain connect_deassert_srst

target halted due to debug-request, current mode: Thread
xPSR: 0x01000000 pc: 0x0803414c msp: 0x20000800
target halted due to debug-request, current mode: Thread
xPSR: 0x01000000 pc: 0x0803414c msp: 0x20000800
** Programming Started **
** Programming Finished **
```

#### More examples

See example programs for various microcontrollers and development boards on Github: [Kendryte](https://github.com/embeddedgo/kendryte/tree/master/devboard), [nRF52](https://github.com/embeddedgo/nrf5/tree/master/devboard), [STM32](https://github.com/embeddedgo/stm32/tree/master/devboard), [IMXRT](https://github.com/embeddedgo/imxrt/tree/master/devboard).

You can find the proper build settings for every supported MCU/board in the board examples subdirectory, eg: [minipro-f405/examples/build.cfg](https://github.com/embeddedgo/stm32/blob/master/devboard/minipro-f405/examples/build.cfg).


### Installing Embedded Go from source (Linux, Mac)

Make a directory for Embedded Go toolchain:

```
mkdir embeddedgo
cd embeddedgo
```

Download [embeddedgo/patch](https://github.com/embeddedgo/patch), [embeddedgo/tools](https://github.com/embeddedgo/tools) and (optionally) [embeddedgo/scripts](https://github.com/embeddedgo/scripts) repositories:

```
git clone https://github.com/embeddedgo/patch
git clone https://github.com/embeddedgo/tools
git clone https://github.com/embeddedgo/scripts
```

Download the Go compiler:

```
git clone https://go.googlesource.com/go
```

Select the latest supporetd Go release, apply the correct patch and build the Embedded Go toolchain (takes about 2 minutes):

```
cd go
git checkout go{{ page.gover }}
patch -p1 <../patch/go{{ page.paver }}
cd src
./make.bash
```

You can run `all.bash` instead of `make.bash` to run tests. More information about installing Go from source is available on the [Go website](https://go.dev/doc/install/source).

The alternate (not recommended) way is to clone the [embeddedgo/go](https://github.com/embeddedgo/go) repository. It contains the latest (unstable) version of the compiler.

```
git clone https://github.com/embeddedgo/go
cd go/src
./make.bash
```

Build the `emgo` tool and move it to the root of the Embedded Go toolchain.

```
cd ../../tools/emgo
../../go/bin/go build
mv emgo ..
```

Add your `embeddedgo` directory to the PATH:

```
export PATH=$PATH:/path/to/the/embeddedgo
```

Use `emgo` instead of `go` to interact with the Embedded Go toolchain. Emgo wraps the go command. It first looks for the Go tree in the directory where it's installed and uses the go tool from it if found. Otherwise it uses `exec.LookPath("go")` to find the go binary elsewhere.

### More information

Read [Embedded Go blog](https://embeddedgo.github.io/).

Ask questions on the Embedded Go [user group](https://groups.google.com/forum/#!forum/embeddedgo) or [reddit](https://www.reddit.com/r/EmbeddedGo/).
