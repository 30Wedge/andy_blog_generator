---
title: "Adding Unit Tests to Legacy C Code with GTest and FFF"
date: 2021-11-29T16:43:12-05:00
draft: false
---

## Table of Contents
  * [Introduction](#introduction)
  * [Some background on jargon used in this article](#some-background-on-jargon-used-in-this-article)
  * [Selecting the sample C app](#selecting-the-sample-c-app)
  * [Base App](#base-app)
  * [Why can't the code be tested now? What's the problem with putting it into a test harness now?](#why-cant-the-code-be-tested-now-whats-the-problem-with-putting-it-into-a-test-harness-now)
  * [The road to a compiling test harness](#the-road-to-a-compiling-test-harness)
  * [A simple demo with FFF mocking library calls](#a-simple-demo-with-fff-mocking-library-calls)
  * [Unit testing a function with no library API calls](#unit-testing-a-function-with-no-library-api-calls)

----

### Introduction

The main purpose of this article is to show a concrete example of how to take some legacy software written in C and add unit tests into the project using GTest and fff.

I've set up an companion git repo to go along with the article here: https://github.com/30Wedge/snake-adding-unit-tests-to-legacy-code. This is where I've written the code that I'm describing and taking excerpts from.
Throughout the article, I'll reference different commits from that repo which show the state of the project at each step of the process.

Here's how this article will read:
 - We'll start with a sample C app with no existing tests and some external dependency(ies).
 - I'll show how to break the dependency to enable the app to be compiled with a *mock* dependency for testing
 - I'll show how to add the GTest and fff libraries into the ecosystem so that they can be used to help write unit tests.
 - Then I'll get the app into a compiling unit test
 - Lastly I'll  write a few example tests to show how the test libraries work.

And here's what this article *wont* cover:  
  - *Why* write unit tests?
      - Other people have wrote much more convincingly on this,
  - Pros. and cons. of different unit test libraries out there.
      - I'm writing about these two libraries because this is what I'm most famliar with.
  - How to write *good* unit tests
      - I'm avoiding this because it depends a lot on the application you're working with and on your team's priorities. I'd recomment Michael Feathers’ book "Working Effectively With Legacy Code" for strategies on how to prioritize testing that I use at work.

You can clone the git repo locally and run some git commands or view the different stages of the github repo in the browser to follow along.

I'll reference commits from the companion repo with note boxes in this format:

{{% notice note %}}
Follow along locally with: _some git command to run to check out the commit on your computer_  
View online with: _some link to see the commit on Github_  
{{% /notice %}}

### Some background on jargon used in this article

**Gtest** - "Google Test". An open source test framework for writing C++ unit tests. https://github.com/google/googletest

**fff** - Fake Function Framework (fff). an open source library for mocking C functions for test. https://github.com/meekrosoft/fff

**"mock function"** "Mocks \[functions\] are an implementation that is controlled by the unit test". The words "Mock", "fakes" and "stubs" are often used interchangably, or with varying definitions. I prefer to use this article's differention between the three terms [Memfault on Mocks](https://interrupt.memfault.com/blog/unit-test-mocking#stubs-mocks-and-fakes-review). I'll often use the work **mock** as a verb meaning - to replace a library function wih a mock function.
Within the tests described in this article, we're *only* going to be using mock functions as described here. The dependencies involved are relatively straight forwards, so we can use the same strategy to mock them all with fff.

**Dependency** I'm going to use this [description for what a dependency is from Coder's Legacy](https://coderslegacy.com/what-are-dependencies-in-programming/)
 > if Program A requires Program B to be able to run, Program A is dependent on Program B. This makes Program B a dependency of Program A.  

In this case, Progam A is our `snake` app, and Program B is the `conio.h` library. `snake` can't run without `conio.h` so `conio.h` is a dependency of `snake`.


**Seam** "A seam is a place where you can alter behavior in your program without editing in that place." This is the definition from "Working Effectively with Legacy Code' by Robert Martin. For this article, I'll focus on "link seams" which are places where you can alter the behavior of your program by changing how it is linked at compile time by changing the include path of the build program. 

### Selecting the sample C app

I looked far and wide for a piece of open source code to fit the profile for this project, and here's the app I chose: https://github.com/troglobit/snake .

I'm  think its absolutely perfect for this example because
 - It is written in C.
 - It is digestible for a blog post -- The entire app is less than a thousand lines of code total.
 - It has one dependency that makes some things hard to test while the dependency is compiled in.
 - It is legacy code. Here's what I count as legacy code:
     - [X] There are no existing tests
     - [X] Nobody who remembers how it works is available to help (The last commit was 7 years ago from someone I don't know).
 - Its a video game, and that's fun.

### Base App

{{% notice note %}}
Follow along locally with:  `git checkout 6befd20`  
View online with:  https://github.com/30Wedge/snake-adding-unit-tests-to-legacy-code/tree/6befd200ed7327e6c6e32fef47a1734b0baa2d7d
{{% /notice %}}

I've taken the core snake code and modified it with a folder structure to mimic what an enterprise software repo might look like.

Here's the original state of the project as we're getting to it -- all of the code and logic is in snake.c - the main function makes direct calls to the graphics library (conio.h) & syscalls. 

Here's what the file tree looks like:
```
.
└── app
    └── src
        ├── conio.h
        ├── Makefile
        ├── snake.c
        └── snake.h
```

You can run these commands to check out what the app does

```bash
cd app/src
make
./snake
```

As it is now, the code is not structured well for unit testing:
  - very few functions are pure without side effects.
  - there are very few seams that we can exploit for unit test.

But we can still do it! With minimal refactoring changes, we can get this code into a unit test harness and work from there.

### Why can't the code be tested now? What's the problem with putting it into a test harness now?

I can summarize my complaints about writing tests for this code into these two bullet points:

 1. There is very little encapsulation. In other words, functions don't follow the "Single-Responsibility Principle" and rely heavily on side effects.
 2. There are very few seams that we can use to sense side effects for unit test.

Both the lack of encapsulation and lack of seams are hurdles on their own and when combined form an even greater hurdle blocking us from writing meaningful tests. I'll take the code in the state its in now, make a perfunctory attempt to write a test for each listed problem to show how it drags down our unit testing efforts.

#### Introducing the example & how lack of encapsulation feeds into the problem

When a function has _way_ too much responsibility, it consequently takes _way_ too much logic to verify all of its responsibilities are performed.

Lets say we want to write unit tests for the `function setup_level()`. This function is over 100 lines long, and has many different responsibilities. It looks like the original author separated the responsibilities into different sections, which are headed with the following block comments.

 - **"Initialize on (re)start"**
 - "Set up global variables"
 - "Fill grid with blanks"
 - **"Fill grid with objects"**
 - "Create snake array of length snake->len"
 - **"Draw playing board"**

That's certainly more than a single responsibility! For simplicity of example I'm going to focus on the three bolded sections.

Here's an except of the function, with the bolded sections listed in full, and the rest of the sections replaced with ellipsis.

```Cpp
void setup_level (screen_t *screen, snake_t *snake, int level)
{
   int i, row, col;

   srand ((unsigned int)time (NULL));

   /* Initialize on (re)start */
   if (1 == level)
   {
      screen->score = 0;
      screen->obstacles = 4;
      screen->level = 1;
      snake->speed = 14;
      snake->dir = RIGHT;
   }
   else
   {
      screen->score += screen->level * 1000;
      screen->obstacles += 2;
      screen->level++;          /* add to obstacles */

      if ((screen->level % 5 == 0) && (snake->speed > 1))
      {
         snake->speed--;        /* increase snake->speed every 5 levels */
      }
   }

   /* Set up global variables for new level */
   //...

   /* Fill grid with blanks */
   //...

   /* Fill grid with objects */       
   for (i = 0; i < screen->obstacles * 2; i++)
   {
      /* Find free space to place an object on. */
      do
      {
         row = rand () % MAXROW;
         col = rand () % MAXCOL;
      }
      while (screen->grid[row][col] != ' ');

      if (i < screen->obstacles)
      {
         screen->grid[row][col] = CACTUS;
      }
      else
      {
         screen->gold++;
         screen->grid[row][col] = GOLD;
      }
   }

   /* Create snake array of length snake->len */
   //...

   /* Draw playing board */
   clrscr();
   draw_line (1, 1);

   for (row = 0; row < MAXROW; row++)
   {
      gotoxy (1, row + 2);

      textcolor (LIGHTBLUE);
      textbackground (LIGHTBLUE);
      printf ("|");
      textattr (RESETATTR);

      textcolor (WHITE);
      for (col = 0; col < MAXCOL; col++)
      {
         printf ("%c", screen->grid[row][col]);
      }

      textcolor (LIGHTBLUE);
      textbackground (LIGHTBLUE);
      printf ("|");
      textattr (RESETATTR);
   }

   draw_line (1, MAXROW + 2);

   show_score (screen);

   textcolor (LIGHTRED);
   gotoxy (30, 1);
   printf ("[ Micro Snake v%s ]", VERSION);
}
```

I'm going to walk through the steps to make a unit test for this function. First, its helpful to describe what the function does in English, to divine what test cases should be made. I'll start by just trying to test the "Draw playing board" section, and I'll start by describing what it is doing:

When `setup_level()` is called, do the following regardless of the parameters.
  1. Clear the screen  
    a. Draw a lightblue box around the screen  
    b. Draw the content of the screen's grid objects to the screen in white characters  
    c. Draw the score  
    d. Draw a red banner with "Micro Snake"+ version information at coordinates 30,1  

This is a great start. The description is a little verbose; it is vague at times; I'm completely ignoring the rest of the sections for now. However, the descriptions outlines a seemingly simple single test case for this section since it seems like we should expect a single behavior regardless of the function parameters.

However, we quickly run into problems if we try to implement this test. How should we verify how the "Draw playing board" section draws the content of the screen (see 1.c above)?

We can see that `setup_level()`  draws the screen grid by using calls to `gotoxy()`  to align subsequent calls to `printf()` on line 183.
Lets ignore the `gotoxy()`  function for now, and focus on how to verify the  `printf()` calls.
Maybe we could compile in a mocked version of `printf()` which records every character printed into a buffer, and make sure the buffer matches some expected value to verify that its drawing the screen correctly? Its not ideal, but possible. But how would a test know what that buffer's expected value should be? We know it should be based on the contents of the screen→grid array, and that array is initialized in this function. Maybe we could make an independent screen_t  object in our test case, and initialize its grid  array in the same way as `setup_level()`, then use that as the expected value to compare our buffer to.

Looking back to the  "Fill grid with objects" section, we can see that `screen→grid` is initialized based on the return of a `rand()` function.
This makes it hard for us to initialize an identical grid in our test case. Maybe we could copy the logic, and take extra steps to seed the random generator with `srand()` using the same value that `setup_level()` does on line 132.

However- that's still not enough to initialize an identical grid, because the "Fill grid with objects" logic depends on the value of `screen→obstacles`.
If we scroll back up through the code, we can see that `screen→obstacles` is initialized in the "Initialize on (re)start" section of the function and that its initial value depends on the level parameter to the function.
We'd have to copy that logic to our test case as well if we're going to initialize an identical grid to verify the function's calls to `printf()`.


I'm going to stop this thought experiment here. See how the complexity of setting up and verifying this single test case has expanded way beyond the initial description of this section? Its a lot of unnecessary work to copy the grid initialization logic into our test.

Ff the code was written with the single responsibility principle in mind, so that the function drawing the grid was not also in charge of initializing the grid, we would be having a much easier time. Here's an example implementation:

```Cpp
void setup_level (screen_t *screen, snake_t *snake, int level)
{
   //...

   /* Draw playing board */
   draw_initial_board(&(screen->grid), screen->score);
}

void draw_initial_board( const char (*grid)[MAXROW][MAXCOL], int score){
   clrscr(); 
   draw_line (1, 1);

   for (row = 0; row < MAXROW; row++)
   {
      gotoxy (1, row + 2);

      textcolor (LIGHTBLUE);
      textbackground (LIGHTBLUE);
      printf ("|");
      textattr (RESETATTR);

      textcolor (WHITE);
      for (col = 0; col < MAXCOL; col++)
      {
         printf ("%c", (*grid)[row][col]);
      }

      textcolor (LIGHTBLUE);
      textbackground (LIGHTBLUE);
      printf ("|");
      textattr (RESETATTR);
   }

   draw_line (1, MAXROW + 2); 
   show_score (score);

   textcolor (LIGHTRED);
   gotoxy (30, 1);
   printf ("[ Micro Snake v%s ]", VERSION); 
}
```

Now we could write a test to verify all of the logic in the "Draw playing board" section by calling `draw_initial_board()` directly with a `grid` that the test case is free to initialize however is easiest. This method skips the unnecessary and brittle work of making sure that the grid is initialized exactly how `setup_level()` does it.
Better still - the scope of the parameters to `draw_initial_board()` has been narrowed down to only pass what the function needs to do its job.

There are still some unfortunate problems with this example - like how to verify the calls the calls to `printf()`? and what to do to verify that the `show_score()`  function was called correctly? (I hand-wavingly simplified its call signature for this example). The next two examples show more pressing problems though:

The main takeaway here is that **shorter simpler functions are less work to test.**

#### How having few seams makes testing harder

Lets analyze the last code block in the previous example with "alternate screen drawing logic". If we refactored the code like this, the test for setup_level()  would look differently.  Originally we broke the function up into sections, and talked about how to test the sections separately. Now that the "Draw playing board" section is entirely implemented in `draw_initial_board()`, we can test `draw_initial_board()` completely independently from `setup_level()` now.
We should only need to test that `setup_level()` calls `draw_initial_board()` with the right parameters to verify this section. We don't need to duplicate test logic to test `draw_initial_board()`'s behavior in the `setup_level()` test.

The best way to test that `draw_initial_board()` gets called is with a mock function.
With FFF, we could define a mock version of `draw_initial_board()` that does nothing but record how many times it gets called and with what parameters, and we would want to compile this test so that was the test version of `setup_level()`  uses the mock function instead of the real `show_score()`.

However, we can't compile the test in this way without modifying the snake.c code ,because there is no seam between `show_score()`, and `draw_initial_board()`.  
`show_score()` is called directly from `draw_initial_board()` (not *indirectly*,  through a function pointer or a macro layer of indirection) and they are both defined in the same file as strong symbols.
There is no way for the C compiler / linker to substitute a mock version of `show_score()` when compiling the test.

We can fix this, by either declaring `show_score()` as a weak symbol and doing some tricky compiler/linker work or moving `show_score()` and related lower level functions to a separate file. Either approach will create a link seam between the two functions that allows a unit test to compile with a mock version of `show_score()`.

I strongly prefer the strategy of moving related functions to a separate file because its less confusing than sorting out weak symbols, and helps structure your code as an added bonus.

If we weren't able to mock `show_score()`, but we still wanted the test to verify that it was called, we'd have to verify that its side effects executed in the test case.
This is inconvenient, but feasible in this example since `show_score()` is a simple function that only calls library API functions without any conditional logic. However, if `show_score()` conditionally called into other functions (that called into other other functions), the complexity would quickly spin out of control.

The takeaway here is: **not having seams in higher level functions make their tests difficult to write and brittle.**

#### Illustrating the lack of seam with conio.h

Sticking with this example, lets talk about what a unit test for `show_score()`  could look like.
Here's the source of the function under test:

```Cpp
void show_score (screen_t *screen)
{
   textcolor (LIGHTCYAN);
   gotoxy (3, MAXROW + 2);
   printf ("Level: %05d", screen->level);

   textcolor (YELLOW);
   gotoxy (21, MAXROW + 2);
   printf ("Gold Left: %05d", screen->gold);

   textcolor (LIGHTGREEN);
   gotoxy (43, MAXROW + 2);
   printf ("Score: %05d", screen->score);

   textcolor (LIGHTMAGENTA);
   gotoxy (61, MAXROW + 2);
   printf ("High Score: %05d", screen->high_score);
}
```

There isn't much logic to test! However, we could try making a super simple test case that only verified that calls to the `gotoxy()` and `textcolor()` conio.h API functions are made with the right parameters in the right order.
My approach is to create a `mock/conio.h` file that gets included by snake.c only in unit test builds. That file would have mock functions of all the conio.h API functions created by FFF, including `gotoxy()` and `textcolor()` inside of a mock conio.h header.  
However, there is no way to link in our mocked conio.h file as the code is currently structured because of the file structure, the conio.h include statement and the C Preprocessor rules.

For review, here's the file structure:
```
src/
├── conio.h
├── snake.c
└── snake.h
```

And here's the include statement in snake.c:

```Cpp
#include "conio.h"
#include "snake.h"
```

And here's a quick refresher on the gcc search path rules: https://gcc.gnu.org/onlinedocs/cpp/Search-Path.html with the relevant section here:
> By default, the preprocessor looks for header files included by the quote form of the directive #include "file" first relative to the directory of the current file, and then in a preconfigured list of standard system directories.

Because conio.h is in the same directory as snake.c, and its included in quote form, the preprocessor is always going to check the app/src/  directory for conio.h first, and use the real library file, not our mocked file.

Fortunately, this problem doesn't take much work to fix with linker options and restructuring the file system (see the next section).

Having a seam to mock conio.h is more important than not having a seam with any of our internal functions, because we never want our tests for application to depend on the behavior of a third party library that might change and is out of our control.

----

### The road to a compiling test harness

Before starting with hacking on the code, I have a picture in mind of where I want the test harness to end up.

To have the tests sense how the graphics are drawn, we'll have to isolate the conio.h library API calls and replace them with mock functions.

Here's a sketch of what the test folder will look like - where `mock/conio.h` is a file with the conio.h API, but with mock definitions created with FFF.
```
    ├── src
    │   ...
    └── test
        ├── mock
        │   └── conio.h
        └── snake_test.cpp
```

#### Introducing a link seam between snake.c and conio.h

{{% notice note %}}
Follow along locally with: `git checkout 07bd51b`  
View online with:  https://github.com/30Wedge/snake-adding-unit-tests-to-legacy-code/tree/07bd51bef65d8644c39d58c3ce7225ae1ed77110
{{% /notice %}}

To have the ability to compile unit tests with a mocked version of the conio.h library, we're going to need a seam to be able to compile our unit test with the mocked library version without changing the actual application code.

As it is now, we're not going to be able to mock out conio.h because of the search path rules. snake.c includes it with double quotes to a relative path reference: `#include "conio.h"`. Take a gander here at the gcc search path rules: https://gcc.gnu.org/onlinedocs/cpp/Search-Path.html. For quoted include paths, the current directory is searched first before any other location, so we wouldn't be able to override which version of conio.h is included in the build system.

However, if the included file (conio.h) isn't directly in the source file's directory, then  the preprocessor searches in additional directories that can be configured by the build system.

First we're going to move conio.h to a different directory so that the preprocessor can't find the file immediately.

Now our source tree looks like this --
```
    └── src
        ├── conio
        │   └── conio.h
        ├── Makefile
        ├── snake
        ├── snake.c
        └── snake.h
```

Then we're going to change the build system to add `app/src/conio` to the search path when building the `snake` app.

This organization gives us a new power -- by not including the full relative path to conio.h inside of the include directive in snake.c, we let the build system choose which version of conio.h to include, opening up the possibility to include a mocked version of conio.h for a unit test to use.

#### Set up Test infrastructure

The next step is to set up the tools we'll be using for test.
 - fff
 - gtest
 
Both of these projects are open source, permissively licensed, and have enough documentaiton to get them integrated in our project quickly.

##### FFF 

{{% notice note %}}
Follow along locally with:  `git checkout cc6aaed`  
View online with:  https://github.com/30Wedge/snake-adding-unit-tests-to-legacy-code/tree/cc6aaed318e2b6df9aaabe90649b51ad498deffa
{{% /notice %}}

This commit only downloads the fff.h library from [its main github page](https://github.com/meekrosoft/fff). I chose to put it in the path: `test/fff/fff.h` to keep the include directory separate from the directory containing the test cases and separate from any mock library definitions.
```
    └── test
        └── fff
            └── fff.h
```

##### Gtest

{{% notice note %}}
Follow along locally with:  `git checkout 28b6817`  
View online with:  https://github.com/30Wedge/snake-adding-unit-tests-to-legacy-code/tree/28b681772ae3f21e09c70d49afb5eaf25619693d
{{% /notice %}}

This commit is integrating GTest into our project. The [linked official tutorial](https://google.github.io/googletest/quickstart-cmake.html) explains "what files to add" and "how to add them" much better than I could, so I'd recommend checking out how that works.

But to answer the question: "Why are we doing these things?" I have 3 related goals that this commit is solving.
 1. We want to create a test binary that can link against the GTest library.
    - This is the whole point of the aticle :) We have some legacy code; we want to get it under test; GTest is the unit test framework of choice; Therefore to make a GTest unit test, the test suite needs to be linked against the GTest library.
 2. We want to install the Gtest library so that our tests can use it.
    - This supports goal number 1. To link the test against the GTest library, we need to have it installed and integrated into our build system somehow.
    - The tutorial uses some magic inside of its CMakeLists.txt file to download the GTest library and put it somewhere that the build system can use to link tests to.
 3. We want a single build system that can both build the tests and the main application.
    - Why have a single build system?
        - Simplicity. I want to have as few build scripts as possible in this example repo.
        - If this were a commercial project, keeping the build system simple (i.e. well-designed) is going to reduce the ammount of maintenance it needs so you can maximize the time spent on engineering your project.
    - The linked tutorial guides you to creates a CMake Project to build the tests, and I was too lazy to figure out how to integrate it into the existing Makefile. So I went ahead and ported the Makefile build script to CMake.
 
##### Verify GTest is working with fff

Before moving on, I'm writing a tiny GTest unit test that includes fff to make sure that the test can link against the GTest library and find the `fff.h` file in its include path. 
The only purpose of this test is to make sure we installed Gtest and FFF succssfully, so its going to be short, like this:
```Cpp
#include <gtest/gtest.h>
#include <fff.h>

TEST(HelloTest, smoke_test_please_compile) {
  // Please compile
  EXPECT_EQ(0,0);
}
```

If we can compile this file, then GTest and FFF are integrated correctly, if not, then something is wrong.

We can do this with these commands:
```bash
cd app
cmake . && make && ./snake_test
```

You'll see some output with a percentage bar showing build progress and finally the familiar Gtest output showing that this test suite has passed.

```
[==========] 1 test from 1 test suite ran. (0 ms total)
[  PASSED  ] 1 test.
```

Also, you might notice that the on the first time running this build, CMake has downloaded the googletest source code to the `app/_deps` directory and has installed it to the `app/lib` directory. This is where the build system is going to get the GTest libraries to link against to build our unit tests.

#### Migrate snake app to CMake

{{% notice note %}}
Follow along locally with: `git checkout 94add88`  
View online with:  https://github.com/30Wedge/snake-adding-unit-tests-to-legacy-code/tree/94add880d8bd121491c2360adcfbfe941565cba0
{{% /notice %}}

This commit moves the snake app's build system to CMake.  
With one source file, and only a few preprocessor defines, it was a pretty straightforwards change.

You should have both a `snake` and `snake_test` binary in your `/app` directory that will launch the app and unit tests respectively.
```bash
cd app
cmake . && make
```

#### Mock the `conio.h` Dependency with fff

{{% notice note %}}
Follow along locally with: `git checkout c0702d9`
View online with: https://github.com/30Wedge/snake-adding-unit-tests-to-legacy-code/tree/c0702d91499d3e2d11b76243f5b0d58ec3d70e68
{{% /notice %}}
This is the last step before we're ready to link the `snake.o` object to the unit test.

To reiterate We're going to use fff to mock the conio dependency for three reasons:
 1. We don't want to show the game's graphics on the screen while we're running the unit test.
     - This would get in the way of the test results and be a general nuisance. In other situations API functions might do things that would be impossible from the computer running the unit test - like open secure network connections to production servers or interract with embedded hardware.
 2. We still want to be able to compile the application code into the test harness.
     - In goal 1, we don't want any conio.h api functions to run, so what if we just tried to compile without the include path to *any* conio.h? The program wouldn't compile because of undefined symbols to the conio.h API functions. We could write stub functions for each API call in the test harness if this was all we really needed.
 3. We want the tests to be able to *sense* how the app interracts with the graphics library.
     - We're testing to make sure that the function behaves the correct way. We want to make sure its calling these coino.h API functions in the correct way too. This is the biggest benefit of using a mocking library.

In the sample repo, I've made a mock version of the library in `app/test/mock/conio.h`. This file has all of the constants copied from `conio/conio.h`, and declares all of the function-like macro API calls of conio.h so that it can be included in snake.c and there will be no undefined symbols.  The difference is how the function-like macros are defined. `mock/conio.h`, uses FFF's the `FAKE_VALUE_FUNC` and `FAKE_VOID_FUNC` families of macros to declare functions to satisfy conio.h's API calls, but also to define the API functions as fake functions that will track usage.

Take a look at the implementation of the mock below.
You can see that each function-like macro that's defined in `conio/conio.h` (the real libray) is defined by a `FAKE_VALUE_FUNC` macro. These macros are defined by fff and are used to automatically make mock functions from function signatures.

##### test/mock/conio.h

```C
//...
/*
 * Define mocks for all of the function-like macros in conio.h.
 */
FAKE_VALUE_FUNC(int, clrscr);
FAKE_VALUE_FUNC(int, clreol);
FAKE_VALUE_FUNC(int, delline);
FAKE_VALUE_FUNC2(int, gotoxy, int, int);
FAKE_VOID_FUNC(textattr, int);
FAKE_VOID_FUNC(textcolor, int);
FAKE_VOID_FUNC(textbackground, int);
//...
```

----

#### What are those fff macros doing? A Deeper dive:

Lets focus on a single API call for a concrete example: `gotoxy(x, y)`. This is a function-like macro in conio.h that sets the color of text to write to the screen. If it were defined as a function, it would have a signature like this `int gotoxy(int x, int y);`.

The example above uses the line `FAKE_VALUE_FUNC2(int, gotoxy, int, int);` to make a mock function satisfying that signature. Under the hood in fff.h, that macro expands to create two primary things.

 1. fff defines a function with the signature `int gotoxy(int x, int y);` that acts like the `conio/conio.h` API call for the code under test to use.
 2. fff creates an instance of a struct called `gotoxy_fake` (the generic name is `function_name` + `_fake`) for the test to use. This struct can be modified to change how the `gotoxy()` function behaves  during the test. Also, the struct has all sorts of information how the fake was used by the test.

##### A concrete example of how to use the mock function created by FFF.

Lets define this example function that the app-under-test might have called `foo()` that makes some conio.h API calls to `gotoxy()`:
```Cpp
void foo() {
    gotoxy(20,30);
    //...
    int return_val = gotoxy(50,70);
    //.. do something with return_val
}
```

Then lets define a sample test that calls the sample function `foo()` once. Before calling `foo()`, the test configures how the mocked `gotoxy()` function will behave. After calling `foo()` the test verifies that `foo()` interacted with `gotoxy()` as expected. We'll leave the actual configuration and verification in placeholders with `//...`.
```
TEST(ExampleTest, test_foo)
{
    // Configure how the mocked version of gotoxy() will behave
    //   by modifying gotoxy_fake
    // ...
    
    // Call the app under test's function:
    foo()
    
    // Verify how the API was used by reading gotoxy_fake's sense fields.
    //...
}
```

In this example, all of the test configuration and verification happens by interacting with the `gotoxy_fake` struct that's generated by the FFF. Here's a list of some of the most useful members of this struct & descriptions of what they do.
     
Most useful fake struct members:
 - `arg0_val`/ `arg1_val`
     - These fields capture the values of the arguments that the fake function was caleld with last.
     - After `foo()` is run, `gotoxy_fake.arg0_val` will be 50 and `gotoxy_fake.arg1_val` will be 70.
 - `arg0_history` / `arg1_history`
     - These fields capture the history of arg values passed to the fake function. By default, they will store up to 50 values before overwriting old values.
     - After `foo()` is run, `gotoxy_fake.arg0_history` will be equal to `{20, 50, 0, 0 ,0, ...}` and `gotoxy_fake.arg1_history` will be equal to `{30, 70, 0, 0 ,0, ...}`.
- `call_count`
     - This is a count of how many times the fake function is called.
     - After `foo()` is run, `gotoxy_fake.call_count` will be 2.
 - `return_val`
     - When the fake function is run, this value is what the fake function returns.
     - If `gotoxy_fake.return_val` is set to -1 before `foo()` is run, then the variable `return_val` will be set to -1 in the function.

Here's how some of that configuration and verification might look
```
TEST(ExampleTest, test_foo)
{
    // Configure how the mocked version of gotoxy() will behave
    //   by modifying gotoxy_fake
    // have gotoxy() return 42
    gotoxy_fake.return_val = 42;
    
    // Call the app under test's function:
    foo()
    
    // Verify how the API was used by reading gotoxy_fake's sense fields.
    // make sure gotoxy() is called twice
    EXPECT_EQ(gotoxy_fake.call_count, 2);
    // make sure the arguments passed to gotoxy() are (20,30) and (50,70) in that order.
    EXPECT_EQ(gotoxy_fake.arg0_history[0], 20);
    EXPECT_EQ(gotoxy_fake.arg1_history[0], 30);
    EXPECT_EQ(gotoxy_fake.arg0_history[1], 50);
    EXPECT_EQ(gotoxy_fake.arg1_history[1], 70);
}
```

### A simple demo with FFF mocking library calls

{{% notice note %}}
Follow along locally with: `git checkout 50b0765`  
View online with: https://github.com/30Wedge/snake-adding-unit-tests-to-legacy-code/tree/50b07658445910a18b2a0f45bf382826f4bfb6cf
{{% /notice %}}

This commit creates a test that applies the mocking strategy above to a real function from snake.c app. Take a look at `show_score()` in snake.c. Its a very simple function that makes a few conio.h API calls then returns unconditionally.

We'll set up the test case in the same way as before. 
 1. Set up input and mock behavior for the application function.
     - In this case, `show_score()` doesn't do anything with the return value from any API functions, so we don't need to configure any mock behavior.
 2. Call the application function.
 3. Verify the function behaved as expected through its mock calls.
 
Here's how the first test looks off the bat:
```
TEST(MockVerify, show_score_calls_some_API_calls) {
TEST(MockVerify, show_score_calls_some_API_calls) {
  /*
   * Set up input to the show_scores() function
   */
  screen_t my_screen;
  my_screen.level = 1;
  my_screen.gold = 2;
  my_screen.score = 3;
  my_screen.high_score =4;
  // Call show_scores() which calls some conio.h API functions in its body
  show_score(&my_screen);

  // We're not going to specify *what* the API functions called were,
  //   but these EXPECT statements show that at textcolor() and gotoxy() were called
  //   some non-zero number of times
  EXPECT_NE(textcolor_fake.call_count, 0);
  EXPECT_NE(gotoxy_fake.call_count, 0);

  // cleanup fakes after test
  RESET_FAKE(textcolor);
  RESET_FAKE(gotoxy);
```

Is the `show_score()` function suited for unit testing? And is this test *good*? I'd answer "No" and  "Absolutely not" to those questions respectively.

However its a start. This test does show how the GTest `EXPECT_` macros and FFF fake structures can be combined to verify how a real application function uses a library API call.

### Unit testing a function with no library API calls

{{% notice note %}}
Follow along locally with: `git checkout 35959c9`  
View online with: https://github.com/30Wedge/snake-adding-unit-tests-to-legacy-code/tree/35959c917ca1888a5111ece07c6bf6b52fb98196
{{% /notice %}}

I'm adding this example to be thorough and show a little more of how to use GTest. Not all unit test cases need to use mocks to configure and verify behavior.

This commit introduces a test suite to test `eat_gold ()`.

Take a look at the implementation of this function in snake.c below:
```C
int eat_gold (snake_t *snake, screen_t *screen)
{
   snake_segment_t *head = &snake->body[snake->len - 1];

   /* We're called after collide_object() so we know it's
    * a piece of gold at this position.  Eat it up! */
   screen->grid[head->row - 1][head->col - 1] = ' ';

   screen->gold--;
   screen->score += snake->len * screen->obstacles;
   snake->len++;

   if (screen->score > screen->high_score)
   {
      screen->high_score = screen->score; /* New high score! */
   }

   return screen->gold;
}
```

Note that there are no other functions called from `eat_gold()`, including API calls, so we don't need to use fff to configure or verify this function.
 
We can sense how this function behaves in two different ways:
 1. based on its `int` return value
 2. based on how it modifies its parameters, `snake` and `screen`.
 
We can configure this function in only one way:
 1. By deliberately crafting its input parameters.

In some ways, this function is more simple to test than `show_score()` was. There are no dependencies to mock.

However, take a look at the definition for the structs of the parameter types `snake_t` and  `screen_t`.
```C
typedef struct
{
   unsigned int    speed;
   direction_t     dir;

   int             len;
   snake_segment_t body[100];
} snake_t;

typedef struct
{
   int level;
   int score;
   int high_score;
   int gold;
   int obstacles;

   char grid[MAXROW][MAXCOL];
} screen_t;
```
Each of these structs has a lot of data. `snake_t` has members that are other structs. If we're only able to configure this function by deliberately crafting the input parameters, then we'll have to do a good amount of work to build these structs correctly.
Since the function communicates output by modifying its parameters, we're also going to have to do work to verify the relevant fields of those structs as output.

Thankfully, this function doesn't have many branching control flow. There's only one `if` statement in the function body. 

As a first step to decide what to test, it can be helpful to describe what the function does in simple English statements.
 - `eat_gold()` updates the `grid` `gold` `score` and `len` field of the `screen` parameter when called.
 - If the `score` field is higher than the `high_score` field, then `eat_gold()` also updates the `high_score` field of `screen`
 - `eat_gold()` always returns the updated `gold` field of the `screen` parameter
 - `eat_gold()` increases the `len` field of the `snake` parameter by one every time it is called.
 
I can then take that description of what the function does, and plan it out into different test cases to cover the basic operation of the function.
 1. Test that the return value is equal to the screen's gold after the test is run.
 2. Test that the screen's parameters are changed as expected during normal conditions.
 3. Test that the screen's parameters are changed as expected when a high score update is expected.
 4. Test that the snake's state is changed as expected.

##### Cutting down on duplication setup with GTest Fixtures

Int the step above, it made sense to build 4 test cases for this function. 

Instead of copying and pasting common initialization code, we can use GTest *fixtures* to share common configuration for each test case that's going to test this function.

A text fixture lets you define data, setup/teardown and other common functions between a suite of tests.

In the words of the GTest docs, here's when to use a test fixture:
> If you find yourself writing two or more tests that operate on similar data, you can use a test fixture. This allows you to reuse the same configuration of objects for several different tests.

I find myself using these liberally in unit tests that involve anything beyond a simple pure function.

For these specific 4 test cases, I'm going to define a test fixture that defines two objects of the function's parameter types (`snake_t m_snake` and `screen_t m_screen`) and then initializes them into a common state so that individual test cases have to modify them as little as possible.

For full details on all the bells and whistles that test fixtures can provide, see the docs: https://github.com/google/googletest/blob/master/docs/primer.md#test-fixtures-using-the-same-data-configuration-for-multiple-tests-same-data-multiple-tests

#### The resulting test cases:

These next sections are a walkthrough of the major components of these tests.

##### Test Fixture EatGoldTest

Below is the definition of the test fixture in full.



```C++
class EatGoldTest : public ::testing::Test
{
protected:
  snake_t m_snake;
  screen_t m_screen;
public:

  const int SNAKE_POS_X = 10;
  const int SNAKE_POS_Y = 10;
  const int SNAKE_LEN = 1;

  const char SCREEN_FILLER = 'X';
  const char SCREEN_BLANK = ' ';
  const int SCREEN_GOLD = 5;
  const int SCREEN_SCORE = 100;
  const int SCREEN_HI_SCORE = 500;
  const int SCREEN_OBSTACLES = 10;

  EatGoldTest() {
    /// set up snake
    m_snake.len = SNAKE_LEN;
    m_snake.body[0] = {SNAKE_POS_X, SNAKE_POS_Y};

    /// set up screen
    // fill screen up with filler to detect no-change in other spaces
    for(int row = 0; row < MAXROW; row++){
      for(int col = 0; col < MAXCOL; col++){
        m_screen.grid[row][col] = SCREEN_FILLER;
      }
    }
    m_screen.gold = SCREEN_GOLD;
    m_screen.score = SCREEN_SCORE;
    m_screen.high_score = SCREEN_HI_SCORE;
    m_screen.obstacles = SCREEN_OBSTACLES;
  }
};
```

This test fixture uses these three things at a high level
 1. Defines shared variables
     - `m_snake` and `m_screen` are variables that can be accessed from unit tests that use this fixture.
 2. Defines constants
     - all of these const declarations like `const int SCREEN_GOLD` share information about the default state of the 
 3. Defines those variables in a shared constructor.
     - A separate instance of this object gets created before every related test case runs. I put the initialization code into the fixture's constructor so that whatever initialization has to run is completed before the test logic runs.

##### Test return code

Now that we've looked at the fixture definition, lets look at the first test case that uses this fixture.

```C++
TEST_F(EatGoldTest, test_eat_gold_return_value_is_correct)
{
  /* Given: default screen & snake*/
  /* When */
  int ret = eat_gold(&m_snake, &m_screen);
  /* Then */
  // decrement gold count
  ASSERT_EQ(ret, m_screen.gold);
}
```
This is all we need. Most of the lines are comments. We don't have to do any of the work to set up `m_screen` or `m_snake` because that's been done in the fixture constructor.

This test verifies that when passed the default `m_screen` object created by the fixture, `eat_gold()` returns the new value of `m_screen.gold`, and nothing else.

Its tempting to try to have one unit test validate as much as possible, but I believe that if each test has a descriptive name and tests one thing, then its easier to figure out what's wrong when something bad happens.

For example, if I change `return screen->gold;` in `eat_gold()` to `return 7;` Here's what the summary output of the test suite would look like:

```GTest
[==========] 7 tests from 3 test suites ran. (2 ms total)
[  PASSED  ] 6 tests.
[  FAILED  ] 1 test, listed below:
[  FAILED  ] EatGoldTest.test_eat_gold_return_value_is_correct
```

Then it is very clear that something is wrong with *only* this function's return value and none of the other behavior.

##### Test Screen modification

Lets look at a more complicated test case, like this test that checks that `eat_gold()` modifies `m_screen` correctly in a scenario that doesn't involve a high score update.

Looking at the `eat_gold()` function, we can break down what we expect the function to change in the screen structure. Here's my expectations of how `eat_gold()` modifies the `m_screen` object broken down into four statements in English.
 - It should replace the grid element at the snake's head position with a `' '` character (and modify no other grid spaces)
 - It should decrease the `screen.gold` value by 1
 - It should increase the `screen.score` value by however much this piece of gold was worth.
 - It should not increase the `screen.high_score` member unless the score is greater than the previous high score.
 
Here's how this test case verifies each of those statements in C++.
 
```C++
TEST_F(EatGoldTest, test_eat_gold_modifies_screen_correctly)
{
  /* Given: default screen & snake*/
  /* When */
  eat_gold(&m_snake, &m_screen);
  /* Then */
  // blank out the head of the snake on the screen
  for(int row = 0; row < MAXROW; row++){
    for(int col = 0; col < MAXCOL; col++){
      if(row == SNAKE_POS_X - 1 && col == SNAKE_POS_Y - 1)
      {
        ASSERT_EQ(m_screen.grid[row][col], SCREEN_BLANK);
      }
      else
      {
        ASSERT_EQ(m_screen.grid[row][col], SCREEN_FILLER);
      }
    }
  }

  // decrement gold count
  ASSERT_EQ(m_screen.gold, SCREEN_GOLD - 1);

  // increase score but not hi score
  int score_increment = SNAKE_LEN * SCREEN_OBSTACLES;
  ASSERT_EQ(m_screen.score, SCREEN_SCORE + score_increment);
  ASSERT_EQ(m_screen.high_score, SCREEN_HI_SCORE);
}
```

See the companion repo for the other two test cases.

#### Are these test cases enough?

I think it depends. These tests give enough coverage so that way an engineer could either add features or refactor the code with confidence that they wouldn't introduce a regression bug.

I definitely don't think these tests cover all failure modes for the function though. If this code wasn't in a `snake` game, but was being certified in an airplane's flight control computer, then it would need a whole lot more "robustness tests" to cover how the function responds to unexpected input.

For example, what would happen when `screen.gold` is 0 when `eat_gold()` is called?
The `screen->gold--;` statement would execute and now `screen.gold` would be -1 and the function would return -1.
From the rest of the code, its clear that this condition is not meant to happen. So how should the function handle it?
   - If robustness testing is an issue on your project, then that question should be answered in the software requirements document.
   - This is worth mentioning for thoroughness, but testing at that level of robustness is out of scope for this page.
   
### Recap

So by now, you've seen the what has to change to get a legacy C app into a unit test harness. We had to break the dependency on the app's third party library, integrate GTest and fff into the build system, write a mock version of the dependency with fff then write some trivial unit tests on functions in the main app.