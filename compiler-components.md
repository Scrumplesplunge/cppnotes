# Compiler Components

It is easy to use many programming languages without having an understanding of
how the code you write is translated into the executable format which runs.
While it is possible to do this with C and C++, I would say that there are more
cases where the abstraction breaks down, and for this reason it is useful to
have an awareness of the translation steps. I will start with handwritten source
code and work my way down to the final executable, explaining the steps which
happen inbetween.

## The C preprocessor

The first major step to happen is evaluation of the _C preprocessor_. The
preprocessor is a simple text transformation which reads your source code and
outputs different source code which is free of _preprocessor directives_ and
_macro invocations_.

### Preprocessor directives

Any line starting with a `#` is a preprocessor directive. These lines instruct
the preprocessor to perform certain actions. There are many available
preprocessor directives, but it is rare to need many other than the following:

  * `#define` - Used to define macros. These are described in more detail below.
  * `#include` - Used to include the contents of another file at the location of
    the directive. Note that this is a literal text substitution with the
    contents of the file, unlike the more restrictive behaviour of `import`
    statements in many other languages. The preprocessor will paste the contents
    of the file where the `#include` directive was found and then continue
    processing from the _start_ of the included contents. In effect, all
    `#include` directives are recursively expanded before the source code is
    compiled. The fully expanded code is known as the _translation unit_.
  * `#ifndef` and `#endif` - Read as "if not defined". The text appearing
    between the `#ifndef` and the `#endif` will be erased if the condition is
    not true. Since preprocessor directives are included recursively, we may
    accidentally include the same header file multiple times. While this does
    not matter with declarations alone, it is an error to define something twice
    and this will frequently fail to compile. The common pattern is to wrap the
    contents of header files in a _header guard_:

        // In foo.h:
        #ifndef PATH_TO_FOO_H_
        #define PATH_TO_FOO_H_

        ...

        #endif  // PATH_TO_FOO_H_

    This will prevent us from including the contents of `foo.h` a second time.
  * `#pragma` - Used to control _implementation-defined behaviour_ of the
    compiler. A particular `#pragma` line may work in one compiler but fail in
    another, so portable code should avoid using them. A particular directive
    which is supported by practically all modern compilers is `#pragma once`,
    which replaces the somewhat cumbersome header-guard from above:

        // In foo.h:
        #pragma once  // Tells the preprocessor to only include this file once.

        ...


Older compilers will only detect preprocessor directives which appear at the
very start of a new line, so for portability reasons, **preprocessor directives
should never be indented**.

### Macros

Macros provide a way to avoid writing complex boilerplate code that for one
reason or another cannot be encapsulated in a function. Macros may be
parameterless, or they may take parameters. However, in both cases it is
important to recognise that macros invocations may behave unpredictably due to
the fact that they do not have to follow normal syntax conventions:

    #define forever while (1)
    forever {
      // Code here will be repeated forever.
    };
    
    #define sneakysneaky } else {
    if (SomeCondition()) {
      ...
      sneakysneaky
      ...  // This is in the else branch: it runs if !SomeCondition()
    }

The body of a macro can span multiple lines if the lines are terminated with
`\`. They may not contain preprocessor _directives_, but they may invoke other
macros (although the same macro will never be expanded twice in any recursive
expansion). Macros may also use the unary `#` operator and the binary `##`
operator to perform special functions with their arguments.

The unary `#` operator converts the raw text of the argument into a string
literal:

    #define ASSERT(condition)  \
      if (!(condition)) {  \
        std::cerr << "Assertion failed: " << #condition << "\n";  \
        std::exit(EXIT_FAILURE);  \
      }

    int x = 1, y = 2;
    // Displays:
    // Assertion failed: x == y
    ASSERT(x == y);

The binary `##` operator concatenates the raw contents of an argument with
either another argument or with any other piece of code. Normally this is not
needed because whitespace is often insignificant and you can just use a space,
but this does allow you to create new identifiers:

    enum class Operation { ADD, SUB, MUL, DIV, };
    int DoADD(int a, int b) { return a + b; }
    int DoSUB(int a, int b) { return a - b; }
    int DoMUL(int a, int b) { return a * b; }
    int DoDIV(int a, int b) { return a / b; }

    int DoOp(Operation operation, int a, int b) {
      switch (operation) {
    #define CASE(op) case Operation::op: return Do##op(a, b); break
        // Expands to: case Operation::ADD: return DoADD(a, b); break;
        CASE(ADD);
        CASE(SUB);
        CASE(MUL);
        CASE(DIV);
      }
    }

### Predefined macros

There are a handful of useful pre-existing macros which you can use to gather
information about context around the site of the macro invocation. These are
some of the ones I consider to be the most useful:

  * `__LINE__` expands to the current line number at the point of use. Note that
    because macros expand outside-in, and because the `\` really just escapes
    the newline in the definition and doesn't leave the newline in the
    expansion, using `__LINE__` inside a macro will expand it at the point of
    use of the outer macro:
    
        1: int x = __LINE__;  // x = 1
        2: #define FOO \
        3:   __LINE__
        4: int y = FOO;  // y = 4

  * `__FILE__` expands to a string-literal containing the name of the current
    file.
  * `__func__` expands to a string containing the name of the enclosing
    function.
  * `__DATE__` expands to a string containing the current date in the format
    "Jan 01 2000".
  * `__TIME__` expands to a string containing the current time in 24-hour format
    "HH:MM:SS".

There are many macros that exist to support _feature detection_, so that you can
conditionally enable special features if the compiler supports them, but I don't
think it is worth listing these here - you can search for them if you are
interested.

## Compilation of translation units

The next stage of compilation takes a translation unit and compiles it in
isolation to produce an _object file_. This file contains compiled code for all
of the functions in the translation unit as well data which the program uses.
This stage of compilation is where most (but not all) optimisations can be
applied to the program. The source code is generally transformed into some
intermediate representation, optimisations are applied to this representation,
and finally the representation is converted into machine code via instruction
selection. Some optimisations cannot be performed here, and some are not even
possible with this method of compilation (look up link-time optimisation for
details).

Consider the following code:

    void DeclaredButNotDefined();

    static int static_int = 0xBEEF;
    int just_int = 0xDEAD;
    const char* const constant_string_literal = "Hello, World!";
    
    int FunctionReturningInt() {
      int temp = static_int + just_int;
      return temp;
    }
    
    const char* FunctionReturningStringLiteral(int x) {
      if (x == 42) {
        return "The answer to life, the universe, and everything!";
      } else {
        return constant_string_literal;
      }
    }

When compiled as C, this produces an object file containing several _sections_.
The most important sections are:

  * `.text` is the section containing the compiled code for our program.
  * `.data` is the section containing values for global variables which should
    be read-write at runtime.
  * `.rodata` section(s) are for data which should be read-only at runtime. For
    example, the contents of string-literals is constant, so it appears in
    `.rodata`.

The file also contains a handful of _symbols_ which are essentially names for
particular locations in particular sections. For example, the code above has the
following symbols:

    *ABS*           test.cc
    .data           .data
    .data           just_int
    .data           static_int
    .data.rel.ro    constant_string_literal
    .rodata.str1.1  .rodata.str1.1
    .rodata.str1.1  .L.str
    .rodata.str1.1  .L.str.1
    .text           .text
    .text           FunctionReturningInt
    .text           FunctionReturningStringLiteral

The first part is the section name, second part is the symbol name. See how
there are symbols for each of the global variables, and for each of the
functions, but no symbols for the function arguments or local variables. There
are also symbols for each of the string literals in `.rodata`. Note also that
there is _no symbol for `DeclaredButNotDefined`_. The object file only contains
symbols for things it has definitions for. Since this function is only declared,
there is no symbol. Say we were to call this function from
`FunctionReturningInt`. This would produce a _reference_ to the symbol for the
function, but would still not define the symbol. It is the responsibility of the
linker to tie together the references and the definitions.

When the same code is compiled as C++, the symbols look slightly different:

    *ABS*           test.cc
    .data           .data
    .data           _ZL10static_int
    .data           just_int
    .rodata.str1.1  .L.str.1
    .rodata.str1.1  .L.str
    .text           .text
    .text           _Z20FunctionReturningIntv
    .text           _Z30FunctionReturningStringLiterali

The names of some of the symbols have been _mangled_. In C, there is no name
mangling and symbols for functions are simply the same as the name of the
function. However, C++ supports function overloading, meaning that you can
define several functions with the same name as long as they have different
parameter types. If this were to be compiled using the C approach, we would have
multiple symbols with the same name.

Instead, C++ compilers use name mangling to encode parts of the type information
for functions _and variables_ into the names. You can look up encoding schemes
online, but just be aware that name mangling exists and know that many tools can
display these as sensible-looking C++ for you if you ask them to.

## Linking

Once translation units have been processed, the linker is responsible for taking
the individual units, finding all references to symbols between the units,
ensuring that every symbol _which is used_ has a definition, and finally laying
out all of the contents into an executable form with all the symbols resolved
into native addresses.

The point about symbols being used is important. Due to the nature of having
headers with declarations and source files with definitions, it is very easy to
accidentally declare a symbol without defining it. However, as I mentioned
before, no symbols or symbol references are made when we _declare_ a function:
only when we _define_ or _call_ one. This means that if we declare a function
and then do not call it, the linker has no idea that anything has gone wrong.
The moment we change our code to call the function, however, we will start
seeing linker errors.

## Example compilation with `clang`

Both the `clang` and `gcc` frontend commands can perform preprocessing,
compilation, and linking internally. Suppose that we have the following project:

    // sum.h
    #pragma once
    #include <vector>
    int sum(const std::vector<int>& numbers);

    // sum.cc
    #include "sum.h"
    int sum(const std::vector<int>& members) {
      int total = 0;
      for (int x : members) total += x;
      return total;
    }

    // main.cc
    #include "sum.h"

    #include <iostream>

    int main() {
      std::vector<int> numbers = {1, 2, 3, 4, 5};
      return sum(numbers);  // Should return 15
    }

We could compile this project with:

    clang++ sum.cc main.cc -o myproject

This will perform preprocessing, compilation, and linking. The compiler will
produce a single file called `myproject` which contains our program.

> Note that we do not have to compile the header. The header only declares
> things which will be included in other translation units anyway, so it doesn't
> make sense to compile the header in isolation.

Often, it is sufficient to compile programs like this. However, there are cases
where we might want to break the compilation down into its component steps. One
of the main reasons to do so is _incremental compilation_. If we had a project
with 10000 source files and we compiled it like this, it might take a while. If
we then modified a single file and compiled it like this again, it would take
approximately the same amount of time; all the of the files which didn't change
would have to be compiled again from scratch.

To save some processing time, we could perform the (preprocessing and) compiling separately from linking:

    # Phase 1: Compile all of the translation units.
    clang++ -c sum.cc -o sum.o
    clang++ -c main.cc -o main.o
    # do the same for all files. You could use a makefile to do this.
    ...

    # Phase 2: Link translation units together.
    clang++ sum.o main.o ... -o myproject

    # Modify a single file.
    vim sum.cc

    # Perform an incremental compilation:
    clang++ -c sum.cc -o sum.o
    # all other translation units are already up to date.  
    clang++ sum.o main.o ... -o myproject

This saves us a lot of time when performing incremental builds at the cost of
being slightly slower for complete rebuilds. However, notice how all of the
compilation steps for the translation units are independent. This means that we
can perform all of these in parallel, potentially even using multiple computers
to do the compiling. Not everyone has multiple computers to compile with, but
almost everyone has multiple cores available in their computers these days, so
it is generally possible to get a decent speed-up from this.
