---
title: "Usability Improvement II"
description: "Literals, Member Function"
summary: Modern C++ Series - Part 09
date: 2025-06-27
series: ["C++"]
weight: 1
tags: ["Modern C++", "Literal", "Member Function", "Override", "Final"]
author: ["Garry Chen"]
cover:
  image: images/cpp-09-00.jpg
  hiddenInList: true
  caption: "Literal"

---


# User-Defined Literals

A *literal* refers to a fixed constant written directly in source code. In C++98, literals could only be of built-in types, such as:

* `"hello"` — a string literal, type `const char[6]`
* `1` — an integer literal, type `int`
* `0.0` — a floating-point literal, type `double`
* `3.14f` — a floating-point literal, type `float`
* `123456789ul` — an unsigned long integer literal, type `unsigned long`

C++11 introduced *user-defined literals*, allowing you to use the `operator""` suffix to convert user-defined literals into actual types.

C++14 expanded this by adding many standard literals to the standard library. The following program demonstrates how they are used:

```cpp
#include <chrono>
#include <complex>
#include <iostream>
#include <string>
#include <thread>

using namespace std;

int main()
{
  cout << "i * i = " << 1i * 1i << endl;
  cout << "Waiting for 500ms" << endl;
  this_thread::sleep_for(500ms);
  cout << "Hello world"s.substr(0, 5) << endl;
}
```

Output:

```
i * i = (-1,0)
Waiting for 500ms
Hello
```

This example shows standard C++ literal suffixes that help create imaginary numbers, time durations, and `basic_string` literals. One thing to note: I used `using namespace std`, which brings in not only `std` but also its inline namespaces, including the namespaces for the literal operators:

* `std::literals::complex_literals`
* `std::literals::chrono_literals`
* `std::literals::string_literals`

In production code, it’s generally discouraged (and should be avoided) to use `using namespace std` globally. But for brevity, examples like this one use it. If you're avoiding global `using`, you should import only the necessary namespaces within the appropriate scope. For example:

```cpp
using namespace std::literals::chrono_literals;
```


Adding support for literals in your own classes is fairly easy. The only restriction is that non-standard literal suffixes must begin with an underscore (`_`). For instance, suppose we have a length class:

```cpp
struct length {
  double value;
  enum unit {
    metre, kilometre, millimetre, centimetre,
    inch, foot, yard, mile,
  };
  static constexpr double factors[] = {
    1.0, 1000.0, 1e-3,
    1e-2, 0.0254, 0.3048,
    0.9144, 1609.344
  };
  explicit length(double v, unit u = metre) {
    value = v * factors[u];
  }
};

length operator+(length lhs, length rhs) {
  return length(lhs.value + rhs.value);
}
```

You *could* write something like `length(1.0, length::metre)`—but most developers wouldn’t want to. Instead, a more pleasant syntax would be:

```cpp
1.0_m + 10.0_cm
```

To support that syntax, we just define these operators:

```cpp
length operator"" _m(long double v) {
  return length(v, length::metre);
}

length operator"" _cm(long double v) {
  return length(v, length::centimetre);
}
```



# Binary Literals

You probably know that C++ supports the `0x` prefix to write hexadecimal literals like `0xFF`. There’s also an older, lesser-used prefix: a leading `0` followed by digits `0–7` for octal values. This still shows up, especially in filesystem-related code—seasoned Unix programmers may find `chmod(path, S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH)` no more readable than `chmod(path, 0644)`.

Since C++14, binary literals are now supported using the `0b` prefix:

```cpp
unsigned mask = 0b111000000;
```

This is useful in bit-level operations and low-level programming.

Unfortunately, the I/O stream manipulators (`dec`, `hex`, and `oct`) still don’t include `bin`, so there’s no direct way to print a number in binary like there is for decimal, hexadecimal, or octal. A workaround is to use `std::bitset`, though the user has to manually specify the number of bits:

```cpp
#include <bitset>
cout << bitset<9>(mask) << endl;
```

Output:

```
111000000
```



# Digit Separators

When numbers get long, it becomes harder to read them clearly. With the introduction of binary literals, this issue becomes even more noticeable. Starting with **C++14**, you can insert single quotes (`'`) anywhere in a numeric literal to make it more readable. How you group the digits is entirely up to you, but here are some common conventions:

* Decimal numbers grouped by 3 digits, matching English units like thousand, million, etc.
* Decimal numbers grouped by 4 digits, matching Chinese units like 万 (ten-thousand), 亿 (hundred-million).
* Hexadecimal numbers grouped by 2 or 4 digits, corresponding to bytes or double-bytes.
* Binary numbers grouped by 3 digits, useful for Unix file permission modes.

Examples:

```cpp
unsigned mask = 0b111'000'000;
long r_earth_equatorial = 6'378'137;
double pi = 3.14159'26535'89793;
const unsigned magic = 0x44'42'47'4E;
```



# default and delete Special Member Functions

In C++, certain special member functions can be automatically generated by the compiler, including:

* Default constructor
* Destructor
* Copy constructor
* Copy assignment operator
* Move constructor
* Move assignment operator

Each of these can be:

* Implicitly declared or user-declared
* Defaulted (i.e., automatically provided) or user-defined
* Active or deleted

These states can combine in various ways, though not all combinations are valid. For instance:

* If it’s implicitly declared, it must be defaulted.
* Only defaulted functions can be deleted.
* If it’s user-provided, it must be user-declared.

By default, if no special constraints exist (like `const` or reference members), and the user doesn't declare anything explicitly, the compiler automatically provides these member functions.

However, if you have user-declared constructors or complex member types, the situation changes:

* Uninitialized `const` or reference members can cause the compiler to delete the default constructor.
* Same applies to copy/move constructors and assignment operators.
* If the user doesn't declare a copy constructor, the compiler will implicitly declare one (not as a template).
* Same for copy assignment.
* If the user declares a move constructor or move assignment, the compiler will delete the copy operations.
* If none of the copy/move/assignment/destructor functions are declared, the compiler will generate move operations.

You don’t need to memorize all of this. Instead, focus on understanding the rationale through hands-on experience. In general, the rules are conservative to avoid unsafe defaults, particularly around move semantics.


You can override the default behavior using `= default` to explicitly request a default implementation or `= delete` to disable it. For example:

Without default constructor:

```cpp
template <typename T>
class my_array {
public:
  my_array(size_t size);
private:
  T* data_{nullptr};
  size_t size_{0};
};
```

If you want a default constructor but also have custom ones, you'd need to write it manually:

```cpp
my_array() : data_(nullptr), size_(0) {}
```

But with default member initialization, you can now simplify:

```cpp
my_array() = default;
```

To disable copying:

```cpp
class shape_wrapper {
  shape_wrapper(const shape_wrapper&) = delete;
  shape_wrapper& operator=(const shape_wrapper&) = delete;
};
```

In pre-C++11, you'd typically declare these functions as `private` to restrict copying, but `= delete` provides clearer intent and better error messages.

Note: Declaring a function as `= delete` is still considered a declaration—so if you delete copy functions, the compiler won’t generate move functions.



# override and final Specifiers

C++11 introduced two specifiers: `override` and `final`. They are not keywords—just markers that apply only when placed at the end of a member function declaration.

`override` Declares that this function overrides a virtual function in the base class. If it doesn’t (due to a typo or signature mismatch), the compiler will issue an error.

Benefits:

* Clearly signals your intent to override
* Helps the compiler catch mismatches early

`final`Declares that:

* A virtual function cannot be overridden by derived classes
* Or a class cannot be inherited from

Usage example:

```cpp
class A {
public:
  virtual void foo();
  virtual void bar();
  void foobar(); // Not virtual
};

class B : public A {
public:
  void foo() override;        // OK
  void bar() override final;  // OK
  // void foobar() override;  // Error: Not a virtual function
};

class C final : public B {
public:
  void foo() override;        // OK
  // void bar() override;     // Error: `bar` is final
};

class D : public C {
  // Error: Class `C` is final and cannot be inherited from
};
```

---