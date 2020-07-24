---
layout: page
title: Getting started
permalink: /getting_started
---

There is an [article]({{ site.baseur }}/2019/11/16/go_on_not_so_small_hardware.html) that describes the first steps with Embedded Go. It assumes you use some kind of Unix like operating system (Linux, OS X, *BSD). This article will be updated until we create a more general description here.

We are waiting for any success story on Windows. Please send a link or do a pull reqeust with an article text to the [embeddedgo.github.io](https://github.com/embeddedgo/embeddedgo.github.io/).

#### How to install Go compiler with support for bare-metal programming (in short)

Download [embeddedgo/patch](https://github.com/embeddedgo/patch) repository:

```
git clone https://github.com/embeddedgo/patch
```

Download the Go compiler:

```
git clone https://go.googlesource.com/go goroot
```

Apply patch and build the Go distribution (takes about 2 minutes):

```
cd goroot
git checkout go1.14.5
patch -p1 <../patch/go1.14.5
cd src
./make.bash
```

You can run *all.bash* instead of *make.bash* to test the compiler. More information about installing Go from source is available on the [Go website](https://golang.org/doc/install/source).

The alternate (not recommended) way is to clone the [embeddedgo/go](https://github.com/embeddedgo/go) repository. It contains the latest (unstable) version of the compiler.

```
git clone https://github.com/embeddedgo/go goroot
cd goroot/src
./make.bash
```

Ask questions on [Embedded Go Group](https://groups.google.com/forum/#!forum/embeddedgo).
