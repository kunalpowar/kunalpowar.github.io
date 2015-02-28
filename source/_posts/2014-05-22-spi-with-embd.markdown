---
layout: post
title: "SPI with EMBD"
date: 2014-05-22 09:52:03 +0530
comments: true
categories: embd, tech, golang, embedded
published: true
---

Once the mcp3008(a spi enabled 10-bit 8 channel ADC) was lost and found amoung our huge hardware heap i quickly started thinking about how to implement SPI in golang and include it as the next communicaton protocol supported under [EMBD](http://embd.kidoman.io). I looked through the existing implementations for the same, so that the users wont be alien to the end-points the library would provide. There was one  interesting problem at hand. EMBD is written to support multiple boards. The core functionality of SPI remains same for all the linux based ones. But in case of beaglebone, one extra step had to be done. SPI is not enabled by default in a bbb. And we dont want users to load the specific dtc everytime it boots up. So the library itself has to handle initializing SPI behind the scenes. Does this mean that i need to write separate files with essentially the same code for bbb and rpi (currently supported boards)..? In the world of clean and DRY style of coding, that would be a horrendous crime. So after a long thought this came out as a viable solution. Every host simply tells the generic spi driver if anything has to be done before the core spi features are available. In this case bbb sends a initializer function to the driver and rpi sends nil. Once the SPI core was ready, writing a package for mcp3008 was a piece of cake, thanks to the awesomely detailed datasheet.

###Sample

The following sample uses the mcp3008 package to get 10 bit Digital value of the analog input at 0th channel via SPI. 
> This works **without any code change** on both beaglebone black and raspberypi

```go
package main

import (
	"flag"
	"fmt"
	"time"

	"github.com/kidoman/embd"
	"github.com/kidoman/embd/convertors/mcp3008"
	_ "github.com/kidoman/embd/host/all"
)

const (
	channel = 0
	speed   = 1000000
	bpw     = 8
	delay   = 0
)

func main() {
	flag.Parse()
	fmt.Println("this is a sample code for mcp3008 10bit 8 channel ADC")

	if err := embd.InitSPI(); err != nil {
		panic(err)
	}
	defer embd.CloseSPI()

	spiBus := embd.NewSPIBus(embd.SPIMode0, channel, speed, bpw, delay)
	defer spiBus.Close()

	adc := mcp3008.New(mcp3008.SingleMode, spiBus)

	for i := 0; i < 20; i++ {
		time.Sleep(1 * time.Second)
		val, err := adc.AnalogValueAt(0)
		if err != nil {
			panic(err)
		}
		fmt.Printf("analog value is: %v\n", val)
	}
}

```

###TODOs
* Cover all the modes for SPI
* Provide support for the differential mode of operation for mcp3008
