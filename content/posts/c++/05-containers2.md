---
title: "C++ Associative Containers Guide: Maps, Sets & Hash Tables"

description: "C++ associative containers - map, set, unordered_map, hash tables, function objects, and performance optimization with practical examples."

summary: "Complete guide to C++ associative containers: map, set, unordered containers, hash tables, function objects, and performance tips."

date: 2025-06-22
series: ["C++"]
weight: 1
tags: ["C++", "Associative Containers", "Hash Tables", "STL Maps", "Performance"]
author: ["Garry Chen"]
cover:
  image: images/cpp-05-00.jpg
  hiddenInList: true
  caption: "Associative"

---



## Function Objects and Their Specializations

Before discussing containers, we need to first introduce two important function objects: `less` and `hash`.


Let's start with `less`, which implements the *less-than* relation. In the standard library, a general version of `less` is defined roughly like this:

```cpp
template <class T>
struct less : binary_function<T, T, bool> {
  bool operator()(const T& x, const T& y) const {
    return x < y;
  }
};
```

In other words, `less` is a function object — a binary function that compares two values of type `T` and returns a boolean. As a function object, it defines the call operator `operator()`, which by default compares objects using `<`.

Admittedly, this implementation is quite straightforward. The reason is that it's sufficient for most situations. In cases where ordering is needed, C++ typically defaults to using `less`, including in many containers and algorithms such as `sort`. If we need the opposite ordering, we can use `greater`.


The `hash` function object, however, is a bit different. Its purpose is to map a value of some type to an unsigned integer hash value (of type `size_t`). There is no general default implementation. Instead, the system provides specializations for common types, like this:

```cpp
template <class T> struct hash;

template <>
struct hash<int> : public unary_function<int, size_t> {
  size_t operator()(int v) const noexcept {
    return static_cast<size_t>(v);
  }
};
```

This is a very simple example. More complex types, like pointers or strings, have more complex specializations. The idea is that for each type, the type’s author can provide a specialized hash function to ensure that different object values produce well-distributed hash values.

Let’s look at an example to better understand:

```cpp
#include <algorithm>   // std::sort
#include <functional>  // std::less/greater/hash
#include <iostream>    // std::cout/endl
#include <string>      // std::string
#include <vector>      // std::vector
#include "output_container.h"

using namespace std;

int main() {
  vector<int> v{13, 6, 4, 11, 29};
  cout << v << endl;

  sort(v.begin(), v.end());
  cout << v << endl;

  sort(v.begin(), v.end(), greater<int>());
  cout << v << endl;

  cout << hex;

  auto hp = hash<int*>();
  cout << "hash(nullptr)  = " << hp(nullptr) << endl;
  cout << "hash(v.data()) = " << hp(v.data()) << endl;
  cout << "v.data()       = " << static_cast<void*>(v.data()) << endl;

  auto hs = hash<string>();
  cout << "hash(\"hello\")  = " << hs(string("hello")) << endl;
  cout << "hash(\"hellp\")  = " << hs(string("hellp")) << endl;
}
```

A sample output might be:

```
{ 13, 6, 4, 11, 29 }
{ 4, 6, 11, 13, 29 }
{ 29, 13, 11, 6, 4 }
hash(nullptr)  = d7c06285b9de677a
hash(v.data()) = ce2dd0aa15a5e30d
v.data()       = 0x130606140
hash("hello")  = 34432ce1c0308a8
hash("hellp")  = 132303b77e63f8c7
```

As you can see, in this implementation, even `nullptr` has a non-zero hash value, and the hash of a pointer is different from the pointer’s raw address. Different implementations may handle this differently.

In this example, notice that the actual *value* of the function object (`less`, `greater`, `hash`) is irrelevant; their *type* determines behavior. For example, with `sort`, the type of the third parameter determines how sorting is done.

The same principle applies to containers: the function object type controls container behavior.


## priority_queue

`priority_queue` is also a container adapter. We didn’t cover it earlier with the other adapters because it uses a comparison function object (default is `less`).

Like `stack`, it only supports limited operations such as `push`, `pop`, and `top`. But the internal ordering is neither LIFO nor FIFO — instead, it's *partially sorted*. Using the default `less`, the largest value appears at the top. To get the smallest value on top, you can pass `greater` as the comparator.

Example:

```cpp
#include <functional>  // std::greater
#include <iostream>    // std::cout/endl
#include <memory>      // std::pair
#include <queue>       // std::priority_queue
#include <vector>      // std::vector
#include "output_container.h"

using namespace std;

int main() {
  priority_queue<
    pair<int, int>,
    vector<pair<int, int>>,
    greater<pair<int, int>>> q;

  q.push({1, 1});
  q.push({2, 2});
  q.push({0, 3});
  q.push({9, 4});

  while (!q.empty()) {
    cout << q.top() << endl;
    q.pop();
  }
}
```

Output:

```
(0, 3)
(1, 1)
(2, 2)
(9, 4)
```


## Associative Containers

Associative containers include `set`, `map`, `multiset`, and `multimap`. Outside of C++, `map` is often called a *dictionary* or *associative array*, and in JSON it's simply called an *object*. In other languages, these containers are often unordered; in C++, associative containers are ordered by default.

Example:

```cpp
#include <functional>
#include <iostream>
#include <map>
#include <set>
#include <string>
#include <tuple>
#include "output_container.h"

using namespace std;

int main()
{
    set<int> s{1, 1, 1, 2, 3, 4};
    cout << s << endl;

    multiset<int, greater<int>> ms{1, 1, 1, 2, 3, 4};
    cout << ms << endl;

    map<string, int> m{{"one", 1}, {"two", 2}, {"three", 3}, {"four", 4}};
    cout << m << endl;
    m.insert({"four", 4});
    cout << m << endl;
    cout << "mp.find(\"four\") == mp.end(): "
         << (m.find("four") == m.end() ? "true" : "false") << endl;
     cout << "mp.find(\"five\") == mp.end(): "
         << (m.find("five") == m.end() ? "true" : "false") << endl;
     m["five"] = 5;
     cout << m << endl;

     multimap<string, int> mmp{{"one", 1}, {"two", 2}, {"three", 3}, {"four", 4}};
     cout << mmp << endl;
     mmp.insert({"four", -4});
     cout << mmp << endl;

     cout << "m.find(\"four\")->second: "
          << m.find("four")->second << endl;
     cout << "m.lower_bound(\"four\")->second: "
          << m.lower_bound("four")->second << endl;
     cout << "(--m.upper_bound(\"four\"))->second: "
          << (--m.upper_bound("four"))->second << endl;
     
     cout << "mmp.lower_bound(\"four\")->second: "
          << mmp.lower_bound("four")->second << endl;
     cout << "(--mmp.upper_bound(\"four\"))->second: "
          << (--mmp.upper_bound("four"))->second << endl;
          
     multimap<string, int>::iterator lower, upper;
     std::tie(lower, upper) = mmp.equal_range("four");
     cout <<"lower != upper: "
          << (lower != upper ? "true" : "false") << endl;
     cout << "lower->second: " << lower->second << endl;
     cout <<"(--upper)->second: "
          << (--upper)->second << endl;

}
```

Output:

```
{ 1, 2, 3, 4 }
{ 4, 3, 2, 1, 1, 1 }
{ four => 4, one => 1, three => 3, two => 2 }
{ four => 4, one => 1, three => 3, two => 2 }
mp.find("four") == mp.end(): false
mp.find("five") == mp.end(): true
{ five => 5, four => 4, one => 1, three => 3, two => 2 }
{ four => 4, one => 1, three => 3, two => 2 }
{ four => 4, four => -4, one => 1, three => 3, two => 2 }
m.find("four")->second: 4
m.lower_bound("four")->second: 4
(--m.upper_bound("four"))->second: 4
mmp.lower_bound("four")->second: 4
(--mmp.upper_bound("four"))->second: -4
lower != upper: true
lower->second: 4
(--upper)->second: -4
```

Containers with "multi" allow duplicate keys. `set` and `multiset` store keys only. `map` and `multimap` store key-value pairs.

Unlike sequence containers, associative containers do not have front or back operations, but they do support functions like `insert` and `emplace`. They also offer search-related functions like `find`, `lower_bound`, and `upper_bound`:

* `find(k)`: Finds an element equal to `k`.
  (Satisfies: `!(x < k || k < x)` )
* `lower_bound(k)`: Finds the first element not less than `k`.
  (Satisfies: `!(x < k)`)
* `upper_bound(k)`: Finds the first element greater than `k`.
  (Satisfies: `(k < x)`)

If you want to locate the full range of elements matching a key in a `multimap`, it's best to use `equal_range`, which returns both bounds:

```cpp
#include <tuple>

multimap<string, int>::iterator lower, upper;

std::tie(lower, upper) = mmp.equal_range("four");
```

If you don't provide a comparator when declaring an associative container, `less` is used by default. If your key type overloads the `<` operator, you don’t need to do anything extra. Otherwise, you need to specialize `less` or provide your own comparison function object.

For custom types, it’s usually recommended to overload the standard comparison operators (`<`, `==`, etc.) and rely on `less`. Keys stored in associative containers should meet the requirements of *strict weak ordering*, meaning:

* **Irreflexive:** For any `x`, `!(x < x)`
* **Asymmetric:** If `x < y`, then `!(y < x)`
* **Transitive:** If `x < y` and `y < z`, then `x < z`
* **Transitive incomparability:** If `x` and `y` are incomparable, and `y` and `z` are incomparable, then `x` and `z` should also be incomparable.

Usually, types naturally satisfy these conditions. But:

* If your type doesn't have a natural ordering (e.g., complex numbers), do you really need to force one?
* Comparison-based lookup, insertion, and deletion have logarithmic complexity `O(log(n))`. Is there a faster alternative?


## Unordered Associative Containers

Starting from C++11, each associative container has a corresponding unordered version:

* `unordered_set`
* `unordered_map`
* `unordered_multiset`
* `unordered_multimap`

These containers are very similar to their ordered counterparts, with the key difference being that they are *unordered*. Instead of requiring a comparison function object for sorting, they require a function object capable of computing a hash value. You can manually provide such a function object when declaring the container, but most often we use the standard `hash` function object and its specializations.

Here’s an example:

```cpp
#include <complex>        // std::complex
#include <iostream>       // std::cout/endl
#include <unordered_map>  // std::unordered_map
#include <unordered_set>  // std::unordered_set
#include "output_container.h"

using namespace std;

template <typename T>
struct complex_hash {
  size_t operator()(const complex<T>& v) const noexcept {
    hash<T> h;
    return h(v.real()) + h(v.imag());
  }
};

int main() {
  unordered_set<int> s{1, 1, 2, 3, 5, 8, 13, 21};
  cout << s << endl;

  unordered_map<complex<double>, double, complex_hash<double>> umc{
    {{1.0, 1.0}, 1.4142},
    {{3.0, 4.0}, 5.0}
  };
  cout << umc << endl;
}
```

The output might be (order not guaranteed):

```
{ 21, 5, 8, 3, 13, 2, 1 }
{ (3,4) => 5, (1,1) => 1.4142 }
```

Note that we added a specialization in the `std` namespace, which is one of the very rare cases where users are allowed to add to the `std` namespace. Normally, adding declarations or definitions to the `std` namespace is prohibited and results in undefined behavior.


From an engineering perspective, the main advantage of unordered associative containers is performance. The insertion, deletion, and lookup operations for ordered associative containers and `priority_queue` have logarithmic complexity `O(log(n))`. Unordered associative containers use hash tables, allowing average-case complexity of `O(1)`!

However, this depends on the quality of the hash function. If the hash function is poorly designed, performance can degrade to `O(n)` — worse than ordered associative containers.



## array

The last container we’ll cover is `array`, which serves as a modern replacement for C-style arrays. C arrays still exist in C++ mainly for backward compatibility, but they differ greatly from C++ containers:

* C arrays do not have `begin` and `end` member functions (though you can use global `begin` and `end` functions).
* C arrays do not have a `size` member function (you need template tricks to obtain their length).
* When passed as function parameters, C arrays decay into pointers, and the callee loses information about their length.

In C, people often defined a macro to get an array’s length:

```c
#define ARRAY_LEN(a) (sizeof(a) / sizeof((a)[0]))
```

C++17 directly provides a `std::size()` function, which works only on real arrays, failing if the array has decayed into a pointer:

```cpp
#include <iostream>  // std::cout/endl
#include <iterator>  // std::size

void test(int arr[]) {
  // Compilation error:
  // std::cout << std::size(arr) << std::endl;
}

int main() {
  int arr[] = {1, 2, 3, 4, 5};
  std::cout << "The array length is " << std::size(arr) << std::endl;
  test(arr);
}
```

C arrays also do not support proper copying behavior. For example, you cannot use a C array as a key in a `map` or `unordered_map`:

```cpp
#include <map>  // std::map

typedef char mykey_t[8];

int main() {
  std::map<mykey_t, int> mp;
  mykey_t mykey{"hello"};
  mp[mykey] = 5;  // Large compilation error
}
```

What should we use instead of C arrays?

We have several options:

* If the array is large, use `vector`, which offers the most flexibility and good performance.
* For character arrays, use `string`.
* If the array has a fixed size (as C arrays do) and is small, use `array`. It preserves the stack allocation behavior of C arrays, but also provides `begin`, `end`, `size`, and other standard member functions.

Using `array` avoids many of the quirks of C arrays. The failing example above can be easily fixed by replacing the C array with `std::array`:

```cpp
#include <array>     // std::array
#include <iostream>  // std::cout/endl
#include <map>       // std::map
#include "output_container.h"

typedef std::array<char, 8> mykey_t;

int main() {
  std::map<mykey_t, int> mp;
  mykey_t mykey{"hello"};
  mp[mykey] = 5;  // OK
  std::cout << mp << std::endl;
}
```

The output is as expected:

```
{ hello => 5 }
```

---