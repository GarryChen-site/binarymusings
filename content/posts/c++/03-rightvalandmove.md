---
title: "C++ Move Semantics Guide: Rvalue References & Forwarding"

description: "C++ move semantics and rvalue references - value categories, perfect forwarding, performance optimization, and modern C++ best practices."

summary: "Understand rvalue references, value categories, perfect forwarding, and performance optimization techniques."

date: 2025-06-19
series: ["C++"]
weight: 1
tags: ["C++", "Move Semantics", "Rvalue References", "Performance", "Modern C++"]
author: ["Garry Chen"]
cover:
  image: images/cpp-03-00.jpg
  hiddenInList: true
  caption: "Move semantics"

---


## Value Categories

We often say that in C++, there are lvalues and rvalues. But that’s not entirely accurate. The C++ standard defines value categories more precisely:

![value category](images/cpp-03-01.png)

Let’s first try to understand these terms literally:

* An **lvalue** is an expression that can usually appear on the left-hand side of an assignment.
* An **rvalue** is an expression that can usually only appear on the right-hand side of an assignment.
* A **glvalue** is a generalized lvalue.
* An **xvalue** is an expiring value.
* A **prvalue** is a pure rvalue.

Still confusing, right? For now, let’s focus only on **lvalues** and **prvalues**.

An **lvalue** is an expression that has an identifier and can have its address taken. The most common examples are:

* The name of a variable, function, or class member
* Expressions that return an lvalue reference, such as `++x`, `x = 1`, or `cout << ' '`
* String literals like `"hello world"`

When calling functions, an lvalue can bind to a parameter of lvalue reference type (e.g., `T&`). Constants can only bind to const lvalue references (`const T&`).

On the other hand, a **prvalue** is an expression that has no identifier and whose address cannot be taken; we often call these “temporary objects.” The most common examples are:

* Expressions that return by value (non-reference), such as `x++`, `x + 1`, or `make_shared<int>(42)`
* Literals (except string literals), like `42`, `true`, etc.

Before C++11, rvalues could bind to const lvalue references (`const T&`) but not to non-const lvalue references (`T&`). Starting with C++11, the language introduced **rvalue references**: `T&&`. Like lvalue references, rvalue references can also be modified by `const` and `volatile`, but typically we don’t use those qualifiers for rvalue references.

Introducing rvalue references adds some complexity to the language but also opens up many optimization opportunities. Since C++ supports function overloading, we can overload functions based on different reference types to implement different behaviors. For example, earlier we used overloads to allow the `smart_ptr` constructor to behave differently:

```cpp
template <typename U>
smart_ptr(const smart_ptr<U>& other) noexcept {
  ptr_ = other.ptr_;
  if (ptr_) {
    other.shared_count_->add_count();
    shared_count_ = other.shared_count_;
  }
}

template <typename U>
smart_ptr(smart_ptr<U>&& other) noexcept {
  ptr_ = other.ptr_;
  if (ptr_) {
    shared_count_ = other.shared_count_;
    other.ptr_ = nullptr;
  }
}
```

You may wonder: is the parameter `other` in the second overload (the one with rvalue reference) an lvalue or rvalue?
By definition, `other` is a variable name — it has an identifier and an address — so it’s still an **lvalue**, even though its type is an rvalue reference.

In particular, when you pass `other` to another function, it will match overloads for lvalue references. In other words:

> **A variable of rvalue reference type is still an lvalue.**

This may feel counterintuitive, but it’s consistent with C++. After all, you can take the address of an rvalue reference variable, just like with any lvalue. We’ll return to this point later.

Now consider this code:

```cpp
smart_ptr<shape> ptr1{new circle()};
smart_ptr<shape> ptr2 = std::move(ptr1);
```

In the first expression, `new circle()` is a **prvalue**. But since we typically pass pointers by value, we don’t really care whether it’s an lvalue or rvalue.

The second expression is more interesting. `std::move(ptr1)` forcibly converts an lvalue reference into an rvalue reference without modifying its contents. In practical terms, `std::move(ptr1)` is equivalent to `static_cast<smart_ptr<shape>&&>(ptr1)`. So `std::move(ptr1)` results in an rvalue reference to `ptr1`, allowing the move constructor to be called.

You can think of `std::move(ptr1)` as a **named rvalue**. To distinguish it from unnamed pure rvalues (`prvalue`), C++ calls such expressions **xvalues** (expiring values). Unlike lvalues, **xvalues** cannot have their address taken — which they share in common with prvalues. Both xvalues and prvalues fall under the broader category of **rvalues**. This diagram may help clarify:

![another angle of value category](images/cpp-03-02.png)

Finally, note that **value category** (lvalue/rvalue) and **value type** (value type/reference type) are two entirely separate concepts.

* Value category refers to the above classification of expressions.
* Value type refers to whether a variable represents an actual value or a reference to another value.

In C++:

* Built-in types, enums, structs, unions, and classes are value types.
* References (`&`) and pointers (`*`) are reference types.

In Java:

* Primitives (like numbers) are value types.
* Classes are reference types.

In Python:

* Everything is a reference type.


## Lifetime and Expression Types

The lifetime of a variable ends when it goes out of scope. If the variable represents an object, the object’s lifetime also ends at that point.
But what about temporary objects (prvalues)? In C++, the rule is: a temporary object is destroyed after the full expression containing it has been evaluated, in the reverse order of their creation — unless *lifetime extension* occurs. Let’s first look at the basic case where no lifetime extension happens:

```cpp
process_shape(circle(), triangle());
```

Here, we create two temporary objects — a circle and a triangle — which are destroyed after `process_shape` finishes executing and its result object is created.

By adding some real code, we can demonstrate this behavior:

```cpp
#include <stdio.h>

class shape {
public:
  virtual ~shape() {}
};

class circle : public shape {
public:
  circle() { puts("circle()"); }
  ~circle() { puts("~circle()"); }
};

class triangle : public shape {
public:
  triangle() { puts("triangle()"); }
  ~triangle() { puts("~triangle()"); }
};

class result {
public:
  result() { puts("result()"); }
  ~result() { puts("~result()"); }
};

result process_shape(const shape& shape1, const shape& shape2) {
  puts("process_shape()");
  return result();
}

int main() {
  puts("main()");
  process_shape(circle(), triangle());
  puts("something else");
}
```

The output may look like this (the order of circle and triangle is unspecified by the standard):

```
main()
circle()
triangle()
process_shape()
result()
~result()
~triangle()
~circle()
something else
```

I had `process_shape` return a `result` object so we can demonstrate something later. You can see that the result object is created last but destroyed first.

To make temporary objects easier to use, C++ has a special **lifetime extension rule**:

> If a prvalue is bound to a reference, its lifetime is extended to match the lifetime of that reference.

We only need to modify one line in the above code to demonstrate this effect. Replace the call to `process_shape` with:

```cpp
result&& r = process_shape(circle(), triangle());
```

Now, the output changes to:

```
main()
circle()
triangle()
process_shape()
result()
~triangle()
~circle()
something else
~result()
```

The creation of `result` still happens at the same place, but its destruction is delayed until the end of `main()`.

⚠️ **Note carefully:**
This lifetime extension rule applies only to **prvalues**, not to **xvalues**. If, for any reason, a prvalue is converted into an xvalue before being bound to a reference, lifetime extension does not occur. Ignoring this can cause subtle bugs.

For example, if we modify the code like this:

```cpp
#include <utility>  // for std::move
…
result&& r = std::move(process_shape(circle(), triangle()));
```

Then we’re back to the previous output pattern. Even though `r` still exists when `something else` is printed, the object it points to has already been destroyed. Dereferencing `r` here leads to **undefined behavior**. Since `r` points to stack memory, it may not cause a crash immediately, but may lead to issues under certain circumstances.


## The Meaning of Move Semantics

So far, we’ve discussed some syntax rules. Like learning grammar in a foreign language, these details can feel dry. They’re sometimes useful but often only make sense in hindsight. For beginners, it’s more important to understand **why** and to master the basics.

In `smart_ptr`, we use rvalue references to implement **move semantics**, which help reduce runtime overhead. For reference-counted pointers, the overhead difference between copy and move is small:

* Move constructors avoid one call to `other.shared_count_->add_count()`.
* The moved-from pointer is nulled, avoiding one call to `shared_count_->reduce_count()` during destruction.

But with **containers**, move semantics are much more impactful.
Consider the following example (assuming `name` is of type `string`):

```cpp
string result = string("Hello, ") + name + ".";
```

Before C++11, this code was not recommended due to inefficiency. The execution roughly went like this:

1. `string(const char*)` creates temporary object 1; `"Hello, "` copied once.
2. `operator+(const string&, const string&)` creates temporary object 2; `"Hello, "` copied twice, `name` copied once.
3. `operator+(const string&, const char*)` creates object 3; `"Hello, "` copied three times, `name` copied twice, `"."` copied once.
4. If Return Value Optimization (RVO) works, object 3 may be constructed directly into `result`.
5. Temporary object 2 destructs, freeing memory for `string("Hello, ") + name`.
6. Temporary object 1 destructs, freeing memory for `string("Hello, ")`.

Since C++ prioritizes performance, a good C++ programmer would write:

```cpp
string result = "Hello, ";
result += name;
result += ".";
```

This version calls the constructor once, calls `string::operator+=` twice, and creates no temporaries — every string is copied only once.
However, the code is much more verbose, especially when concatenating many strings. Starting from C++11, this is no longer necessary.

The one-line version now works like this:

1. `string(const char*)` creates temporary object 1; `"Hello, "` copied once.
2. `operator+(string&&, const string&)` appends directly onto temporary 1, moving the result into temporary 2; `name` copied once.
3. `operator+(string&&, const char*)` appends directly onto temporary 2, moving the result into `result`; `"."` copied once.
4. Temporary 2 destructs (already empty; nothing freed).
5. Temporary 1 destructs (already empty; nothing freed).

In terms of performance, every string gets copied only once. Though additional temporaries are created and destroyed, these involve no memory allocation, so they’re cheap. The programmer sacrifices just a bit of performance for much better code readability.
And even this "performance sacrifice" is only relative to highly optimized C or C++ code — this C++ code would still vastly outperform equivalent code written in languages like Python.

A key point is that C++ objects are **value types by default**.
Consider:

```cpp
class A {
  B b_;
  C c_;
};
```

In many other languages — like Java and Python — `A` would store pointers to `B` and `C` (even though these languages abstract away pointers).
In C++, `B` and `C` are stored directly inside `A`'s memory layout.

This approach has both pros and cons:

* **Advantage:** Better memory locality, which is a huge performance benefit on modern CPUs.
* **Disadvantage:** Copying objects becomes more expensive, since C++ copies the entire object, not just a pointer like Java does.

That’s why C++ needs **move semantics** as an optimization, while Java-like languages don’t even require the concept.

**Move semantics** make it feasible to return large objects (such as containers) from functions and operators in C++. This improves code **clarity**, **readability**, and **developer productivity** without sacrificing performance.

All modern C++ standard containers are heavily optimized for move operations.


## How to Implement Move Semantics?

To make your objects support move semantics, you usually need to do the following:

1. Your object should have separate **copy constructor** and **move constructor** (unless you intend to support only moves and not copies — like `unique_ptr`).
2. Your object should have a **member `swap` function**, to quickly exchange members with another object.
3. In your object’s namespace, provide a global `swap` function that calls the member `swap` function. Supporting this makes it easier for others (and yourself) to include your object in other objects and easily implement their `swap` functions.
4. Implement a general-purpose `operator=`.
5. All these functions should be marked `noexcept` if they don't throw exceptions. This is especially important for move constructors.

Our current `smart_ptr` implementation already demonstrates this:

#### Copy constructor:

```cpp
smart_ptr(const smart_ptr& other) noexcept {
  ptr_ = other.ptr_;
  if (ptr_) {
    other.shared_count_->add_count();
    shared_count_ = other.shared_count_;
  }
}
```

#### Templated copy constructor:

```cpp
template <typename U>
smart_ptr(const smart_ptr<U>& other) noexcept {
  ptr_ = other.ptr_;
  if (ptr_) {
    other.shared_count_->add_count();
    shared_count_ = other.shared_count_;
  }
}
```

#### Move constructor:

```cpp
template <typename U>
smart_ptr(smart_ptr<U>&& other) noexcept {
  ptr_ = other.ptr_;
  if (ptr_) {
    shared_count_ = other.shared_count_;
    other.ptr_ = nullptr;
  }
}
```

The move constructor takes resources from another object, clears that object’s resources, and leaves it in a destructible state.

#### Member `swap` function:

```cpp
void swap(smart_ptr& rhs) noexcept {
  using std::swap;
  swap(ptr_, rhs.ptr_);
  swap(shared_count_, rhs.shared_count_);
}
```

#### Global `swap` function:

```cpp
template <typename T>
void swap(smart_ptr<T>& lhs, smart_ptr<T>& rhs) noexcept {
  lhs.swap(rhs);
}
```

#### General-purpose `operator=`:

To make assignment safe even for `a = a;`, we can implement a trick that works for both lvalues and rvalues and avoids using `if (&rhs != this)`:

```cpp
smart_ptr& operator=(smart_ptr rhs) noexcept {
  rhs.swap(*this);
  return *this;
}
```


## Do Not Return References to Local Variables

A common C++ mistake is to return a reference to a local object. Since local objects are destroyed when the function ends, returning a reference to them results in **undefined behavior**. Anything may happen.


Before C++11, returning a local object usually meant copying it, unless the compiler could apply **Named Return Value Optimization (NRVO)** to construct the object directly in the caller’s stack frame.

Since C++11, RVO still applies where possible. If RVO is not possible, the compiler will attempt to **move** the local object instead of copying it. This happens automatically; using `std::move` manually here doesn't help — it may even disable RVO.

example:

```cpp
#include <iostream>
#include <utility>

using namespace std;

class Obj {
public:
  Obj() { cout << "Obj()" << endl; }
  Obj(const Obj&) { cout << "Obj(const Obj&)" << endl; }
  Obj(Obj&&) { cout << "Obj(Obj&&)" << endl; }
};

Obj simple() {
  Obj obj;
  return obj;  // NRVO likely
}

Obj simple_with_move() {
  Obj obj;
  return std::move(obj);  // disables NRVO
}

Obj complicated(int n) {
  Obj obj1;
  Obj obj2;
  if (n % 2 == 0) {
    return obj1;
  } else {
    return obj2;
  }
}

int main() {
  cout << "*** 1 ***" << endl;
  auto obj1 = simple();
  cout << "*** 2 ***" << endl;
  auto obj2 = simple_with_move();
  cout << "*** 3 ***" << endl;
  auto obj3 = complicated(42);
}
```

The output might be:

```
*** 1 ***
Obj()
*** 2 ***
Obj()
Obj(Obj&&)
*** 3 ***
Obj()
Obj()
Obj(Obj&&)
```

Using `std::move` actually disables NRVO.



## Reference Collapsing and Perfect Forwarding

Finally, let’s talk about something a bit more complex: **Reference Collapsing** (also called "Reference Folding").
You will encounter this in generic programming, so it’s worth understanding since we’re discussing lvalue and rvalue references.

We know that for a type `T`:

* An lvalue reference is `T&`.
* An rvalue reference is `T&&`.

Now:

* Is `T&` always an lvalue reference? **Yes.**
* Is `T&&` always an rvalue reference? **No.**

The key issue comes from type deduction in templates. Let’s simplify:

For a template function like:

```cpp
template <typename T>
void foo(T&& param);
```

* If you pass an **lvalue**, `T` will deduce to `T&`.
* If you pass an **rvalue**, `T` will deduce to the object type itself.

Now:

* If `T` is `type&`, then `T&&` becomes `type& &&`, which **collapses** into `type&` (an lvalue reference).
* If `T` is `type`, then `T&&` remains `type&&` (an rvalue reference).


Remember: rvalue reference **variables** are still lvalues when used. This can be demonstrated:

```cpp
void foo(const shape&) { puts("foo(const shape&)"); }
void foo(shape&&) { puts("foo(shape&&)"); }

void bar(const shape& s) {
  puts("bar(const shape&)");
  foo(s);
}

void bar(shape&& s) {
  puts("bar(shape&&)");
  foo(s);
}

int main() {
  bar(circle());
}
```

The output will be:

```
bar(shape&&)
foo(const shape&)
```

Even though `bar` received an rvalue, the variable `s` inside `bar` is still an lvalue, so it calls the `foo(const shape&)` overload.
If we want `bar` to call the rvalue overload of `foo`, we need to write:

```cpp
foo(std::move(s));
```

or:

```cpp
foo(static_cast<shape&&>(s));
```

But if both overloads of `bar` do almost the same thing, except for forwarding, why even have two versions?



Many standard library functions don’t even know the target parameter type but still need to preserve whether the argument was an lvalue or rvalue.

C++ provides `std::forward` for this purpose. Like `std::move`, it leverages reference collapsing, but maintains the value category.

We can simplify the two `bar` functions into one:

```cpp
template <typename T>
void bar(T&& s) {
  foo(std::forward<T>(s));
}
```

Now, for the following calls:

```cpp
circle temp;
bar(temp);       // lvalue
bar(circle());   // rvalue
```

The output is:

```
foo(const shape&)
foo(shape&&)
```

Exactly as desired.

When `T` is a template parameter, `T&&` in this context is called a **forwarding reference** (formerly known as **universal reference**) because it can preserve both lvalues and rvalues.

---








