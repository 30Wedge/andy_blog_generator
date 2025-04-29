---
title: "Real Time C++ Book Review"
date: 2025-04-15T12:09:39-04:00
draft: true
---

## Summary

"Real-Time C++," by Christopher Kormanyos [[1]](#references), is one of the rare intermediate level software books for embedded software.
It assumes that the reader has foundational knowledge in both C++ and microcontroller(MCU) programming.
Given a foundation in both subjects, I think that reading this book can effectively put some new tools in the reader's tool belt.

A good reader will need context awareness and critical thinking to get the most out of this book.
Kormanyos has a mastery of embedded math software.
From his public github alone [[2]](#references), you can see that he's published a number of his own C++ math implementations and contributed to high profile projects like Boost.
In this book, he works with a very narrow meaning of the famously nebulous [[3]](#references) term "Real-Time software": low latency math-heavy computation enabled by clever algorithms.
As a result, the 3 parts of the book cumulatively build up a technical understanding in order to write clever math-heavy algorithms.
To his credit, he follows up efficiency claims with time and program space measurements.
However some of the arcane C++ wizardry that enables these high performance algorithms come with subtle and serious design tradeoffs that
aren't going to be obvious to all readers and aren't explained.
For example, when given some out of bounds data, some of the functions listed will happy return a wildly incorrect result.
I would have liked to swap out the few chapters on multitasking (which would be better fleshed out in a separate book) for chapters on bounds checking an error handling.
Perhaps discussing those tradeoffs are genuinely out of scope of the book.

## Part 1 - Language Technologies for Real-Time C++

This part of the book lists out foundational skills in C++, embedded software and performance-oriented software.
I think this section should be read like a grocery list to make sure that you have what you need before you start cooking your meal in parts 2 and 3.
This section is full of sample listings.
Read through each sample code and make sure you understand exactly what's going on.
Odds are that if there's something in a sample that you don't understand, then you can get a working knowledge of the feature from the chapter text, but you'll need an external reference to really grok the feature.

I first read this book as a junior engineer with solid embedded skills and weak C++ skills; so this helped me fill some knowledge gaps.
Unnamed namespaces [[4]](#references), `static_assert` [[5]](#references) and `std::chrono_literals` [[6]](#references) are a few of the language features I picked up from this book.
Later on I read a much more thorough book on C++ [[7]](#references), which I'd now recommend doing *before* reading "Real-Time C++."
There are pitfalls that new and experienced C++ programmers can fall into without paying close attention.
Specifically, chapters 1, 3, 4, 5 and 6 consist of punchy sections with the following ingredients:
 1. Introduce a C++ feature.
 2. One or two code samples to showcase the feature.
 3. A few paragraphs with a declarative description of the feature.  

My main problem with this section is that when reading the descriptive paragraphs, I often had trouble discerning if it described *everything* about the given feature, or if there is more information that I need to understand before using.
Some language footguns that I thought are important were glossed over until a later section of the book.
Also, these sections can make some broad subjective statements.
When combined as a whole, these have the potential to lead a junior developer to draw the wrong conclusions and make an absolute mess of the codebase.
For example: Section 6.8 "Use Comments Sparingly" combined with Section 6.20 "Use Template Metaprogramming to Unroll Loops" makes me quite wary.
Especially since the book doesn't mention that often compilers have `#pragma unroll` directives that can perform the same function in a single line without giving SFINAE a thought.

For an example of a glossed-over footgun: Section 4.11 states "There are no added runtime or storage costs associated with qualifying member functions as `const`."
This is completely true.
Section 3.11 similarly states "`std::array` can be used as a drop-in replacement for C-style arrays" which is true in many cases, but not all.
One missing caveat is that `std::array` will not decay to a pointer type.
A C programmer(me) might be accustomed to passing arrays as pointer-length arguments to functions like `unsigned do_some_linear_array_operation(unsigned const* array, size_t array_len);`.
That C programmer might be tempted to port that function to `std::array` like this: `template <size_t LEN> unsigned do_some_linear_array_operation(std::array<unsigned, LEN> const& array);`.
The pitfall here is that the compiler will happily instantiate one copy of this function for each unique array length passed to it, unless it is able to optimize away the copies.
Instead, that C programmer could have ported the function using iterators like `template <typename InputIt> unsigned do_some_linear_array_operation(InputIt first, InputIt last);` to cut down on one instantiated copy per iterator type.
Kormanyos does cover this problem in greater detail in chapter 16 "Extending the C++ Standard Library and the STL", where he shows how to roll your own STL data structures and algorithms in more detail.

As far as "optimized" programming goes, Kormanyos's recommendations are mostly sound.
I've heard the myth that virtual functions use high performance overhead.
Section 4.5 dismisses that idea in general, but acknowledges cases where their light overhead might be too much.
Chapter 6 then lists some generally good tools for performance programming that could be categorized into the following bins:

 1. Measure your code.
 2. Do as little computation at runtime as possible.
 3. Tailor your code to your hardware.
 4. Give your compiler what it needs to generate optimized code.

"Measurement" breaks down into measuring the runtime of your code, as well as studying the compiler's generated assembly and RAM/ROM usage.
Throughout the book, efficiency claims are backed up by tables showing runtime and memory usage on different target hardware, which is an important skill.
I think the ability to run your own performance experiments is the most important performance skill of all.
You can test any performance claim with hard proof, and experiments enable future learning.
Kormanyos doesn't explicitly say how and why to measure runtime until Section 9.6 though.

Tools to reduce work at runtime encompass C++ techniques like `constexpr` as well as making good big-O algorithm choices.
I appreciate some especially embedded-focused techniques like using bitshifts instead of multiplication/division if possible.

Hardware-specific optimization techniques are centered around processor capabilities.
For example, 64-bit integer operations are going to be a lot more expensive on an 8-bit processor than a 64-bit processor.
I think this section does a good job assigning relative importance to each technique and pruning extraneous information.
For example, it can be tempting to think that hand-written assembly will make your program faster, but this section stresses that this should only be a last resort after making other easier changes (like compiler flags, algorithm choice, etc.).

## Part 2 - Components for Real-Time C++

Part 2 moves above language fundamentals and moves into writing idiomatic C++ building blocks for embedded code.
Throughout this part of the book, the content becomes more abstract.
This helps showcase real applications for these C++ features which is helpful to see uses for them in action in real embedded code.
Korymanyos writes out implementations of the "MCAL" (MicroController Abstraction Layer) interface that gets referenced throughout the book.
He says this interfaces was inspired by AUTOSAR standards [[8]](#references), which is a practical in-use by industry.
If you're not satisfied, then there's an extraordinary amount of similar code online that you can reference [[9]](#references).

Like all nontrivial abstractions, the components in this part come with implied usage constraints and leak implementation details[[10]](#references).
There's not a lot of discussion about what information "leaks" through the abstraction, and the constraints that come with each example.
That being said, I think that's OK to leave those thoughts to reader, and focus the book on showcasing embedded C++ code in action.

I'll dig into only one of the abstractions to suggest some improvements then move on.

Chapter 7 outlines a system to access registers with C++ features. The core of which is the [`reg_access`](https://github.com/ckormanyos/real-time-cpp/blob/master/ref_app/src/mcal/mcal_reg_access_static.h#L20-L24) template class with an interface of only static functions. The definition looks like this:
```C++
	template<typename RegisterAddressType,
		typename RegisterValueType,
		const RegisterAddressType address,
		const RegisterValueType value = static_cast<RegisterValueType>(0U)>
	struct reg_access_static final
	{
		static RegisterValueType reg_get() { volatile RegisterValueType* pa = reinterpret_cast<RegisterValueType*>(address); return *pa; }
		static void reg_set() { volatile RegisterValueType* pa = reinterpret_cast<volatile RegisterValueType*>(address); *pa = value; }
		// ... Other access functions
	}
```
The objective here is to abstract memory-mapped register access for any target microcontroller into the same interface.
One example of using this interface to access an 8-bit register in 16-bit address space looks like [this:](https://github.com/ckormanyos/real-time-cpp/blob/master/ref_app/src/mcal/atmega4809/mcal_gpt.cpp#L142)
```C++
    const std::uint8_t rtc_cnt_2 = mcal::reg::reg_access_static<std::uint16_t, std::uint8_t, mcal::reg::rtc_cnt>::reg_get();
```

All of the C++ namespacing and templating makes the syntax a little foreign to a C programmer, although it is functionally equivalent to the below C syntax for accessing a register.
```C
#define REG_READ(addr) (*(uint8_t volatile*)(addr))
//...
uint8_t reg_value = REG_READ(rtc_cnt_addr);
```
The C++ version adds some extra type information to use C++'s improved type casts.

I'm a little confused why this interface is through a static class and not discrete functions.
Also I didn't like how much copy-pasta is required for each register access.
Asking the user to enter the address type, value type and address together correctly every time introduces more opportunities for bugs.
I think it could be improved a bit to use the C++ type system to bind the register data types and literal address together.

I'd change the `value` from a constant template parameter to a function parameter so that it could be set dynamically too.
So first I would modify the `reg_access` class like the following:
```C++
	template <typename RegisterAddressType,
			typename RegisterValueType,
			RegisterAddressType const address>
	struct reg_access_static final {
		static void reg_set(RegisterValueType value)
		{
			RegisterValueType volatile* pa = reinterpret_cast<
					RegisterValueType volatile*>(address);
			*pa = value;
		}
	};
```

Next, I'd make a separate file with a namespace for my target hardware.
Then you could define all the registers you want to access with type aliases to a specialized version of the `reg_access_static` class.
Here's an example for a few STM32 GPIO registers:
``` C++
namespace stm32g041xx
{
	constexpr uint32_t gpio_a_base = 0x50000000UL;

	using gpio_a_mode = reg::reg_access_static<uint32_t,
			uint32_t,
			gpio_a_base>;
	using gpio_a_otype = reg::reg_access_static<uint32_t,
			uint32_t,
			gpio_a_base + 0x4UL>;
	using gpio_a_ospeed = reg::reg_access_static<uint32_t,
			uint32_t,
			gpio_a_base + 0x8UL>;
	using gpio_a_pupd = reg::reg_access_static<uint32_t,
			uint32_t,
			gpio_a_base + 0xCUL>;

} // namespace stm32g041xx
```

This keeps the register type and address information encapsulated.
Your application can access those registers through the aliases like the following:

```C++
// Read-modify-write
uint32_t mode_val = mcal::stm32g0::gpio_a_mode::reg_get();
mode_val |= 0x1u;
mcal::stm32g0::gpio_a_mode::reg_set(mode_val);

// Constant write
mcal::stm32g0::gpio_a_ospeed::reg_set(0x88u);
```

I made sure the above compiles down to the following minimal assembly.
For reference, I compiled with 32b ARM GCC 8.5.0 with the `-0g` flag.
Godbolt is a great tool for analyzing compiler behavior for snippets like this [[11]](#references)
```ASM
        mov     r3, #1342177280
        ldr     r2, [r3]
        orr     r2, r2, #1
        str     r2, [r3]
        mov     r2, #136
        str     r2, [r3, #8]
```

## Part 3 + Appendices - Mathematics and Utilities for Real-Time C++

Part 3 gets interesting, if a little bit academic at times. We get deep dives
into heavy hitting math, as well as a step-by-step guide to extend the standard
library.

After a short introduction to floating point numbers, the first chapter quickly
gets into math that goes over my head. I had never heard of the "Cylindrical
Bessel Function" before reading this chapter, but now I've seen a C++
implementation of one!

Reading in between the lines of impressive mathematics will get you a general
approach to calculate these fancy functions with some performance-mindedness.
Overall, the function is going to have a triple-constraint of precision,
runtime and code size. Compile-time computation, Taylor series and other
approximations in the chapter are tools that can help bring down the run time.
FPU-enabled hardware can only do good (unless you add hardware cost as a constraint).
One interesting gotchya is that sometimes math techniques can simplify a complex
function to a recursive function, which throws me back to taking CS college
classes and calculating big-O complexity.
Its interesting that it can be a competetive design choice for the engineer to
decide whether its better to have a function `f(x)` that is O(1), but takes
longer for most values of x than a reduced function that is `O(N)` where
runtimes scale with the input.

I only have two criticisms in the math chapters: 1. is what I mentioned in the
introduction, that there's only a small mention of how to handle arguments out
of range, because int the projects I've worked its been pretty important to
guarantee a function's correctness. And 2. some of the listings use a very
terse, clever, mathematical coding style. I realize it probably helps fit into
a printed page, but IMO when working with a team this kind of code can become a
liability if not well documented about *why* its like this, or. For example I
counted a single line of code in Section 13.4 that initializes a variable with
an expression containing 21 operators.

One of the chapters in this section covers how to write or extend the STL. It
turns out there's a *lot* required to write a STL-conformant container class.
The listing with the header definition for a custom class gets quite clunky
since it spans 3 pages in my print book. However I was impressed by Kormanyos's
ability to cover what's important.
This part of the book certainly benefits from his contributions to Boost.

The final chapters go back to Part 1's style of a cursory tour of C++ features.
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
