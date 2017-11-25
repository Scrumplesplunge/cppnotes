# Smart Pointers

In [Value Types](value-types.md) I concluded by mentioning that there was
a better way of handling lifetime and ownership concerns than careful manual
application of `new` and `delete`. _Smart Pointers_ are the solution
I mentioned. In this entry I am going to build up to smart pointers by
introducing a few new concepts, and then cover how _and when_ to use smart
pointers.

## Initialisation and Destruction

Before we can get into smart pointers, I have to introduce a couple of things
which C++ has but C does not.

### Constructors

A _constructor_ is a special function which is responsible for creating an
instance of a `struct` or a `class`. If we do not write one, the compiler will
try to define a default one for us which recursively applies default
constructors for all of the member variables in the type.

Constructors have slightly different syntax from normal functions, and slightly
different behaviour:

    struct Rectangle {
      // We can declare multiple destructors as long as they take arguments of
      // different types, or a different number of arguments.
      Rectangle(int x, int y);
      
      int width;
      int height;
    };

    Rectangle::Rectangle(int x, int y) {
      width = x;
      height = y;
      std::cout << "Created a new rectangle with size "
                << width << "x" << height << ".\n";
    }

    int main() {
      // Define a variable called my_rectangle and call the constructor.
      Rectangle my_rectangle{3, 5};
    }

> There are a lot of details about this code which I have skipped over here.
> I will cover `struct` and `class` definitions, member functions, and concepts
> such as `public`, `private`, and inheritance in my page about _Object Oriented
> Programming_.
>
> TODO: Link to OOP page once written.

Whenever we create a new instance of `Rectangle`, either with _automatic storage
duration_ (ie. as a "value") or with _dynamic storage duration_ (ie. with `new`,
as a pointer value), the constructor happens automatically.

### Destructors

A _destructor_ is like the opposite of a constructor. It is called when the
value is no longer needed. If you have used `__del__` in Python, or `finalize`
in Java, this might seem familiar. Both of these functions execute just before
the memory for those objects are reclaimed. However, it is generally
unpredictable how much time will pass after you stop using a variable before
these functions run because the garbage collecter has to notice that they are no
longer in use. In contrast, C++ has very deterministic rules about when
destructors run:

    void Foo() {
      MyObject a;  // Constructor for a runs.
      MyObject b;  // Constructor for b runs.
      MyObject* c = new MyObject;  // Constructor runs for *c.
      if (SomeCondition()) {
        MyObject d;  // Constructor for c runs.
        ...
        // Destructor for d runs.
      }
      ...
      // Destructor for *c does not run!
      // Destructor for b runs.
      // Destructor for a runs.
    }

There are some interesting observations here:

  * The destructor for the value associated with a variable will always run when
    that variable goes out of scope.
  * Destructors will always run in the opposite order from the way they were
    introduced if multiple go out of scope at the same time.
  * Since `c` is a pointer, its _value_ is only the _address_ of the instance of
    `MyObject`, so the `MyObject` destructor does not run. Pointers are
    trivially destructible, meaning they have a destructor that does nothing.

The syntax of destructors is similar to that of constructors, but with a `~`
before the name of the type:

  struct MyType {
    MyType();  // Constructor.
    ~MyType();  // Destructor.
  };

  MyType::MyType() {}  // Constructor implementation.
  MyType::~MyType() {}  // Destructor implementation.

I'm not sure why this was the chosen syntax, but `~` is bitwise-not in C++, so
I like to think of it as `not MyType` because destructing it means it is not
a `MyType` any more :)

### Poor Man's std::string

We can use constructors and destructors to call `new` and `delete`. This allows
us to create types like `std::string` which can contain a variable amount of
data:

    namepace poor_mans {
  
    class String {
     public:
      String(const char* c_string);
      ~String();
      void PrintLine();
     private:
      int length_;
      char* data_;
    };
  
    String::String(const char* c_string) {
      // In C, strings are represented by a pointer to an array of chars. The
      // length is encoded by storing a zero-byte at the end, and promising not
      // to use any zero-bytes in the string itself. We can use strlen to
      // measure a c-string.
      length_ = std::strlen(c_string);
      data_ = new char[length_ + 1];  // Plus one for the zero-byte.
      for (int i = 0; i < length_; i++) data_[i] = c_string[i];
      data_[length_] = 0;
    }

    String::~String() {
      delete[] data_;  // We allocated an array, so we have to use delete[].
    }

    void String::PrintLine() { std::cout << data_ << "\n"; }

    }  // namespace poor_mans

    void Foo() {
      String test("Hello, World!");  // Constructor runs! new char[...] called!
      ...
      test.PrintLine();  // Prints "Hello, World!\n"
      ...
      // Destructor runs! delete[] called!
    }

_I want to make it very clear that you should not use this class. It has a lot
of bugs because I omitted things which are not relevant to the example. Just
focus on the constructor and destructor._

_Joe takes off his naysayer hat_.

## RAII

Since constructors and destructors happen at particular times, we can use them
to do a lot more than just set up or destroy the plain data that an object
contains. For example, we can use them to handle locking and unlocking a mutex:

    class Lock {
     public:
      Lock(Mutex* mutex);
      ~Lock();
     private:
      Mutex* mutex_;
    };

    Lock::Lock(Mutex* mutex) {
      mutex_ = mutex;
      mutex_->Lock();
    }

    Lock::~Lock() {
      mutex_->Unlock();
    }

    void BoringWay() {
      mutex.Lock();  // Manually lock the mutex.
      DoStuff();
      mutex.Unlock();  // Manually unlock. If we forget to do this, we deadlock.
    }

    void RaiiWay() {
      Lock lock(&mutex);  // Calls mutex.Lock().
      DoStuff();
      // mutex.Unlock() is called automatically! We don't have to remember!
    }

This saves us a line here, but imagine if `DoStuff()` was instead 20 lines of
code, with some loops and conditions and some _early return statements_. If we
returned from `BoringWay` in the middle, we would skip the `Unlock` and deadlock
next time we try to lock the mutex. With `RaiiWay`, the destructor simply runs
when we return. Even if we remember to write `mutex.Unlock()` before every
`return` statement, we will still deadlock if something `throw`s an exception
and something outside the function `catch`es it. But what happens with
`RaiiWay`? You guessed it; it works. The destructor will be automatically called
as the exception bubbles up, through a process known as _stack unwinding_.

Wrapping up resources like this so that we can take advantage of the automatic
and faultless destructor execution to make sure we handle resources properly is
known as _Resource Acquisition Is Initialisation_, or _RAII_. C++ uses it for
memory management, files, networking, concurrency primitives, and countless
other things.

## Introducing Smart Pointers

There is a principle in C++ called `the rule of three`, or `the rule of five`
for C++11 and onwards. The details are out of scope for this article, but one
thing which this principle mentions is that a type should _either_ be
responsible for lifetime or resource ownership, _or_ it should do neither of
these things and instead do something else. If we have a type which conceptually
has to do both, we should factor it into two: one that owns or managed
resources, and one which does things with those resources.

For this reason, it seems that our `poor_mans::String` class from above is
wrong. It handles the lifetime of its `data_` field, and it also does things
with that data. What we should really do is create a type for managing the data,
and then use this in our string:

    namespace poor_mans {

    class CharBuffer {
     public:
      // Precondition: data must have been allocated with new char[].
      CharBuffer(char* data);
      ~CharBuffer()
      char* get();
     private:
      char* data_;
    };

    CharBuffer::CharBuffer(char* data) { data_ = data; }
    CharBuffer::~CharBuffer() { delete[] data; }
    char* CharBuffer::get() { return data_; }

    }  // namespace poor_mans

Now that we have the `CharBuffer` type, we can simplify the `String` class:

    namespace poor_mans {

    class String {
     public:
      String(const char* c_string);
      // We don't need a custom destructor any more! The default one is enough!
      void PrintLine();
     private:
      int length_;
      CharBuffer buffer_;
    };

    // This syntax may look new and strange, but it is necessary here. Check out
    // my page about user-defined types to learn more about it.
    String::String(const char* c_string)
        : length_(std::strlen(c_string)),
          buffer_(new char[length_ + 1]) {
      for (int i = 0; i < length_; i++) buffer_.get()[i] = c_string[i];
      buffer_.get()[length_] = 0;
    }

    void String::PrintLine() { std::cout << buffer_.get() << "\n"; }

    }  // namespace poor_mans

Now that the `String` class no longer has to handle the lifetime of its data, it
doesn't need a custom destructor any more. The `CharBuffer` class that we have
created is basically just a pointer with a clever destructor. Other than the
fact that we now have to write `buffer_.get()[i]` instead of `buffer_[i]`, this
seems like a win-win.

### `std::unique_ptr`

What we have just written is a _smart pointer_, albeit certainly not as well as
the standard library version of the same thing: `std::unique_ptr`.
A `unique_ptr` is a type which holds a pointer to a value. It is called
`unique_ptr` because it is the "unique" owner of the data. You can't copy
a `unique_ptr` into another `unique_ptr`. We could entirely replace our use of
`CharBuffer` above by using `std::unique_ptr<char[]>`.  Furthermore,
`std::unique_ptr<char[]>` both supports the `*buffer_` syntax to access the
object that it points at, and also using the square-bracket syntax directly, so
we can write `buffer_[i]` again instead of `buffer_.get()[i]`.

`std::unique_ptr` is a _zero cost abstraction_. When used properly, the compiled
program that uses a `std::unique_ptr` is never slower than the code you would
write manually to do the same thing. On top of this, because it is an RAII type
it can save us from all sorts of memory leaks that might come from early
returns, exceptions being thrown, or other easy mistakes.

### `std::shared_ptr`

A less common issue to face, but nonetheless one which occasionally crops up, is
when it is not really possible to have only one owner for a resource. Suppose
that we wanted to save memory with our `String` class by having multiple
instances of the same string share the same data buffer. We didn't add any way
for someone to edit the contents of our string, so it seems safe to do this.
However, we have the following problem:

    String* a = new String{"Hello, World!"};
    String b{*a};  // We want to support copying like this.
    delete a;  // We don't want this to delete the buffer - b still needs it.

A simple solution would be to use a _reference counter_. We create a counter
which we set to 1 when we create the buffer. When we add another string that
uses the buffer, we increment the counter. When a string is destructed, we
decrement it. Once the number reaches zero, we have destroyed the last string
and we can clean up the buffer.

We could implement this like so:

    class ReferenceCountedBuffer {
     public:
      // Create a new reference counted buffer with count=1.
      // Precondition: data must be created with new char[].
      ReferenceCountedBuffer(char* data);
      // Create a new reference to an existing reference-counted buffer.
      ReferenceCountedBuffer(const ReferenceCountedBuffer& other);
      // Destroy this reference to the buffer.
      ~ReferenceCountedBuffer();

      char* get();

     private:
      int* counter_;
      char* data_;
    };

    ReferenceCountedBuffer::ReferenceCountedBuffer(char* data) {
      // Note that the new expression has parentheses, not square brackets. This
      // sets the counter to 1.
      counter_ = new int(1);
      data_ = data;
    }

    ReferenceCountedBuffer::ReferenceCountedBuffer(
        const ReferenceCountedBuffer& other) {
      counter_ = other.counter_;  // Use the existing counter.
      (*counter_)++;  // Increment the reference count.
      data_ = data;
    }

    ReferenceCountedBuffer::~ReferenceCountedBuffer() {
      (*counter_)--;  // Decrement the reference count.
      if (counter_ != 0) return;  // There are still other references.
      // This was the last reference. Delete the data *and* the counter.
      delete[] data_;
      delete counter_;
    }

    char* ReferenceCountedBuffer::get() { return data_; }

If we replace `CharBuffer` with `ReferenceCountedBuffer` now, we can add
a second constructor for our string class:

    String::String(const String& other)
        : length_(other.length_), buffer_(other.buffer_) {}

    void Demo() {
      // Now we can do this:
      String* temp = new String{
          "I am a string with a huuuuuuuu....ge amount of data!"};
      String b{*temp};  // Very cheap! Doesn't have to copy all of the bytes!
      delete temp;  // Doesn't delete the buffer!
      // b goes out of scope, reference count hits zero, buffer is deleted!
    }

This reference counter works, but a lot more care is needed if you want to
handle multithreaded access or cleverly call `delete` or `delete[]` depending on
what type you are storing. The standard library provides `std::shared_ptr` to
achieve this. It is a reference counted pointer with the same interface as
`std::unique_ptr` except that it is also copy-assignable. Our
`ReferenceCountedBuffer` is essentially a bad implementation of
`std::shared_ptr<char[]>`.

### `std::weak_ptr`

One of the main drawbacks with reference counted memory management is what
happens when we introduce a cyclic dependency. Consider this example:

    struct LinkedListNode {
      std::shared_ptr<LinkedListNode> previous, next;
      int value;
    };

    void CycleDemo() {
      // std::make_shared is a helper function for creating an instance of
      // std::shared_ptr<T> for a given T. It has a slight performance advantage
      // over writing std::shared_ptr<T>(new T), too.
      std::shared_ptr<LinkedListNode> foo = std::make_shared<LinkedListNode>();
      std::shared_ptr<LinkedListNode> bar = std::make_shared<LinkedListNode>();
      foo->next = bar;  // Reference count for bar increases to 2.
      foo->value = 1;
      bar->previous = foo;  // Reference count for foo increases to 2.
      bar->value = 2;
      // What happens here?
    }

When we reach the end of `CycleDemo`, both `foo` and `bar` go out of scope and
the reference counts for their values both decrease by one. However, both of
them still have references, so neither of them are destructed. Because neither
of them are destructed, neither of the remaining references ever go away and we
leak both of the nodes!

The conventional solution to this is to try to avoid writing code which can
result in cycles of `std::shared_ptr`. If this can be done trivially, then
great. If it is absolutely necessary to have cyclic access, there is
a compromise available: `std::weak_ptr`.

`std::weak_ptr` is like a `std::shared_ptr`, and is created from
a `std::shared_ptr`, but it does not increase the reference count. Instead, it
pretends that it is not a reference. When you want to read the value in
a `std::weak_ptr` you must temporarily `lock` it, which turns it into
a `std::shared_ptr`. Since a `std::weak_ptr` does not increase the reference
count, the value will be destroyed if all the `std::shared_ptr` instances are
destroyed. If this is the case, the `lock` will fail.

    struct WeakLinkedListNode {
      std::weak_ptr<WeakLinkedListNode> previous;
      std::shared_ptr<WeakLinkedListNode> next;
      int value;
    };

    void AccessPrevious(std::shared_ptr<WeakLinkedListNode> node) {
      std::shared_ptr<WeakLinkedListNode> previous = node.previous.lock();
      if (previous) {
        std::cout << "Previous value is " << previous->value << "\n";
      } else {
        std::cout << "There is no previous value. Either it did not exist or "
                     "all of the references were destroyed and it "
                     "disappeared.\n";
      }
    }

    void WeakCycleDemo() {
      std::shared_ptr<WeakLinkedListNode> foo =
          std::make_shared<WeakLinkedListNode>();
      std::shared_ptr<WeakLinkedListNode> bar =
          std::make_shared<WeakLinkedListNode>();
      foo->next = bar;  // Reference count for bar increases to 2.
      foo->value = 1;
      bar->previous = foo;  // Reference count for foo stays at 1.
      bar->value = 2;

      AccessPrevious(bar);

      // What happens now?
    }

When we reach the end of `WeakCycleDemo`, `foo` and `bar` go out of scope. Since
`foo` was constructed before `bar`, `bar` is destructed before `foo`. When this
happens, `bar`'s reference count decreases to 1. This is not zero, so the node
is not deleted. Then, `foo` is destructed. Since there is only a `std::weak_ptr`
to `foo`'s value in `bar`, this drops the reference count to 0 and the value
`foo` points at is also destructed. This in turn causes the remaining reference
to the value of `bar` to disappear and that value is also deleted.

## When to use pointers to values

So far we have only used smart pointers to access arrays of values. Is there
ever a reason why we should use a smart pointer to access a _single_ value
instead of just creating that value directly with automatic storage duration?

Short answer: yes. Long answer: always default to using automatic variables
instead of dynamic ones. You should only use dynamic storage duration if you
have a reason to do so. A few reasons you might do so are:

  * If there is no real correlation between the lifetime of your type and the
    lifetime of one of its member variables, you might consider using pointers.
    You could have a non-owning pointer to another automatic variable as long as
    that automatic variable will always outlive your type, or you can just use
    a smart-pointer and give ownership to your type. It can pass ownership to
    something else when it no longer wants to own it.
  * If you are using _runtime polymorphism_ you cannot store values. The way
    I got my head around this was by imagining the size of the types. If the
    `Derived` type adds an extra variable, it requires extra space. Since we
    have value semantics, we would have to store that in the `Base`, but there
    isn't any room. If we just use pointers to the values instead, the size is
    always just the size of the pointer and the data can be stored somewhere
    else where we have some room. If we want to handle ownership at the same
    time, we can just use smart pointers:
   
        class Base { ... };
        class Derived : public Base { ... };

        void Test() {
          // This does *not* work as expected (but it does compile... be wary).
          Base base;
          Derived derived;
          base = derived;
          // This works.
          std::unique_ptr<Base> base = std::make_unique<Derived>();
        }

  * If the object you are allocating is *huge*, you might run into issues if you
    try to use automatic storage duration. This is because automatic variables
    are usually implemented with a call stack, and most operating systems put
    a cap on the size of the call stack for processes. In my case, there is
    a limit of 8MB for the initial stack of any program. This means that if
    I write:

        void Bad() {
          // std::array uses automatic storage for all of its contents.
          std::array<char, 10000000> test;  // For me, sizeof(test) == 10000000
          test[9999999] = 1;
        }

        void Good() {
          // std::vector uses a dynamic storage buffer for its contents. Very
          // little space is used on the stack, and 10000000 bytes are allocated
          // from the free-store instead.
          std::vector<char> test(10000000);  // For me, sizeof(test) == 24
          test[9999999] = 1;
        }

    Calling `Bad` will almost certainly crash (unless the compiler optimizes out
    the whole array) while calling `Good` will almost certainly succeed (unless
    I don't have 10MB of spare memory available for my silly experiment).
