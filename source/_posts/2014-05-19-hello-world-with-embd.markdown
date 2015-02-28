---
layout: post
title: "Hello world with EMBD"
date: 2014-05-19 17:27:17 +0530
comments: true
categories: 
published: true
---

###A Little Story
Here we are, one of the usual night outs, hacking on a raspberry pi to get it working towards [the bot](http://kidoman.io/engineering/thebot.html) idea. Started with one blinking led and then 4. Why 4? We thought if we are able to control 4 leds, then we can expand it to a 4wire parallel communication based on sending nibbles of data. With the poor pi giving us just 9 usable GPIO's this became a big problem. How do we interface those 10s of sensors? In comes I2C, a well-known hardware communication protocol, which uses just two wires and can talk to 128 individual devices. Now we didn't have an appetite to use existing C or python based libraries to talk I2C. Instead we wanted to write our own in golang. To get us started we got hold of an incomplete implementation of the protocol written in golang, which we picked up and started developing on. Thus came our first library solely containing I2C implementation. The trend continued by writing more and more such libraries right from the basic GPIO operations to interfacing with various sensors available in the market. As theoretically predicted, few of the libraries worked without any code change on beagle bone, another awesome prototyping board based on Linux.  After this, many "nights" were spent to make the library smart enough to work on any Linux based prototyping boards without any code change.  Thus was born the superheroic framework "EMBD".

To get you started with EMBD, lets cook up a famous "hello world" (of the hardware world) recipe. 

###Ingredients Required
* 1 EMBD
* 1 led
* 1 Raspberry pi
* Few wires
* Breadboard
* Fire extinguisher (incase you decide to short the + and -)
* 1 Sublime Text Editor (personal favorite)
 
At the end of the article is a video depicting the complete process. But i'd recommend to read through before jumping to it. I'll not be talking much about the language itself here, but you can refer this to get you started with golang. One of the most compelling reasons for choosing golang for EMBD was its cross compatibility. ARM being one of the fully supported architectures, it suits well for most of the prototyping boards. Meaning you can code on any golang development friendly machine and then build for any target. Follow the instructions here to setup golang on your machine.
The all important hardware setup
Before you start with coding, lets see how to setup the hardware. In any project where you use a prototype board, its a good practice to keep the board and the external circuit separate. So we'll be using a breadboard to connect our led to the pi. Might seem like an overkill, but its "good practice". Fix the led on the breadboard and before you do so make a note of the -ve and +ve terminals. (How you ask? the longer of the leads is +ve). Connect the -ve side to ground pin(3rd from top right on the pin header), and the +ve to the 4th pin from top left of the pin header. This is pin 4, which will be driving the led. Now power up your pi (via USB or external source), and get an ssh access to it on your host machine.

###Some Electronics
At the core of the pi lies a ARM11 processor. But the processor alone does not provide many peripherals which are a must for any prototyping board. This is done using BCM2835 ARM Peripherals chip. It provides all the awesome features like SPI, I2C, GPIO, HDMI, USB etc. We'll be using the GPIO feature here. What's GPIO?  General Purpose Input Output. Simply put, it exposes certain pins on the board whose behavior can be controlled. Behaviors like the pin acting as an output and giving +3.3V(logic high) or 0V(logic low) the voltages vary according to boards.

This example tries to light up a led (light emitting diode) connected to a pin on raspberry pi. What does it take to light up led's?.... a lil EMBD flare..

###Lets code...
Create a led_blink.go file and copy the code as shown below. I'd recommend using Sublime Text + GoSublime package.

```go
package main
 
import (
	"fmt"
	"time"
 
	"github.com/kidoman/embd"
	_ "github.com/kidoman/embd/host/rpi"
)
 
func main() {
	if err := embd.InitGPIO(); err != nil {
		panic(err)
	}
	defer embd.CloseGPIO()
 
	ledPinNum := 4
 
	led, err := embd.NewDigitalPin(ledPinNum)
	if err != nil {
		panic(err)
	}
	err = led.SetDirection(embd.Out)
	if err != nil {
		panic(err)
	}
	defer led.Close()
 
	for i := 0; i < 20; i++ {
		time.Sleep(1 * time.Second)
		fmt.Printf("setting led at %v to high\n", ledPinNum)
		led.Write(embd.High)
 
		time.Sleep(1 * time.Second)
		fmt.Printf("setting led at %v to low\n", ledPinNum)
		led.Write(embd.Low)
	}
}
```

Al right, we have the code. Lets see what it does in more detail.

Here all the necessary packages are imported, embd being the most important. embd also provides packages for hosts like rpi and bbb. In this example we import the rpi package. The underscore tells go to import this even though its not directly used in the code.

####_embd.InitGPIO()_
This tries to get the board info. Why you ask? EMBD supports multiple prototyping platforms. The same code can work on a different LINUX based boards like radxa, beagle bone etc.  The board info is then used to access the corresponding drivers for GPIO control

####_embd.CloseGPIO()_
This command handles the very important clean up once the program exits.

####_embd.NewDigitalPin(ledPInNum)_
This is where all the magic happens. In LINUX everything is a file or dir. And so is a pin. This command uses a gpio exporter to access a directory (/sys/class/gpio/gpio4 in this case). This directory corresponds to the physical pin on the board, and has various files for supported pin functionalities. It returns an interface (led in this case) which has methods to control the behavior of the pin. It returns an error if any.

####_led.SetDirection(embd.Out)_
This is one of the functions implemented by "led" interface that sets up the pin to be a output or input. _**embd.Out**_ is used when you want to drive anything connected to the pin (led in this case), and _**embd.In**_ is used when the input has to be read (eg: reading a switch state). 

####_led.Write(embd.High) / led.Write(embd.Low)_
This functions writes 1 or 0 to the value file within the pin's directory. This results in the corresponding pin giving a HIGH (3.3V) or 0V output respectively. _**embd.High**_ and _**embd.Low**_ are a types referring to 1 and 0 respectively.

Now that you know what is happening, lets build the code so that our awaiting led gets its deserved light. As mentioned earlier, go supports cross compiling. A Raspberry pi has ARM architecture and LINUX os, both supported by golang. To build a executable you need to set the variables GOARCH and GOOS on your terminal as follows.

```bash
GOARCH=arm GOOS=linux go build led_blink.go
```

This should dump a led_blink executable in the same directory as the code. scp this executable onto your raspberry pi and run it as sudo user.

```bash
scp led_blink pi@<ip>:~
sudo ./led_blink
```

Voila..!!!!! The led springs up to life blinking every second. There's your  first ever implementation of embedded programming using EMBD.

{% vimeo 93421433 %}

###What's Next..??
Take further look at EMBD, and find more exciting samples to experiment on. What do you know...? One day you might be writing a new feature yourself and send in the pull requests. We are waiting...


