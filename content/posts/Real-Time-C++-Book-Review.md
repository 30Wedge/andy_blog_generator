---
title: "Real Time C++ Book Review"
date: 2025-04-15T12:09:39-04:00
draft: false
---

## Summary

"Real-Time C++" by Christopher Kormanyos [[1]](#references), is one of the rare
intermediate level books for embedded software. It assumes that the
reader has foundational knowledge in both C++ and microcontroller (MCU)
programming. Given a foundation in both subjects, I think that reading this
book can effectively put some new techniques in the reader's tool belt. Context
awareness and critical thinking will help a good reader to get the most out of
this book.

Kormanyos is an expert at writing portable math software. From his github
account alone [[2]](#references), you can see that he's published a number of
his own C++ math implementations and contributed to high profile projects like
Boost. In this book, he works with a very narrow meaning of the famously
nebulous [[3]](#references) term "real-time software": low latency math-heavy
computation enabled by clever algorithms. As a result, the 3 parts of the book
cumulatively build up a technical understanding in order to write clever
math-heavy algorithms. He follows up efficiency claims with time and program
space measurements, which is excellent. Mathematical reasoning is laid out in a
series of equations like a math textbook. These are the strongest aspects of
the book. However some of the C++ wizardry that enables these high performance
algorithms come with subtle and serious design tradeoffs which might not be
suitable for other types of "real-time software." These aren't going to be
obvious to all readers and often aren't explained in the book. For example,
when given some out of bounds data, some of the functions listed will happy
return a wildly incorrect result. I would have liked to swap out the few
chapters on multitasking (which could be a whole separate book) for chapters on
bounds checking and error handling. Perhaps discussing those tradeoffs are
genuinely out of scope of the book.

## Part 1 - Language Technologies for Real-Time C++

This part of the book rattles off foundational topics in C++, embedded software
and performance-oriented software. I think this section should be read like a
grocery list to make sure that you have what you need before you start cooking
your meal in parts 2 and 3. Much of the section reads like the following:
 1. Introduce a C++ feature.
 2. One or two code samples to showcase the feature.
 3. A few paragraphs with a declarative description of the feature.

Read through each sample code and make sure you understand exactly what's going
on. Odds are that if there's something in a sample that you don't understand,
then you can get a working knowledge of the feature from the chapter text
alone, but you'll need an external reference to really grok the feature.

I first read this book as a junior engineer with solid embedded skills and weak
C++ skills; so this helped me fill some knowledge gaps. Unnamed namespaces
[[4]](#references), `static_assert` [[5]](#references) and
`std::chrono_literals` [[6]](#references) are a few of the language features I
picked up from this book. Later on I read a much more thorough book on C++
[[7]](#references), which I'd now recommend doing *before* reading "Real-Time
C++." There are pitfalls that new and experienced C++ programmers can fall into
without paying close attention, that aren't well marked. And some sections can
mix objective and subjective statements when describing a feature. When
combined as a whole, these have the potential to lead a junior developer to
draw the wrong conclusions and make an absolute mess of the codebase. For
example: Section 6.8 "Use Comments Sparingly" (subjective) combined with
Section 6.20 "Use Template Metaprogramming to Unroll Loops" (one big footgun)
makes me quite wary. Especially since the book doesn't mention that often
compilers have `#pragma unroll` directives that can perform the same function
in a single line without giving SFINAE a thought.

Kormanyos's recommendations are pretty sound for writing performant code.
Section 4.5 addresses the myth that virtual functions have a high performance
hit with a level head. In general they're fine, but he acknowledges cases where
their slight overhead might still be too much. Chapter 6 then lists some
generally good tools for performance programming that could be categorized into
the following bins:

 1. Measure your code.
 2. Do as little computation at runtime as possible.
 3. Tailor your code to your hardware.
 4. Give your compiler what it needs to generate optimized code.

"Measurement" breaks down into measuring the runtime of your code, as well as
studying the compiler's generated assembly and RAM/ROM usage. Throughout the
book, efficiency claims are backed up by tables showing runtime and memory
usage on different target hardware. IMO I think the ability to run your own
performance experiments like this is the most important software performance
skill to have. You can test any performance claim with hard proof; plus
experiments enable future learning. Kormanyos gets into the nitty gritties in
how and why to measure runtime in Section 9.6.

Tools to reduce work at runtime encompass C++ techniques like `constexpr` as
well as making good big-O algorithm choices. I appreciate some especially
embedded-focused techniques like using bitshifts instead of
multiplication/division if possible.

Hardware-specific optimization techniques are centered around processor
capabilities. For example, 64-bit integer operations are going to be a lot more
expensive on an 8-bit processor than a 64-bit processor. I think this section
does a good job assigning relative importance to each technique and pruning
extraneous information. For example, it can be tempting to think that
hand-written assembly will make your program faster, but this section stresses
that this should only be a last resort after making other easier changes (like
compiler flags, algorithm choice, etc.).

## Part 2 - Components for Real-Time C++

Part 2 moves above language fundamentals and moves into writing idiomatic C++
building blocks for embedded code. Throughout this part of the book, the
content becomes more abstract. This helps showcase real applications for these
C++ features which is helpful to see uses for them in action in real embedded
code. Korymanyos writes out implementations of the "MCAL" (MicroController
Abstraction Layer) interface that gets referenced throughout the book. He cites the
AUTOSAR standards [[8]](#references) for inspiration, which is practical for
industry experience. If you're not satisfied, then I've found loads of similar
code for reference reference [[9]](#references).

Like all nontrivial abstractions, the components in this part come with implied
usage constraints and leak implementation details[[10]](#references). The chapters
don't have a lot of discussion about what information "leaks" through the
abstraction and the constraints that come with each example. That being said, I
think that's OK to leave those thoughts to reader, and focus the book on
showcasing embedded C++ code in action.

I was especially interested in Chapter 7 -- using C++ features for better
type-controlled register accesses. With the right abstraction, there's a lot of
room for improvement on register access in C++ over C!

Chapter 9 is especially useful to read too, because it shows an implementation
of an object-oriented SPI interface and its integration in a low-overhead ISR.
Again, the listed code relies on some implicit assumptions that aren't laid out
ad nauseum, but that's okay. This was the first time I've seen a real use for a
`friend` function declaration. Doing so allows the ISR to access class-private
members directly instead of through function interfaces that prevent the
compiler from keeping the ISR tightly optimized.

## Part 3 + Appendices - Mathematics and Utilities for Real-Time C++

Part 3 gets interesting with deep dives into heavy hitting math, as well as a
step-by-step guide to extend the standard library. After a short introduction
to floating point numbers, the first chapter quickly gets into math that goes
over my head. I had never heard of the "Cylindrical Bessel Function" before
reading this chapter, but here it shows how to solve one in C++.

Reading in between the lines of impressive mathematics will get you a general
approach to calculate these fancy functions with some performance-mindedness.
Each function is going to have competing constraints of precision, runtime and
code size. Compile-time computation, Taylor series and other approximation
techniques in the chapter are tools that can help bring down the run time.
FPU-enabled hardware can only help bring down runtime (at the expense of
hardware cost). One interesting gotchya is how sometimes math techniques can
simplify a complex function to a recursive function. Then it can be a
competitive design choice for the engineer to decide whether its better to
implement a function `f(x)` with an `O(1)` algorithm that longer for most
values of `x`, or with a recursive function that is `O(N)` where runtimes scale
with the input.

I only have two criticisms in the math chapters: 1. is what I mentioned in the
introduction, that there's only a small mention of how to handle arguments out
of range, because int the projects I've worked its been pretty important to
guarantee a function's correctness. And 2. some of the listings use a very
terse, clever, mathematical coding style that I don't like. I realize brevity
helps it fit into a printed page, but IMO when working with others this kind of
code can become a liability if its not well documented *why* its written like
this. For example I counted a single line of code in Section 13.4 that
initializes a variable with a very complex expression containing 21 operators.

One of the chapters in this section covers how to write or extend the STL. It
turns out there's a *lot* required to write a STL-conformant container class.
The listing with the header definition for a custom class gets quite clunky
since it spans 3 pages in my print book. However I was impressed by Kormanyos's
ability to cover what's important. This part of the book certainly benefits
from his contributions to Boost.

The appendices go back to Part 1's style of a cursory tour of C++ features.
Its good for getting a lay of the of the land, but not necessarily to master a
features. For example, Appendix A lists all 4 types of casts in C++, but
doesn't explain more than that they're different.

Anyways, if you've read this much of my rambling, then you might as well go buy
the book and read it yourself. Its ambitious enough to be worth a read, and
hopefully you have the critical thinking skills to consider its flaws and the
overall larger pictures.

## References
 - [1] "Real-Time C++: Efficient Object-Oriented and Template Microcontroller Programming", 3rd edition, by Christopher Kormanyos https://link.springer.com/book/10.1007/978-3-662-62996-3
 - [2] https://github.com/ckormanyos
 - [3] Jack Ganssle on "Real Time Programming" in the 90's https://www.ganssle.com/articles/realtime.htm
 - [4] C++ Unnamed namespaces https://en.cppreference.com/w/cpp/language/namespace#Unnamed_namespaces
 - [5] C++ `static_assert()` https://en.cppreference.com/w/cpp/language/static_assert
 - [6] C++ `std::chrono_literals` https://en.cppreference.com/w/cpp/chrono/operator%22%22s
 - [7] "Professional C++", by Mark Gregoire https://www.wiley.com/en-us/Professional+C%2B%2B%2C+5th+Edition-p-9781119695547
 - [8] AUTOSAR Classic-platform https://www.autosar.org/standards/classic-platform
 - [9] Serenity-OS "AK" C++ utilities https://github.com/SerenityOS/serenity/tree/master/AK
 - [10] "The Law of Leaky Abstractions", Joel Spolsky https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/
 - [11] Godbolt https://godbolt.org/
