---
title: "Better C++ Register Access"
date: 2025-07-09T13:27:37-04:00
draft: false
---


This post started out as a part of the Real Time C++ Book review, but got
too unwieldy so I moved it out into its own post.

Chapter 7 of "Real Time C++", by by Christopher Kormanyos outlines a system to
access registers with C++ features.
The core of the feature is the
[reg_access](https://github.com/ckormanyos/real-time-cpp/blob/master/ref_app/src/mcal/mcal_reg_access_static.h#L20-L24)
template class with an interface of only static functions. The book's
definition looks sort of like this:
```C++
	template<typename RegisterAddressType,
		typename RegisterValueType,
		const RegisterAddressType address,
		const RegisterValueType value = static_cast<RegisterValueType>(0U)>
	struct reg_access_static final
	{
		static RegisterValueType reg_read() { volatile RegisterValueType* pa = reinterpret_cast<RegisterValueType*>(address); return *pa; }
		static void reg_write() { volatile RegisterValueType* pa = reinterpret_cast<volatile RegisterValueType*>(address); *pa = value; }
		// ... Other access functions
	}
```
The objective of the class is to abstract memory-mapped register access for any target microcontroller into the same interface.
For example, you can use interface to access an 8-bit register in 16-bit address space like [this:](https://github.com/ckormanyos/real-time-cpp/blob/master/ref_app/src/mcal/atmega4809/mcal_gpt.cpp#L142)
```C++
	const std::uint8_t rtc_cnt_2 = mcal::reg::reg_access_static<std::uint16_t, std::uint8_t, mcal::reg::rtc_cnt_address>::reg_read();
```

All of the C++ namespacing and templating makes the syntax a little foreign to a C programmer.
Although the above C++ snippet is functionally equivalent to the below C snippet for accessing a register.

```C
#define REG_READ(addr) (*(uint8_t volatile*)(addr))
//...
uint8_t reg_value = REG_READ(rtc_cnt_addr);
```
The C++ snippet encodes a little more type information than the C snippet, and takes advantage of
C++ improved type casting.

I think I can even get more milage out of C++ compile-time features though.

### Improving on the book's reg_access class
I didn't like how much copy-paste is required for each register access.
Also we have access to C++20 since the book was published.
C++20 introduces
[concepts](https://en.cppreference.com/w/cpp/header/concepts.html), which seem like a good idea for sanity-checking *every* template
class's arguments, and [new aggregate initialization syntax](https://en.cppreference.com/w/cpp/language/aggregate_initialization.html), which makes initializing
POD structs more appealing.

So I came up with a modified `reg_access` class like the following:
```C++
#include <concepts>

namespace reg
{
template <typename RegisterAddressType,
		typename RegisterValueType,
		RegisterAddressType const address,
		typename RegisterBitfieldType = RegisterValueType>
	requires std::unsigned_integral<RegisterAddressType>
		|| std::unsigned_integral<RegisterValueType>
struct reg_access_static final {
	using bits_t = RegisterBitfieldType;
	static RegisterValueType reg_read(void)
	{
		RegisterValueType volatile* pa
				= reinterpret_cast<RegisterValueType volatile*>(
						address);
		return *pa;
	}
	static void reg_write(RegisterValueType value)
	{
		RegisterValueType volatile* pa
				= reinterpret_cast<RegisterValueType volatile*>(
						address);
		*pa = value;
	}
	/* Return a pointer to bitfield type for simplified read/modify/write */
	static RegisterBitfieldType volatile* bits()
	{
		return reinterpret_cast<RegisterBitfieldType volatile*>(
				address);
	}
};
} // namespace reg
```
This works a little differently than the book's version.
Instead of using the class by writing out the whole `reg_access_static` instantiation with
each template parameter specified every time you access a register, you would write an alias
classes for each register *once*, then use that alias to access the register.
I thought this is better because it binds the register data types and literal
address together in the alias type, which cuts down on line length. Also it changes
the `value` parameter to `reg_write` from a constant template parameter to a
function parameter so  it can be set dynamically.

Here's an example of what those specializes aliases of the `reg_access_static`
class would look like.
I think its a good idea to put these aliases in a separate file within a namespace for
the target hardware.
I prefer the namespace strategy to only introduce namespaces to avoid name
clashes, therefore keeping nested namespaces relatively flat.
For this example, I'm choosing a STM32G041xx microcontroller as target hardware,
and defining registers for a few STM32 GPIO registers:

``` C++
#include <cstdint>

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

Then your application can access those registers through the registers aliases
like the following:

```C++
// Read-modify-write update the gpio A0 mode field
uint32_t mode_val = stm32g041xx::gpio_a_mode::reg_read();
mode_val &= ^0x3u;
mode_val |= 0x1u;
stm32g041xx::gpio_a_mode::reg_write(mode_val);

// Write GPIO A1 and A3 OSpeed = 2
stm32g041xx::gpio_a_ospeed::reg_write(0x88u);
```

### More structure with packed bitfield register definitions

I also added an optional feature to define a bitifeld struct for a register.

Now be warned, using [bitfield register access relies on compiler-specific
behavior](https://blogs.sw.siemens.com/embedded-software/2019/12/02/why-not-use-bit-fields-for-device-registers/).
This is often a **"bad thing"** and plenty of developers have justified reasoning
to avoid compiler-specific behavior.

Code like this might fail in subtle ways if you switch compilers. If you do choose to use it, then you take on the
responsibility of learning your compiler's specific layout rules. For example,
[ARM specifies](https://developer.arm.com/documentation/dui0067/d/arm-compiler-reference/c-and-c---implementation-details/structures--unions--enumerations--and-bitfields?lang=en)
that its compilers lay out packed bitfields in target endian order (for
example, on a little endian target, the first struct field occupies the lowest
address in memory) with an alignment of 1. [GCC defers its behavior to the ABI
spec](https://gcc.gnu.org/onlinedocs/gcc/C-Implementation.html).

However if you accept those conditions, then this syntax can be very nice to
work with.
Using these bitfield structs leverages type system to bind register field
definitions to their relevant register which
reduces another source of mistakes and helps make the code self-documenting.

Here's revisiting the STM32 GPIO register example, but this time defining
`RegisterBitfieldType`s for a few of the registers.
``` C++
#include <cstdint>

namespace stm32g041xx
{
constexpr uint32_t gpio_a_base = 0x50000000UL;

typedef struct __attribute__((packed)) gpiox_moder_t {
	union {
		struct {
			unsigned int mode0 : 2;
			unsigned int mode1 : 2;
			unsigned int mode2 : 2;
			unsigned int mode3 : 2;
			unsigned int mode4 : 2;
			unsigned int mode5 : 2;
			unsigned int mode6 : 2;
			unsigned int mode7 : 2;
			unsigned int mode8 : 2;
			unsigned int mode9 : 2;
			unsigned int mode10 : 2;
			unsigned int mode11 : 2;
			unsigned int mode12 : 2;
			unsigned int mode13 : 2;
			unsigned int mode14 : 2;
			unsigned int mode15 : 2;
		};
		uint32_t raw;
	};
};

struct __attribute__((packed)) gpiox_ospeedr_t {
	union {
		struct {
			unsigned int ospeed0 : 2;
			unsigned int ospeed1 : 2;
			unsigned int ospeed2 : 2;
			unsigned int ospeed3 : 2;
			unsigned int ospeed4 : 2;
			unsigned int ospeed5 : 2;
			unsigned int ospeed6 : 2;
			unsigned int ospeed7 : 2;
			unsigned int ospeed8 : 2;
			unsigned int ospeed9 : 2;
			unsigned int ospeed10 : 2;
			unsigned int ospeed11 : 2;
			unsigned int ospeed12 : 2;
			unsigned int ospeed13 : 2;
			unsigned int ospeed14 : 2;
			unsigned int ospeed15 : 2;
		};
		uint32_t raw;
	};
};

using gpio_a_mode = reg::reg_access_static<uint32_t,
		uint32_t,
		gpio_a_base,
		gpiox_moder_t>;
using gpio_a_ospeed = reg::reg_access_static<uint32_t,
		uint32_t,
		gpio_a_base + 0x8UL,
		gpiox_ospeedr_t>;
/* emitting the other registers to keep the snippet short */

} // namespace stm32g041xx
```

Now the above example application snippet can be written like the following.
Much easier to understand which register fields are being modified right?
```C++
// Read-modify-write update the gpio A0 mode field
uint32_t mode_val = stm32g041xx::gpio_a_mode::bits()->mode0 = 1u;

// Write GPIO A1 and A3 OSpeed = 2
// note: this is "aggregate initialization" syntax introduced in C++20
stm32g041xx::gpio_a_ospeed::bits_t const a_speed{
	.ospeed1 = 2u,
	.ospeed3 = 2u,
};
stm32g041xx::gpio_a_ospeed::reg_write(a_speed.raw);
```

### Going further?
Could it be better? of course!

You could add template constraints to implement read-only and write-only
registers by encoding that information in the `RegisterAddressType` type.
Then you could write a constraint to raise a compile error if `reg_write` or
`reg_read` are used on a read-only or write-only register accordingly.

You could also implement overloads for `reg_read` and `reg_write` to use
`RegisterBitfieldType` types directly. That would make those functions a little
type-safer when paired with bitfield registers, and remove the need for the
union with the `raw` field. I avoided doing this because, because you would
have to define volatile copy assignment operators for each register bitfield. I
found that defining register types with the union-ed `raw` field was more
straightforward to use, than what I could come up with.

### Testing
I made sure the above examples all compile down to assembly equivalent to the
following.
```ASM
	// Load GPIOA::MODE address in r2
	movs	r2, #160
	lsls	r2, r2, #23
	// Read GPIOA::MODE register value
	ldr	 r3, [r2]
	// Set lower 2 bits (GPIOA::MODE0) to 0x1 in register value
	movs	r1, #3
	bics	r3, r1
	subs	r1, r1, #2
	orrs	r3, r1
	// Write modified value back to GPIOA::MODE
	str	 r3, [r2]
	// Load GPIOA::OSPEED address
	ldr	 r3, .L5
	movs	r2, #136
	// write 0x88 to GPIOA::OSPEED register
	str	 r2, [r3]
.L2:
	// GPIOA::OSPEED adddress
	.word   1342177288
```

I compiled with the C++ 32b ARM GCC 11.2.1 (none) compiler with
the `-Og` flag for optimization, `-std=c++20` flag to lock down the standard,
and `--specs=nano.specs  -mfloat-abi=soft -mthumb  -mcpu=cortex-m0plus -mthumb`
flags for STM32G041xx target support. [Godbolt.org](https://godbolt.org/) was a
huge help for analyzing these snippets.
