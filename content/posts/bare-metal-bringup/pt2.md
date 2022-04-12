---
title: "2. Gathering our tools"
date: 2022-04-04T16:30:45-04:00
draft: true
---

This post goes over the tools (software and hardware) to gather before we can get a minimal bare metal project running.

I'll go over the pieces of hardware I'm using, then list out the essential software components, explain what they do,
choose specific programs, then show how to find
and install them.

### Hardware

Here's the two pieces of hardware I'll reference for the first part of this article.

#### The Target Microcontroller -- the MSP430

This Texas Instruments MSP430G2452 is the microcontroller that I have will run the code in this series.
The picture above is just the chip, but its part of a larger TI launchpad devkit system.
The launchpad provides a USB power supply, a debug probe (more in the next section), and some pin headers, buttons and
LEDs that we can use to test our code quickly.

This is the oldest microcontroller I could find in my closet (the sticker on my box says 2012 - as I'm writing this in 2022).

For your understanding, the target hardware doesn't 100% matter.
I'll still explain the steps to get a project building that should apply for most suppliers.

Here's the bare bones checklist I ran through for this part: 
 - [ ] This board has the chip we're developing on it.
 - [ ] The target chip's IO pins exposed on the board.
 - [ ] Something is supplying power to the target chip:
    - The board has some circuitry that uses power from the USB port to power the target chip. Its easy to take this for
    granted.
 - [ ] We can access the debug interface to the chip. We'll need that to load code and interract with the chip (more on
 what that means in the next section).
 - [ ] We have access to all the software tools specific to this part, whether the tools are open source or proprietary
 (see the Software section).

This LaunchPad checks all those boxes.  

Additionally, I chose this part deliberately as an example of a microcontroller that *could* need to be replaced in a
professional project. 
Over the span of a long project, microcontrollers can come and go, as cheaper designs come to market, as scope creep requires
a higher powered core, or as parts are discontinued by the manufacturer. 
The [TI
website](https://www.ti.com/product/MSP430G2452?keyMatch=MSP430G2452&tisearch=search-everything&usecase=GPN#order-quality)
shows that this part is still listed as "Active" (no plans to remove) and in supply. (The chip designer's website is one
of the best online references to use at this stage of a project.) 
This part is already pretty cheap at about $0.73 per chip, and in good supply but its still possible that it could get
swapped out in the future.

For example, maybe your project isn't using all of the chip's resrouces, and it makes financial sense to swap it out for a
newer, cheaper target like the MSP430FR2311 (at $0.52 per chip).

For another example, its possible that the scope could exceed the chip's capabilities and require a
higher powered ARM target like one from the STM32F0 series (at $1.30 per chip).

The point is that *if* its possible for the target chip to change during a project, then a little upfront planning will
help that transition a lot.

Before moving on, if you don't know these terms in the context of embedded software, its good to learn them
 - **chip family**  
 - **architecture**  
 - **chip series**  
 - **embedded target** 

#### Debug Probe -- Also the MSP430 (?)

![attached_hw_debug_probe]( /hw_debug_probe.jpg "hw_debug_probe")

You write software on your development machine, then load it onto your target microcontroller.
The debug probe is the piece of hardware that interfaces the two machines together.
Without one you won't be able to load any of the software that you write onto your target device, so its pretty
important to have.

Good thing that the MSP430 comes with a debug probe built in.


The green circled part contains some circuitry that makes up a debug probe.

Alternatively, you could use a general purpose debug probe like a Segger J-Link which is versitile enough to connect to
all sorts of different targets in all sorts of different circuits.

#### Some hardware to validate the board -- like a cheap logic analyzer

You'll write some software, load it onto your target, but then how will you know that it works?
Ideally you'd like some observable output from your program to show you that its working.

This is among the cheapest logic analyzers I could find that I bought on
[Sparkfun](https://www.sparkfun.com/products/18627) for about $20. 

I'll use it ocasionally to verify that I've compiled code and loaded it onto the target correctly.
Traditionally, you'd connect an LED to one of the GPIO pins and write simple code that blinks it on and off -- in
jargon, that test code is called a "blinky". 
I'm going to do something similar, but using the logic analyzer because its slightly less hardware to use, and it makes
nicer pictures to include in a written post :) 

----

### Software

Here's the minimal checklist of software tools that I'll use to build this project.
These are the minimum that get bundled up toether into an IDE.  
Anything here labeled **user choice** is a software component that is likely to have alternates available.  

 - [ ] debugger client (prefered) OR code loader
    - sometimes **user choice** (sort of)
 - [ ] debugger server
    - sometimes **user choice**
    - **sometimes generic; some are proprietary**
    - The purpose is to interpret between the debugger client running on your PC and the target hardware communicating
    through the debug probe.
 - [ ] cross compiling toolchain
    - **sometimes user choice**
    - Either supplied by the vendor, or supported by open source compilers (clang or gcc)
 - [ ] build system
    - **user choice**
 - [ ] text editor
    - **user choice**

Not required - but very helpful.
 - [ ] vendor library
    - **user choice**
    - Most often these are supplied by the vendor
    - Technically not necessary, but extremely helpful, and still useful in a minimal environment

#### Debugger Client - gdb
##### Code Loader - the alternate option
#### Debugger Server - mspdebug

For an official supported version:
`sudo apt install mspdebug`
#### Cross Compiling Toolchain

##### Why is it and Why do we need it?
The **toolchain** is the collection of programs that will convert your source code into an executable binary file.
The **cross compiling** modifier indicates that the executable binary is in a format that's *different* from the machine
that it was built on (as if it was *cross*ing over from one architecture to another).
In this case, our binaries are going to be built on our x86 laptop, but they're going to be executable on the MSP430.

Why a "toolchain" and not just a "compiler"?  
Although colloquially you might hear some developers say they "compiled the code".
"Compiling" is only one step of several to convert C code to an executable.  
Additionally there are auxillary programs like `nm` that aren't directly involved in the build process, but can be used
to debug problems with the build output.



#### Build System

##### Why is it and Why do we need it?
For a small number of source files and libraries, you don't really *need* a build system.
You could manually invoke the cross compiler tools for every input file whenever its time to build the project (like
we'll do in part 3 of this series).

However, as projects grow more complicated beyond a single source file, you'll either start spending the majority  of
your time on the build, or will need a build system.

This is a program like [Make](https://www.gnu.org/software/make/), [CMake](https://cmake.org/) or
[Ninja](https://ninja-build.org/) that invokes those cross comipling tools automatically for you based on some sort of
script you provide.

I'll introduce CMake as our build system in later posts. I chose it because its among the most popular build systems for
C++ and I appreciate its ability to support cross compiled builds.

##### How to install it

I installed it through `apt` with the command `sudo apt install cmake`.
I followed the instructions on the [cmake site](https://cmake.org/install/)

#### Text Editor

[There's enough written about text editors out there, to the point where I can't add much](https://xkcd.com/378/).
I like to use VSCode, but ultimately it doesn't matter as long as it gets the source code out of your head and into a
file.
