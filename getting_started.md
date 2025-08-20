---
layout: page
title: Getting started
permalink: /getting_started
egver: 1.24.5
---

*Updated: 2025-08-14*

### Prerequisites

1. Go complier.

   You can download it from [go.dev/dl](https://go.dev/dl/).

2. Git command.

   To install git on Linux use the package manager provided by your Linux distribution (apt, pacman, rpm, ...).

   Windows users may check the [git for Windows](https://gitforwindows.org/) website.

   The Mac users may use the git command provided by the [Xcode](https://developer.apple.com/xcode/) commandline tools. Another way is to use the [Homebrew](https://brew.sh/) package manager.

### Embedded Go toolchain

The Embedded Go is released as as [Go toolchain](https://go.dev/doc/toolchain). All you need to get started is this toolchain, along with a simple utility for loading compiled programs onto your hardware.

1. Install the Embedded Go toolchain.

   Make sure the `$GOPATH/bin` directory is in your `PATH`, as tools installed with the `go install` command will be placed here. If you didn't set the `GOPATH` environment variable manually you can find its default value using the `go env GOPATH` command.

   Then install the Embedded Go toolchain using the following two commands:

   ```sh
   go install github.com/embeddedgo/dl/go{{ page.egrver }}-embedded@latest
   go{{ page.egver }}-embedded download
   ```

2. Install egtool.

   ```sh
   go install github.com/embeddedgo/tools/egtool@latest
   ```

### First program

As an example we will write a simple program for Raspberry Pi Pico 2. See also the similar descriptions for [Teensy 4.x](https://github.com/embeddedgo/imxrt?tab=readme-ov-file#getting-started) and [STM32](https://github.com/embeddedgo/stm32?tab=readme-ov-file#getting-started).

1. Create a project directory containing the `main.go` file with your first Go program for RPi Pico 2.

   ```go
   package main

   import (
   	"time"

   	"github.com/embeddedgo/pico/devboard/pico2/board/leds"
   )

   func main() {
   	for {
   		leds.User.Toggle()
   		time.Sleep(time.Second / 2)
   	}
   }
   ```

4. Initialize your project.

   ```sh
   go mod init firstprog
   go mod tidy
   ```

5. Copy the `go.env` file suitable for your board (here is one for [Pico 2](https://github.com/embeddedgo/pico/blob/master/devboard/pico2/examples/go.env) and another one for a [board with 16 MB flash](https://github.com/embeddedgo/pico/blob/master/devboard/weacta10/examples/go.env)).

6. Compile your first program.

   ```sh
   export GOENV=go.env
   go build
   ```

   or

   ```sh
   GOENV=go.env go build
   ```

   or

   ```sh
   egtool build
   ```

   The last one is like `GOENV=go.env go build` but looks for the `go.env` file up the current module directory tree.

7. Connect your Pico 2 to the computer in the BOOT mode (press the onboard button while connecting it to the USB).

8. Load and run.

   ```sh
   egtool load
   ```

### More information

Read [Embedded Go blog](https://embeddedgo.github.io/).

Ask questions on the Embedded Go [user group](https://groups.google.com/forum/#!forum/embeddedgo) or [reddit](https://www.reddit.com/r/EmbeddedGo/).
