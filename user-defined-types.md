# User-Defined Types

In [Basic Syntax](basic-syntax.md), I mentioned some of the ways of introducing
custom types in C: `enum`, `union` and `struct`, to be precise, with `typedef`
allowing you to define an alias for an existing type. I will cover these in more
detail here, and will also introduce some of the adaptations that C++ makes to
these things.

## Foreword on implicit conversions

Before I get into the different user-defined types, it is worth mentioning the
nature of type conversions in C and C++. With warnings disabled, almost all of
the scalar types in C++ will _implicitly_ convert to any other scalar type if
used in the right context. There are many constraints describing what should
happen when certain types are converted to certain other types, and in some
cases a conversion for which the destination type cannot represent the source
type will result in _undefined behaviour_. I would encourage you to read section
4 of the C++ language standard if you care about the specific details, but at
the very least **always compile your code with as many compiler warnings enabled
as possible**.

## `enum`

An `enum` is a type which consists of a collection of named integer values. You
may specify particular values for particular names, or use the default values.
By default, each name will take the value one higher than the previous name
defined in the `enum`, and the first name will receive the value `0`.

    enum MyEnum {
      FOO,  // Receives the default value: 0
      BAR,  // Receives the default value: 1
      BAZ = 100,  // User-specified value as 100
      BANG,  // Receives the default value 101
    };

A common use case for `enum` types is to define a collection of possible error
codes for a system. Using the named type in code instead of just using an `int`
makes the code clearer. However, since an `enum` has an underlying integer type,
it can be implicitly converted to any other integer type.

    MyEnum a = FOO;
    MyEnum b = 1;  // Compile error: no conversion from int to MyEnum.
    int x = a;  // Fine: implicit conversion from MyEnum to int.

Thanks to this property, it is common in C to use an anonymous `enum` to define
a collection of constant values. This is less common in C++ as there are many
other ways which are preferable.

The underlying integral type of an `enum` is implementation defined, but is
generally accepted to be an `int`. C++11 added syntax for specifying
a particular underlying type for `enum` types:

    enum MyEnum : unsigned char { ... };

This allows you to conserve space if there are only very few values and you are
intending to pack several instances together.

## `enum class`

C++11 added the `enum class` type. In principle, these serve the same purpose as
normal `enum` types. However, an `enum class` type cannot be implicitly
converted to any other type:

    enum class MyEnum { FOO, BAR };
    int x = MyEnum::FOO;  // Compile error: no implicit conversion available.

Names in an `enum class` have to be qualified. It was necessary to write
`MyEnum::Foo` here instead of just `FOO`. This allows multiple different `enum`
types to reuse the same name, which can be helpful when there are cases which
have particularly common names like `OK` or `SUCCESS`.

## `struct` and `class`

The `struct` and `class` keywords both introduce a new compound type. These are
probably the most familiar of the available types if you are coming from
a background in Object Oriented Programming. So, what is the difference between
them?

Short answer: nothing you can't change. The distinction made by C++ is that
`struct` is `public` by default, and `class` is `private` by default.

### Access specifiers

If you have used Java before, you have probably written `public`, `private`, and
possibly `protected` in a `class`. C++ also has these, although the syntax is
a little different. Instead of writing the word before every declaration, you
instead write it to denote a section:

    class MyClass {
     public:  // Start a public section.
      MyClass();  // Public constructor.
      void foo();  // Public member function.
      int x;  // Public variable.
     private:  // Start a private section.
      MyClass(int x, int y);  // Private constructor.
      void bar();  // Private member function.
      int y;  // Private variable.
     public:  // Start another public section.
      ...
    };

You may open the same section type multiple times, but generally it is best not
to. If you write definitions without any access specifier before them, they will
use the default access level. For `class` types, this is `private`. For `struct`
types, this is `public`. However, as you can see, you can start a `class` with
`public` and from that point on it is effectively the same as a `struct`.

Since `struct` and `class` are mostly interchangeable, you may choose to give
them semantic interpretations of their own. Personally, I use `struct` for types
which are public-only, data-only simple containers, and use `class` for types
which have data-type invariants to uphold, or have member functions.

The `public` versus `private` distinction also applies when inheriting from
another `class` or `struct`:

    class BaseClass {};

    class DerivedClass : BaseClass {};  // Private inheritance by default.
    struct DerivedStruct : BaseClass {};  // Public inheritance by default.

    class DerivedClass : public BaseClass {};  // Override default.
    struct DerivedStruct : private BaseClass {};  // Override default.

(Note that it is fine for a `struct` to inherit from a `class`, or vice versa).

Here we have `public` and `private` inheritance. You are probably familiar with
`public` inheritance, but a simple summary of `private` inheritance is that you
can write your class as if you are inheriting, but an instance of your class may
not be treated as an instance of its superclass by code outside of the class.

### Member Functions

In Object Oriented Programming, a _method_ is a function which is related to
a class in a special way. In C++, these are called _member functions_. A member
function has special properties when compared to a normal function. To start
with, a member function has unrestricted access to the internals of the class it
is declared in. Secondly, unless it is declared `static` it will always run with
an implicit reference to an instance of the class:

    class MyClass {
     public:
      void foo();
     private:
      int x;
    };

    void MyClass::foo() {
      x = 1;  // Sets x = 1 on the instance for which foo was called.
    }

    MyClass my_class;
    my_class.foo();

### Special Member Functions

There are several member functions in C++ which have special meaning or syntax.
These functions allow you to define properties of your class that decide how it
can be used. Many of these will be generated by default by the compiler where
possible:

  * `MyClass::MyClass()` - The _default constructor_, used to create new
    instances of your class when the constructor was not explicitly invoked with
    any parameters.
  * `MyClass::MyClass(const MyClass& other)` - The _copy constructor_, which is
    used to create a new instance of your class which is a copy of another
    instance.
  * `MyClass::MyClass(MyClass&& other)` - The _move constructor_, which is used
    to _move_ an instance of your class into another one. Unlike the copy
    constructor, this can disassemble the `other` instance. This allows the move
    constructor to, for instance, take ownership of internal buffers without
    having to copy them. Move constructors can be far more efficient than copy
    constructors for this reason.
  * `T MyClass::operator=(const MyClass& other)` - The _copy assignment
    operator_. Like the copy constructor, this is used to make a copy of another
    instance of the class. However, unlike the copy constructor this is copying
    another instance into an instance which already exists. This may have to do
    additional work like discarding the existing contents.
    
    The copy assignment operator may return different types depending on what
    usage you would like to support. The two most common options are to either
    return `void`, which is probably the most efficient, or to return
    a reference to this instance of the class by using `MyClass&` and `return
    *this;`, which allows you to chain assignment operations like `x = y = z`.
  * `T MyClass::operator=(MyClass&& other)` - The _move assignment operator_.
    What the copy assignment operator is to the copy constructor, the move
    assignment operator is to the move constructor.
  * `MyClass::~MyClass()` - The _destructor_, used to destroy an instance of
    your class when it is no longer needed.

Note the syntax for `operator=`. You may override the behaviour of many
operators in C++ using similar syntax. For example, `operator+` allows you to
define what addition means for your type, and `operator*()` allows you to define
what it means to dereference your type as if it were a pointer. It is tempting
to go overboard with operator overloading to make it easy to write short code
with your class, but it is generally wise to only do so if it is universally
familiar syntax for the type. You shouldn't overload operator `>=` for your type
to behave like the monadic bind operator from Haskell, for example.

### Constructor initialiser lists

The body of a constructor always runs after every member variable has been
initialised. This means that once code between the curly braces of the
constructor starts to run, all of the variables have already been initialised.
If we assign to them now, we use the assignment operators, not the constructors.
This can be wasteful. Additionally, if we have a `const` member variable, we
cannot assign to them at this stage.

The proper way to set the value of member variables of a class from
a user-defined constructor is to use the _constructor initialiser list_:

    struct Rectangle {
      Rectangle(int x, int y);
      const int width, height, area;
    };

    Rectangle::Rectangle(int x, int y)
        : width(x), height(y), area(x * y) {}

The constructor initialiser list follows the parameter list of the constructor.
It begins with a `:` and then has a comma-separated list of member variables
with values specified inside parentheses. Regardless of what order these are
specified in, they will always be executed in the order in which the member
variables are declared in the struct/class. To avoid compiler warnings and the
common bugs that they try to prevent, you should write this list in the same
order as the variables are declared.

### Initialiser lists and uniform initialisation syntax

_Uniform initialisation syntax_ was an attempt to solve some particular
ambiguities in the syntax for initialising C++ variables:

    class MyType {
     public:
      MyType();
      MyType(int x);
      MyType(int x, int y);
    };

    MyType foo(1, 2);  // Variable called foo.
    MyType bar(1);  // Variable called bar.
    // *Function declaration* for baz which takes nothing and returns MyType!
    MyType baz();

    // Attempt to use std::istreambuf_iterator to read from std::cin into
    // a std::string. Instead, this fails to compile because it is parsed as
    // a function declaration.
    std::string my_string(std::istreambuf_iterator<char>(std::cin),
                          std::istreambuf_iterator<char>());

The proposed solution was to allow writing `{` and `}` instead of `(` and `)`
for invoking constructors:

    MyType baz{};  // Variable called baz.
    // Successfully reads std::cin into a std::string.
    std::string my_string{std::istreambuf_iterator<char>(std::cin),
                          std::istreambuf_iterator<char>()};

C++11 also added _initialiser lists_ to the language. These provide a much more
convenient way of instantiating plain structs or invoking constructors:

    struct Rectangle {  // No user-defined constructor.
      int x = 100, y = 100;  // Default values!
    };

    class Circle {
     public:
      // Since we declared a custom constructor, the compiler-generated default
      // constructor is disabled.
      Circle(float radius) : radius_(radius) {}
      float radius() const { return radius_; }
      float area() const { return M_PI * radius_ * radius_; }
     private:
      const radius_;
    };

    Rectangle r1{};  // Default constructor. Rectangle is {x=100, y=100}.
    Rectangle r2{25};  // Only set x. Rectangle is {x=25, y=100}.
    Rectangle r3{25, 50};  // Set both. Rectangle is {x=25, y=50}.
    Circle c1{};  // Compiler error: no default constructor for Circle.
    Circle c2{20};  // Call the constructor for Circle.

Some types additionally define constructors which directly accept the
`initializer_list` type:

    std::vector<int> my_integers = {1, 2, 3, 4, 5};
    std::map<std::string, int> my_numbers = {
      {"foo", 1},
      {"bar", 42},
      {"baz", 9001},
    };

Strictly speaking, both of these cases invoke the `initializer_list` constructor
to create a temporary instance and then use the move constructor to transfer the
value into the type. However, the compiler can often elide the extra work and
the syntax for creating variables with an `=` is arguably more familiar.

## `union`

I briefly described the purpose of a `union` type in [Basic
Syntax](basic-syntax.md). A `union` type consists of exactly one of its fields:

    union MyUnion {
      Foo foo;
      Bar bar;
    };

Here, `MyUnion` _either_ contains `foo`, _or_ it contains `bar`, but it cannot
contain both. Prior to C++11, types which had user-defined constructors or
destructors could not be used inside `union` types because there was no way for
the compiler to know which constructor to call when creating an instance of the
`union` type. Since C++11, it is possible to use complex types inside `union`
types provided that you implement your own constructor and destructor for the
`union` type which can correctly destruct its contents.

Often, it is far more desirable to use a `std::variant` than to use a `union`
directly: `std::variant` is a C++17 feature which implements a _tagged union_,
meaning that unlike a native `union`, it also tracks which field is populated.
A `std::variant` will correctly construct and destruct its contents based on
what it contains, and it also supports safe access to the contents:

    #include <string>
    #include <variant>

    // Allow using "foo"s instead of std::string{"foo"}.
    using namespace std::literals;

    std::variant<int, std::string, Rectangle> my_thing = 42;
    my_thing = Rectangle{10, 20};  // Fine.
    my_thing = "Hello, World!"s;

## `using` declarations

In C, the keyword `typedef` can be used to introduce type aliases. However, the
syntax can be confusing, and it does not work with templates:

    typedef int MyInt;  // MyInt now means int.
    typedef void (*Bar)(int x, int y);  // Bar now means void(*)(int, int).

    template <typename T>
    typedef std::map<std::string, T> MyStringMap;  // Compile error.

C++ adds an alternative syntax for defining type aliases:

    using MyInt = int;
    using Bar = void(*)(int x, int y);

This syntax is widely considered to be easier to read, and on top of this it
supports being used as part of a template definition:

    template <typename T>
    using MyStringMap = std::map<std::string, T>;

    MyStringMap<int> my_map;  // Same as std::map<std::string, int>.
