```go
import (
	"embedded/rtos"

	"github.com/embeddedgo/stm32/hal/gpio"
	"github.com/embeddedgo/stm32/hal/spi"
	"github.com/embeddedgo/stm32/hal/spi/spi1"

	_ "github.com/embeddedgo/stm32/devboard/f4-discovery/board/init"
)

func main() {
	// Allocate GPIO pins

	pa := gpio.A()
	pa.EnableClock(true)
	sck := pa.Pin(5)
	miso := pa.Pin(6)
	mosi := pa.Pin(7)

	pe := gpio.E()
	pe.EnableClock(true)
	cs := pe.Pin(3)

	// Configure SPI pins

	spi1.UsePinMaster(spi.SCK, sck)
	spi1.UsePinMaster(spi.MOSI, mosi)
	spi1.UsePinMaster(spi.MISO, miso)
	
	cs.Set() // CS active state is low
	cs.Setup(&gpio.Config{Mode: gpio.Out})

	// Configure and enable SPI

	d := spi1.Driver()
	d.Setup(spi.Master|spi.CPOL1|spi.CPHA1|spi.SoftSS|spi.ISSHigh, 10e6)
	d.SetWordSize(8)
	d.Enable()

	const (
		CTRL_REG4 = 0x20
		CTRL_REG3 = 0x23
		STATUS    = 0x27
		OUT_X_L   = 0x28
		OUT_X_H   = 0x29
		OUT_Y_L   = 0x2A
		OUT_Y_H   = 0x2B
		OUT_Z_L   = 0x2C
		OUT_Z_H   = 0x2D
	)

	write := func(addr, val uint8) {
		cs.Clear()
		d.WriteReadByte(addr)
		d.WriteReadByte(val)
		cs.Set()
	}

	read := func(addr uint8) byte {
		cs.Clear()
		d.WriteReadByte(1<<7 | addr)
		val := d.WriteReadByte(0)
		cs.Set()
		return val
	}

	// Startup sequence

	rtos.Nanosleep(10e6 - rtos.Nanotime())
	write(CTRL_REG4, 0x67)
	write(CTRL_REG3, 0xC8)

	// Reading acceleration data using the status register

	for {
		status := read(STATUS)
		if status>>3&1 == 0 {
			continue
		}
		datalost := status>>7&1 != 0

		xl := read(OUT_X_L)
		xh := read(OUT_X_H)
		yl := read(OUT_Y_L)
		yh := read(OUT_Y_H)
		zl := read(OUT_Z_L)
		zh := read(OUT_Z_H)

		x := int(int16(xl)|int16(xh)<<8) * 2000 / 32768
		y := int(int16(yl)|int16(yh)<<8) * 2000 / 32768
		z := int(int16(zl)|int16(zh)<<8) * 2000 / 32768

		println(x, y, z, datalost)
	}
}
```