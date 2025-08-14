---
layout: page
title: Supported hardware
permalink: /supported_hardware
github: https://github.com/embeddedgo
---

Some suppoorted hardware is considered *first class*. The distinction is mostly about available flash and RAM. Go is flash hungry mainly because any executable binary contains a lot of information about itself. It's required by the garbage collector, automatically growing stacks, stack traces, reflection and other runtime things. The runtime itself is also a serious piece of code providing goroutines, channels, maps, automatic memory management and more.

These big binaries aren't a problem in case of the OS capable systems with gigabytes of RAM and terabytes of disc storage. In the microcontroller word, flash memory has always accounted for a significant portion of the unit cost, so it was available in homeopathic quantities (from Go's point of view). Fortunately, this situation has changed with the introduction of eXcution In Place technology and relatively large/cheap external SPI flash memories. Now you can afford to waste some of your flash on "unnecessary things" as you do in case of disk-based storage.

To be specific, for now the first class hardware is primarily one that has at least 2 MiB of flash and is at the same time easily available and relatively inexpensive for DIY use cases.

### First class

#### [Raspberry Pi RP2350]({{page.github}}/pico) (aka Pico 2)

[Raspberry Pi Pico 2]({{page.github}}/pico/tree/master/devboard/pico2)

[![Raspberry Pi Pico 2]({{page.github}}/pico/raw/master/devboard/pico2/doc/board.png)]({{page.github}}/pico/tree/master/devboard/pico2)

[WeAct RP2350A-V10]({{page.github}}/pico/tree/master/devboard/weacta10)

[![WeAct RP2350A-V10]({{page.github}}/pico/raw/master/devboard/weacta10/doc/board.png)]({{page.github}}/pico/tree/master/devboard/weacta10)

[WeAct RP2350B core board]({{page.github}}/pico/tree/master/devboard/weactb)

[![WeAct RP2350B core board]({{page.github}}/pico/raw/master/devboard/weactb/doc/board.png)]({{page.github}}/pico/tree/master/devboard/weactb)

#### [i.MX RT]({{page.github}}/imxrt)

[Teensy 4.x]({{page.github}}/imxrt/tree/master/devboard/teensy4)

[![Teensy 4.x]({{page.github}}/imxrt/raw/master/devboard/teensy4/doc/board.jpg)]({{page.github}}/imxrt/tree/master/devboard/teensy4)

[FET1061-S]({{page.github}}/imxrt/tree/master/devboard/fet1061)

[![FET1061-S]({{page.github}}/imxrt/raw/master/devboard/fet1061/doc/board.jpg)]({{page.github}}/imxrt/tree/master/devboard/fet1061)

#### [STM32 microcontrollers]({{page.github}}/stm32)

[DevEBox STM32H743]({{page.github}}/stm32/tree/master/devboard/devebox-h743)

[![DevEBox STM32H743]({{page.github}}/stm32/raw/master/devboard/devebox-h743/doc/board.jpg)]({{page.github}}/stm32/tree/master/devboard/devebox-h743)

### Other

#### [STM32 microcontrollers]({{page.github}}/stm32)

[STM32F4-Discovery]({{page.github}}/stm32/tree/master/devboard/f4-discovery)

[![STM32F4-Discovery]({{page.github}}/stm32/raw/master/devboard/f4-discovery/doc/board.jpg)]({{page.github}}/stm32/tree/master/devboard/f4-discovery)

[STM32_MiNi_Pro]({{page.github}}/stm32/tree/master/devboard/minipro-f405)

[![STM32_MiNi_Pro]({{page.github}}/stm32/raw/master/devboard/minipro-f405/doc/board.jpg)]({{page.github}}/stm32/tree/master/devboard/minipro-f405)

[NUCLEO-L496ZG]({{page.github}}/stm32/tree/master/devboard/nucleo-l496zg)

[![NUCLEO-L496ZG]({{page.github}}/stm32/raw/master/devboard/nucleo-l496zg/doc/board.jpg)]({{page.github}}/stm32/tree/master/devboard/nucleo-l496zg)

[Azure IoT DevKit]({{page.github}}/stm32/tree/master/devboard/az3166)

[![Azure IoT DevKit]({{page.github}}/stm32/raw/master/devboard/az3166/doc/board.jpg)]({{page.github}}/stm32/tree/master/devboard/az3166)

[MXCHIP EMW3162]({{page.github}}/stm32/tree/master/devboard/emw3162)

[![MXCHIP EMW3162]({{page.github}}/stm32/raw/master/devboard/emw3162/doc/board.jpg)]({{page.github}}/stm32/tree/master/devboard/emw3162)

#### [Kendryte K210 SOC]({{page.github}}/kendryte)

[Maix Bit]({{page.github}}/kendryte/tree/master/devboard/maixbit)

[![Maix Bit]({{page.github}}/kendryte/raw/master/devboard/maixbit/doc/board.jpg)]({{page.github}}/kendryte/tree/master/devboard/maixbit)

#### [nRF52 microcontrollers]({{page.github}}/nrf5)

[nRF52840-Dongle (PCA10059)]({{page.github}}/nrf5/tree/master/devboard/pca10059)

[![nRF52840-Dongle]({{page.github}}/nrf5/raw/master/devboard/pca10059/doc/board.jpg)]({{page.github}}/nrf5/tree/master/devboard/pca10059)
