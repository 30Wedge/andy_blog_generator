---
title: "2. Gathering our tools"
date: 2022-04-04T16:30:45-04:00
draft: true
---

This post goes over the tools (software and hardware) to gather before we can get a minimal bare metal project running.

I'll list out the software and hardware components I'm using.

For each of them, I'll
 - Explain what the component does in general.
 - Describe the specific component I'm using
 - Then show how to find and install the component.

### Hardware

Here's the hardware compeonts I'll reference for the first part of this article. And really, its only one part!

#### The Target Microcontroller -- the MSP430

This Texas Instruments MSP430G2452 is the microcontroller that I have will run the code in this series. 

![attached_hw_debug_probe]( /posts/bare-metal-bringup/msp_430.jpg "Isolated microcontroller")

This little plastic and metal caterpillar-looking thing contains all of the circuitry that will run our software. 

This chip can clip into a devkit (TI brands it as a "LaunchPad") to make it more convenient to develop on.
I removed it from the devkit to take a picture, but for the rest of the project I'll keep it clipped in.  
The devkit provides power to the micocontroller from the USB connector, pin headers, buttons, LEDS, and a built in debug
probe (more on this in the next section).

![attached_hw_debug_probe]( /posts/bare-metal-bringup/launchpad.jpg "Microcontroller plugged into the dev board")

For those following along, the target hardware doesn't 100% matter.
I'll still explain the steps to get a project building that should apply for most suppliers.

 You should be able to keep going with any target hardware as long as it checks these boxes
  - [ ] You can prove the right power the part. (Just about any devkit with a USB cable checks this box)
  - [ ] We can access the debug interface to the chip. We'll need this to load code and interract with the chip (more on
 what that means in the next section).
  - [ ] We have access to all the required software tools specific to this part, whether the tools are open source or proprietary
 (see the Software section).

I bought this particular devkit on EBay a few years ago.  
If there aren't any on EBay, you should be able to order one from the official [TI site](https://www.ti.com/tool/MSP-EXP430G2ET#included).

#### Debug Probe -- ... Also the MSP430 (?)

You want to write software on your development machine, then program it onto your target microcontroller.
The problem here is that while your development machine has a USB port to interface with hardware, your target
microcontroller doesn't speak the USB protocol to allow programming or debugging.
Your microcontroller probably supports a protocol like SWD (serial wire debug) or ISP (In System Programming) to allow
software to be updated to it.
See [this nice table](https://mtm.cba.mit.edu/2021/2021-10_microcontroller-primer/#programming-protocol-reference-table) for look at what the different protocols are and their connectors look like.

The **debug probe** is the piece of hardware that interfaces between your development machine and microcontroller to
allow for programming.
Without one you won't be able to load any of the software that you write onto your target device, so its pretty
important to have.

Its a good thing that most devkits come with a debug probe built in.

![attached_hw_debug_probe]( /posts/bare-metal-bringup/hw_debug_probe.jpg "hw_debug_probe")


The circled part of the devkit board contains some circuitry that makes up a debug probe. So that's what I'll be using.

Alternatively, you could use a general purpose debug probe like a Segger J-Link which is versitile enough to connect to
just about any target that supports a JTAG or SWD interface (or maybe others).
Those can get quite expensive, and offer more features than we need for this project.

#### Optional -- Some exta hardware to validate your program -- like a cheap logic analyzer

Note - for those following along you wont need this!  
The launchpad includes some LEDs and buttons on the board -- that will suffice to validate that programming has
succeded.


As we go along, we'llY write some software, attempt to compile it, and load it onto the target. 
The problem is knowing whether it works or not.
You need some observable output from your program to show you that its working.

Traditionally, you'd connect an LED to one of the GPIO pins and write simple code that blinks it on and off -- in
jargon, that test code is called a "blinky". 

I'm going to do something similar with the built-in LEDs, but I'll also connect a logic analyzer to the GPIO pin that's
controlling the LED.

A logic analyzer can graph the digital level of a GPIO pin over time. 
I'm using this because it makes nicer pictures to include in a written post than an LED :) 

This is among the cheapest logic analyzers I could find that I bought on
[Sparkfun](https://www.sparkfun.com/products/18627) for about $20. 

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

#### Cross Compiling Toolchain - gcc

The word **toolchain** is a specific term for embedded - and it doesn't seem to have a standard definition. 
I'll use it to mean "the collection of all of the programs that are used to convert the code we write into a format that the target
microcontroller can execute."

This sounds a lot like a compiler. And a compiler is one of the essential pieces of a toolchain. But there's also the
assembler, the linker

I'm using a version of **gcc** that supports MSP430 for this specific project. 
 - explain what the tool is
 - explain the specific program
 - I'm using a **gcc** variant for the toolchain for this project.
 - How to find and install

gcc is an open source compiler collection, and there are all sorts of variants available that support different target
architectures.

I'm using the toolchain found on [TI's official site](https://www.ti.com/tool/MSP430-GCC-OPENSOURCE), and I decided to
build the toolchain from source.
That's a hassle worth another post.

If you're following along on ubuntu, you should be able to get the appropriate toolchain from apt with `apt-get install
gcc-msp430`.

#### Debugger Client - gdb
Sometimes the debugger is included as part of the toolchain -- and it is bundled with TI's toolchain, but I'm listing it separately here because its a more independent piece.

Here, specifically, I'm referring to the **debugger client**.
For developing software that runs on the same machine as the development machine, the debugger 

I'm using **gdb** as my debugger for this project.

Like gcc, gdb is open source and there are many variants to support different target architectures. Its worth noting
that gdb and gcc are completely separate programs, despite the similar names.

 - explain what the tool is
 - explain the specific program
 - How to find and install

##### Code Loader - the alternate option
#### Debugger Server - mspdebug

 - explain what the tool is
 - explain the specific program
 - How to find and install


For an official supported version:
`sudo apt install mspdebug`
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
