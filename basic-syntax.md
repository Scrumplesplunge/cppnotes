# Basic Syntax

C++ syntax is built around the syntax of C. When C++ was first created, it was
roughly a superset of C. That is, all C programs were valid C++ programs.
However, as the language has evolved, this has ceased to be true. There are
features in C which are unavailable in C++ (such as variable-length stack arrays
or designated initializers), and obviously there are features of C++ which are
not available in C.

## Declarations and definitions

A distinction which isn't always obvious in other languages is the distinction
between _declarations_ and _definitions_. In C and C++, it is possible to
_declare_ something and then _define_ it later. The declaration tells you about
the type of something, and the definition tells you about the contents. For
example, a function called `foo` which takes an `int` and a `char` and returns
a `float` could be declared with:

    float foo(int a, char b);

Once the function is declared, the compiler will allow us to call it:

    float bar() { return 2.0f * foo(42, 'a'); }

And as long as we provide the definition somewhere in our program, the linker
will be able to put the program together later. The distinction between
declarations and definitions is important, as it is mostly declarations which
form _header files_ that can be included in a C program, and definitions which
form the _source files_.

## C/C++ Syntax

Many aspects of C syntax will be familiar to people who have experience with
programming in any of a plethora of languages. I don't plan on describing every
detail when most of it should be easy enough to guess. I will only point out
a few specific details.

### Variable and function declarations

If you have previously used a language like Java, you might read

    int x;

as

> `x` is a variable of type `int`.

In this example, you could apply the same reasoning in C and you would be
correct. However this is not always the case. For instance, a pointer type in
C is denoted using an asterisk. You will frequently see snippets like
`(int*)` in C code, and you may read `int*` as "pointer to `int`". Yet, the
declaration

    int* x, y;

does not declare two pointers to ints. Instead, it declares `x` as a pointer to
`int`, and `y` as an `int`. The position of the spaces is entirely insignificant
and this is probably more easy to understand when it is written as:

    int *x, y;

Really, we should read the above line as

> The expressions `*x` and `y` both have type `int`.

This is perhaps a little cumbersome at first, but this pattern continues to
apply when you consider a more complicated declaration:

    int *foo(int x, int y);

We might read the above as

> If the expressions `x` and `y` have type `int`, then the expression
> `*foo(x, y)` has type `int`.

In fact, it is even okay to declare functions like this:

    int x, y, foo(int a, int b), z;

If we were to try and make sense of this with our Java hat on, it would look
like rubbish. With our C hat on, we can see that `foo(int a, int b)` is simply
another expression of type `int`. Of course, this particular example is horrific
and ugly and should be reserved for obfuscated code contests.

### Using compound types

There are a handful of ways to construct new type in a C program:

  * An `enum` type is an integral type with a collection of named values.
  * A `union` type has at most one of its fields populated.
  * A `struct` type has a number of fields consisting of other types.

In C, it is necessary to spell out which kind of type you are using when you use
it:

    enum Foo {
      FOO_BAR,
      FOO_BAZ,
    };

    Foo incorrect = FOO_BAR;  // This will compile in C++ but not in C.
    enum Foo correct = FOO_BAR;  // This will compile in both C and C++.

Another way of introducing new type names in C is to create a `typedef` of an
existing type to give that type another name:

    typedef enum Foo FooEnum;
    enum Foo a = FOO_BAR;
    FooEnum b = FOO_BAZ;

You should read `typedef enum Foo FooEnum` in a similar manner to how
I described variable types above:

    enum Foo foo_value;  // The expression "foo_value" has type "enum Foo".
    typedef enum Foo FooEnum;  // The expression "FooEnum" *is* type "enum Foo".

Note that a `typedef` does not require you to specify what kind of type it is.
It is perfectly legal to declare a typedef with the same name as the type:

    struct Foo {
      ...  // Contents of struct definition
    };
    typedef struct Foo Foo;

    enum Foo correct = FOO_BAR;
    Foo also_correct = FOO_BAZ;  // This will compile in both C and C++.

In fact, it is possible to express this all at once with:

    typedef struct Foo {
      ...   // Contents of struct definition
    } Foo;

However, this is unnecessary in C++.

### The `const` keyword

Variables in C/C++ can be declared as `const`. What this means depends upon the
placement of `const` with respect to other parts of the type expression, but
generally `const` is promising that attempts to change the value via this name
will fail. For example:

    int x = 0;  // "x" is an "int"
    int const y = 0;  // "y" is an "int const".
    const int z = 0;  // "const int" is a "nicer" way of writing "int const"
    int* const a = &x;  // "a" is a const pointer to int, and points at x.
    int const* b = &x;  // "b" is a pointer to const int, and points at x.
    int* c = &y;  // Compile error: y is a const int.

    x = 1;  // Fine
    y = 1;  // Compile error: y is const.
    z = 1;  // Compile error: z is const.
    *a = 1;  // Fine: the value of a is const, but the value it points at isn't.
    a = &y;  // Compile error: a is const.
    *b = 1;  // Compile error: *b is a const int.

The cases above speak for themselves, but there is one point of particular
interest: the variable `b` is a pointer to `const int` which points at `x`, but
`x` is not a `const int`. What this means is that `x` cannot be modified via the
name `b`. The key point is that `const` is a property of the _name_, not of the
_value_. It is fine to convert a non-const value into a const one in this way
since the new name will not be able to modify the value. The converse is not
true, which is why the definition for `c` causes a compile error.

## Hello World

I will now briefly describe Hello World in C and C++, transitioning from the
former to the latter and covering the main points about each implementation. 

### The C way

    #include <stdio.h>

    int main() {
      printf("Hello, World!\n");
    }

The C code is pretty straightforward. We declare a single function called
`main`. `main` is the entry point for C and C++ programs; it is where execution
of the program begins\*. In C, the type signature for `main` is somewhat
flexible, but it generally returns `int` and accepts either zero parameters or
two. The two parameter case is a subject for another article.

`main` prints "Hello, World!" followed by a newline using `printf`, which is
provided by the header `stdio.h` from the C standard library. Sharp-eyed readers
might be thinking, "shouldn't there be a `return 0;`?". In the _particular_ case
of `main`, `return 0;` is the default behaviour, and it is often omitted.

### The C way, in C++

    #include <cstdio>

    int main() {
      std::printf("Hello, World!\n");
    }

We can do the same thing in C++. Each of the C standard library headers has an
equivalent header in C++. Header `foo.h` is available as `cfoo`. However, there
is an additional difference: the C++ versions of these headers puts all
definitions inside the _`std` namespace_. All of the standard library features
for both C and C++ are exposed through this namespace. For the purposes of our
example, this simply means that we have to put `std::` before `printf` to tell
the compiler where to look for `printf`.

### The C++ way

C++ has a different mechanism for printing text to `stdout`: _streams_. Although
these are not widely popular, they have some clear advantages over the
C functions for output. This allows us to rewrite Hello World as follows:

    #include <iostream>

    int main() {
      std::cout << "Hello, World!\n";
    }

The `stdout` stream is called `cout`, and we print the text by using the `<<`
operator on the stream and the text. This is overloaded functionality;
`operator<<` is still bit-shifting when it has two integral arguments, but when
the left argument is a stream it instead means "print to stream".
