---
title: "1. What is the Problem?"
date: 2022-04-04T16:30:56-04:00
draft: true
---

### The question that started this series:

> How would one go about working with a board without an IDE?


I Googled this question a little while ago and got to [this Redit
post](https://www.reddit.com/r/embedded/comments/gzj59g/compile_from_scratch_without_ide/) with some quick answers, and
technical tutorials, but
nothing as satisfying as a [deep dive explanation](https://diataxis.fr/).

I put together this series with the goal of answering that question with working examples that explain *how* to do this,
along with sections explaining the reasoning *why* these code samples solve the problem.

 - How do you bring up a bare metal embedded software project without an IDE?
 - How can you structure an embedded software project to change target hardware easily?

I'll start with compiling a blinky program for the MSP430 with CMake and gcc, then porting it to some new (undecided)
hardware.
Then I'll port it to some new hardware like its 2020 and adjust the build system to accommodate. I'll finish the series by showing how to reuse higher-level code shared between targets, and how to write unit tests that exercise that shared code.

### Pros and cons to working without an IDE

Lets dive into the technical stuff though. Other than to satisfy the curiosity at what processes happen between C code in an IDE window
and a running blinky on a devkit, here's the pros and cons I could think of for wanting to work in a minimal dev
environment.

Pros: in one word **flexibility** --
  - You won't be committed to a single chip manufacturer (if done correctly).
  - More flexibility to adopt modern practices like TDD (if done correctly).
  - You won't be committed to a proprietary all-in-one IDe like Keil or IAR.
  - Easier to accomodate different developer preferences to different editors.
 
 Cons: in one word **complexity** --
  - Learning to use these tools can be much more frustrating than learning to use an IDE. The IDE takes care of a lot of
  details for you.
  - Have to be willing to invest the developer-time to developing the build system.
  - There is more room to make mistakes that would hamper or at least limit the advantages of this approach for large
  scale projects.
    - This point hardly matters for small teams or personal projects.

### Other Writing

 - [Vivonomicon's STM32 Baremetal
 Examples](https://vivonomicon.com/2018/04/02/bare-metal-stm32-programming-part-1-hello-arm/)
     - This is the start of a similar, and excellent blog series.
     - This series starts off with getting a project built for STM32, and continues with working with all sorts of
     [other peripherals and external hardware] (https://vivonomicon.com/category/stm32_baremetal_examples/)
     - clearly the best I think.
     - No code companion.
 - https://www.purplealienplanet.com/node/69
     - Another similar, and pretty good blog post. 
     - The writing is brief, and provides examples for a ST board.
 - http://searchingforbit.blogspot.com/2015/06/before-main.html
     - The text is somewhat hard to read, but this slideshow provides neat diagrams on what cross compiling toolchains do.


