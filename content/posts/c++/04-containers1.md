---
title: "C++ STL Containers Guide: Vector, List, Deque Performance"

description: "C++ STL containers - vector, list, deque, queue, stack. Memory layout, performance, when to use each container with practical examples."

summary: "Complete guide to C++ STL containers: vector, list, deque, queue, stack with memory layouts, performance tips, and usage examples."

date: 2025-06-20
series: ["C++"]
weight: 1
tags: ["C++", "STL Containers", "Vector", "Performance", "Data Structures"]
author: ["Garry Chen"]
cover:
  image: images/cpp-04-00.jpg
  hiddenInList: true
  caption: "Containers"

---



There are already plenty of learning materials about containers. For reference, cppreference offers very comprehensive documentation. Today, we’ll take a more informal approach. Rather than repeating existing reference material, we’ll try to deepen your understanding of some key containers.


## string

Strictly speaking, `string` is not usually classified as a container in C++. But since it shares many similarities with containers, we’ll start with `string`.

`string` is a specialization of the template `basic_string` for the `char` type. You can think of it as a container that stores `char` elements. The main difference between `string` and "true" container classes is that containers can store objects of any type.

Like most containers, `string` provides the following member functions:

* `begin()` — returns an iterator to the start
* `end()` — returns an iterator to the end
* `empty()` — checks whether it’s empty
* `size()` — returns the size
* `swap()` — swaps contents with another container

*(For those unfamiliar with C++ containers: C++ uses half-open intervals for `begin` and `end`: when non-empty, `begin` points to the first element and `end` points to one past the last element; if empty, `begin == end`. In the case of `string`, for compatibility with C strings, `end` refers to the position after the terminating `\0` character.)*

These features are shared by almost all containers:

* Containers have a beginning and an end
* Containers can check if they're empty
* Containers track their size
* Containers support swapping

Of course, these are just common features; each container serves a different purpose.

Memory Layout of `string`:

![string](images/cpp-04-01.png)

The memory layout of `string` is very similar to `vector`, as you'll see later. In both structure and functionality, `string` and `vector` are closely related.

The main purpose of `string` is to store strings. Unlike C-style strings:

* `string` automatically manages memory and lifecycle.
* `string` supports concatenation (`+` and `+=` operators).
* `string` supports searching (`find`, `rfind`).
* `string` allows safe reading from `istream` (using `getline`).
* `string` allows passing data to functions expecting `const char*` (via `c_str()`).
* `string` supports conversion to and from numbers (`stoi` series, `to_string()`).

Whenever possible, you should use `string` for string management in your code. However, when designing interfaces, things get a bit more complicated. Generally, I don't recommend using `const string&` in function parameters unless you're sure the caller already has a `string`. If the function does little processing, using `const char*` avoids unnecessary construction and destruction of `string` objects when the caller only has a C string, which can be costly.

If your function implementation needs to use `string` functions, consider:

* Use `const string&` or `string_view` (C++17) for read-only access. `string_view` is ideal because even C strings won’t cause extra memory copies.
* Use `string` (by value) if you need to modify the string internally without affecting the caller.
* Use `string&` if you intend to modify the caller's string (usually not recommended).

You’re likely already familiar with `string`. Here’s a simple example:

```cpp
string name;
cout << "What's your name? ";
getline(cin, name);
cout << "Nice to meet you, " << name << "!\n";
```


## vector

`vector` is probably the most widely used container. Though its name comes from mathematics, it’s more accurate to think of it as a dynamic array. It's roughly equivalent to Java's `ArrayList` or Python's `list`.

Just like `string`, elements in a `vector` are stored in contiguous memory, and functions like `begin()`, `end()`, `front()`, and `back()` behave similarly. The memory layout is very similar to `string`:

![vector](images/cpp-04-02.png)

Besides the general container features, `vector` supports the following operations (partial list):

* Element access via subscript `[]` (like `string`)
* `data()` returns a raw pointer to the data (like `string`)
* `capacity()` returns the size of allocated memory (in elements) (like `string`)
* `reserve()` adjusts allocated capacity; after success, `capacity()` changes (like `string`)
* `resize()` changes the actual size; after success, `size()` changes (like `string`)
* `pop_back()` removes the last element (like `string`)
* `push_back()` adds an element at the end (like `string`)
* `insert()` inserts an element before a specified position (like `string`)
* `erase()` removes an element at a specified position (like `string`)
* `emplace()` constructs an element directly at a specified position
* `emplace_back()` constructs an element directly at the end

Pay attention to functions like `push_...` and `pop_...`. When these exist, it usually means that insertions and deletions at those positions are efficient. For `vector`, inserting or deleting at the end is efficient due to its contiguous memory layout; other positions require shifting elements unless memory reallocation occurs.


When functions like `push_back`, `insert`, `reserve`, or `resize` cause memory reallocation, or when `insert` or `erase` move elements, `vector` tries to move elements to the new memory region. `vector` typically guarantees strong exception safety. If your element type does not provide a noexcept move constructor, `vector` will fall back to copying instead.

Therefore, for custom types that are expensive to copy, you should define a move constructor and mark it `noexcept`, or store smart pointers in the container. That’s why we earlier marked move constructors as `noexcept` in our `smart_ptr` example.

Here’s a demonstration:

```cpp
#include <iostream>
#include <vector>

using namespace std;

class Obj1 {
public:
  Obj1() { cout << "Obj1()\n"; }
  Obj1(const Obj1&) { cout << "Obj1(const Obj1&)\n"; }
  Obj1(Obj1&&) { cout << "Obj1(Obj1&&)\n"; }
};

class Obj2 {
public:
  Obj2() { cout << "Obj2()\n"; }
  Obj2(const Obj2&) { cout << "Obj2(const Obj2&)\n"; }
  Obj2(Obj2&&) noexcept { cout << "Obj2(Obj2&&)\n"; }
};

int main() {
  vector<Obj1> v1;
  v1.reserve(2);
  v1.emplace_back();
  v1.emplace_back();
  v1.emplace_back();

  vector<Obj2> v2;
  v2.reserve(2);
  v2.emplace_back();
  v2.emplace_back();
  v2.emplace_back();
}
```

Sample output:

```
Obj1()
Obj1()
Obj1()
Obj1(const Obj1&)
Obj1(const Obj1&)
Obj2()
Obj2()
Obj2()
Obj2(Obj2&&)
Obj2(Obj2&&)
```

Notice that the only difference between `Obj1` and `Obj2` is the `noexcept` on the move constructor — yet this small difference changes whether `vector` chooses to move or copy objects. This is very important.


The `emplace...` family of functions was introduced in C++11 to improve container performance. Try changing `v1.emplace_back()` to `v1.push_back(Obj1())`. The result will be the same, but `push_back` creates an extra temporary object, resulting in an extra (move or copy) constructor call and destructor call. If moving is cheap, the difference is small; if only copying is allowed, the performance difference can be significant.

Modern CPU architectures heavily favor contiguous memory access. `vector`'s contiguous layout is a major performance advantage. When you're unsure which container to use, default to `vector`.

The main downside of `vector` is that resizing may require relocating many elements. If you know in advance that the container will grow large, call `reserve()` early — this can bring significant performance gains.


## deque

`deque` stands for **double-ended queue**. Its main purpose is to support:

> Adding and removing elements freely from both the front and the back of the container.

Compared to `vector`, `deque` has the following differences:

* `deque` provides `push_front`, `emplace_front`, and `pop_front` member functions.
* `deque` does not provide `data`, `capacity`, or `reserve` functions.

Memory Layout:

![deque](images/cpp-04-03.png)

* If you only insert or remove elements at the front or back, objects inside the `deque` never need to be moved.
* Elements are partially contiguous (which is why there’s no `data()` function).
* Since most of the storage is still contiguous, iteration performance remains quite good.
* Storage is split into equal-sized chunks. Subscript access is efficient, roughly implemented as:
  `index[i / chunk_size][i % chunk_size]`.

If you need a container that frequently inserts and removes elements from both ends, `deque` is a good choice.


## list

In C++, `list` represents a **doubly linked list**. Compared to `vector`, it’s optimized for inserting and deleting elements in the middle:

* `list` provides efficient O(1) insertions and deletions at any position.
* `list` does not support random access via subscript.
* Like `deque`, `list` provides `push_front`, `emplace_front`, and `pop_front`.
* Like `deque`, it does not have `data`, `capacity`, or `reserve`.

Memory Layout:

![list](images/cpp-04-04.png)

Although `list` offers flexibility to insert elements at any position, since each element is allocated separately in non-contiguous memory, its iteration performance is worse than both `vector` and `deque`. This significantly offsets the theoretical advantage of not having to move elements during insertion or deletion.

If you don't need to iterate much but often insert or remove elements in the middle, `list` can be a good choice.

Another thing to note: since certain standard algorithms don’t work with `list`, it offers its own member functions as replacements, including:

* `merge`
* `remove`
* `remove_if`
* `reverse`
* `sort`
* `unique`

Example:

```cpp
#include "output_container.h"
#include <iostream>
#include <algorithm>
#include <list>
#include <vector>
using namespace std;

int main()
{
  list<int> lst{1, 7, 2, 8, 3};
  vector<int> vec{1, 7, 2, 8, 3};

  sort(vec.begin(), vec.end());    // works fine
  // sort(lst.begin(), lst.end()); // error
  lst.sort();                      // works fine

  cout << lst << endl;
  // Output: { 1, 2, 3, 7, 8 }

  cout << vec << endl;
  // Output: { 1, 2, 3, 7, 8 }
}
```


## forward_list

Since `list` is a doubly linked list, is there a singly linked list in C++? The answer is yes: starting from C++11, `forward_list` became part of the standard library.

Memory Layout:

![forward_list](images/cpp-04-05.png)

Most C++ containers support an `insert` function to insert an element before a specified position. For `forward_list`, this is hard to achieve (think about why), so the standard library instead provides `insert_after`.

Compared to `list`, `forward_list` lacks:

* `back`
* `size`
* `push_back`
* `emplace_back`
* `pop_back`

Why would we want this more limited version of `list`? Because for small elements, `forward_list` can save a significant amount of memory; and for short lists, not being able to traverse backwards usually isn’t a big problem. Better memory utilization often leads to better performance, especially when memory is tight.

For now, it’s enough for you to simply know that `forward_list` exists. If you don’t feel a need for it, you probably don’t need it.



## queue

Before wrapping up, let’s briefly introduce two more container-like types. These are not full containers but rather **container adaptors** — they wrap existing containers.

First up is `queue`, a **first-in, first-out (FIFO)** data structure.

By default, `queue` is implemented on top of `deque`. Compared to `deque`, it has the following changes:

* No random access via subscript.
* No `begin()` or `end()`.
* `emplace` replaces `emplace_back`; `push` replaces `push_back`; `pop` replaces `pop_front`.
* No other `push_...`, `pop_...`, `emplace...`, `insert`, or `erase` functions.

Its actual memory layout depends on the underlying container. Conceptually, its structure looks like this:

![queue](images/cpp-04-06.png)

Since `queue` doesn’t offer iterators, we cannot directly iterate through it. Instead, we typically write:

```cpp
#include <iostream>
#include <queue>

int main()
{
  std::queue<int> q;
  q.push(1);
  q.push(2);
  q.push(3);
  while (!q.empty()) {
    std::cout << q.front() << std::endl;
    q.pop();
  }
}
```



## stack

Similarly, `stack` is a **last-in, first-out (LIFO)** data structure.

By default, `stack` is also implemented using `deque`, but conceptually it's more similar to `vector`. Compared to `vector`, its differences are:

* No random access via subscript.
* No `begin()` or `end()`.
* `back` becomes `top`; no `front`.
* `emplace` replaces `emplace_back`; `push` replaces `push_back`; `pop` replaces `pop_back`.
* No other `push_...`, `pop_...`, `emplace...`, `insert`, or `erase` functions.

Usually, `stack` is visualized like a vertical `vector`:

![stack](images/cpp-04-07.png)

A small detail to be aware of: the direction of growth for `stack` differs from the system memory stack we discussed earlier. In `stack`, the bottom is the low address and grows upward; in memory management, it’s typically the reverse (high address at the bottom, grows downward). This distinction usually doesn’t matter when using `stack`, but it helps avoid confusion if you ever need to inspect memory structures.

Example code (similar to `queue`, but reversed output):

```cpp
#include <iostream>
#include <stack>

int main()
{
  std::stack<int> s;
  s.push(1);
  s.push(2);
  s.push(3);
  while (!s.empty()) {
    std::cout << s.top() << std::endl;
    s.pop();
  }
}
```

---
