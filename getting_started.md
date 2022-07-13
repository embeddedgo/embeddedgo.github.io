---
layout: page
title: Getting started
permalink: /getting_started
gover: 1.18.4
paver: 1.18.4
egrel: 1.18.4.1
---

### Download, install and use

#### 1. Download

Download a binary release suitable for your system.

| File | OS | Arch | Size |
| ---- | -- | ---- | ---- |
| [embeddedgo-{{ page.egrel }}-darwin-amd64.tar.gz](https://www.lnet.pl/embeddedgo/embeddedgo-{{ page.egrel }}-darwin-amd64.tar.gz) | macOS   | x86-64 | 150MB |
| [embeddedgo-{{ page.egrel }}-darwin-arm64.tar.gz](https://www.lnet.pl/embeddedgo/embeddedgo-{{ page.egrel }}-darwin-arm64.tar.gz) | macOS   | ARM64  | 147MB |
| [embeddedgo-{{ page.egrel }}-linux-amd64.tar.gz](https://www.lnet.pl/embeddedgo/embeddedgo-{{ page.egrel }}-linux-amd64.tar.gz)   | Linux   | x86-64 | 148MB |
| [embeddedgo-{{ page.egrel }}-linux-arm64.tar.gz](https://www.lnet.pl/embeddedgo/embeddedgo-{{ page.egrel }}-linux-arm64.tar.gz)   | Linux   | ARM64  | 142MB |
| [embeddedgo-{{ page.egrel }}-windows-amd64.zip](https://www.lnet.pl/embeddedgo/embeddedgo-{{ page.egrel }}-windows-amd64.zip)     | Windows | x86-64 | 165MB |

#### 2. Install

Optionally remove the previous Embedded Go instalation and any related changes in PATH.

Extract the archive you just downloaded.

Add the extracted directory to your PATH. In case of Linux and Mac you can run `scripts/setup.sh` to do this.

#### 3. Use

Embedded Go is a superset of the original Go so you can use it the same way as the original one both for embedded and system programming.

The main tool of the original Go toolchain is the `go` command. Embedded Go obviously provides `go` too. It works the same as the original one. However, Embedded Go provides also a thin wrapper over `go` called `emgo`. Using emgo instead of go gives several benefits but the two most important are:

1. Make it easier to use Embedded Go for embedded programming in presence of the orignal Go compiler (different, more satble version) used for system programming.

2. Allow you to customize some of the Embedded Go environment variables using a `build.cfg` file. It is especially useful when you work on multiple projects for different embedded targets at the same time.

#### 4. Example (Linux)

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

Create a simple project for STM32F4-DISCOVERY board

```
$ mkdir blinky
$ cp embeddedgo-{{ page.egrel }}-linux-amd64/project_template/build.cfg blinky
$ cd blinky
$ emgo mod init blinky
go: creating new go.mod: module blinky
$ cat >main.go
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

Build

```
$ emgo build
main.go:6:2: no required module provides package github.com/embeddedgo/stm32/devboard/f4-discovery/board/leds; to add it:
        go get github.com/embeddedgo/stm32/devboard/f4-discovery/board/leds
$ emgo get github.com/embeddedgo/stm32
go: downloading github.com/embeddedgo/stm32 v0.10.0
go: added github.com/embeddedgo/stm32 v0.10.0
$ emgo build
$ ls
blinky.elf  build.cfg  go.mod  go.sum  main.go
```

#### 5. More examples

See example programs for various microcontrollers and development boards on Github: [Kendryte](https://github.com/embeddedgo/kendryte/tree/master/devboard), [nRF52](https://github.com/embeddedgo/nrf5/tree/master/devboard), [STM32](https://github.com/embeddedgo/stm32/tree/master/devboard).

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

Use `emgo` instead of `go` to interact with the Embedded Go toolchain. Emgo wraps the go command. It first looks for the Go tree in the directory where it is installed and uses the go tool from it if found. Otherwise it uses `exec.LookPath("go")` to find the go binary elsewhere.

### More information

Read [Embedded Go blog](https://embeddedgo.github.io/).

Ask questions on [Embedded Go Group](https://groups.google.com/forum/#!forum/embeddedgo).

