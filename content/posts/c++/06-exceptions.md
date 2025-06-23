---
title: "Exceptions"
description: "Use it or not, that's a question"
summary: Modern C++ Series - Part 06
date: 2025-06-23
series: ["C++"]
weight: 1
tags: ["Modern C++", "Exceptions", "Error Handling", "RAII", "noexcept"]
author: ["Garry Chen"]
cover:
  image: images/cpp-06-00.jpg
  hiddenInList: true
  caption: "Exceptions"

---


So far, we’ve mentioned exceptions several times. Today, let’s talk about exceptions in depth.


Let’s be clear right from the start: if you’re not sure whether you should use exceptions, the answer is **yes, you should**. If you need to avoid exceptions, you must have a *specific* reason for doing so.

Now let’s dive into the topic of exceptions.


# A World Without Exceptions

First, let’s look at what a world without exceptions looks like. The most typical example is C. Suppose we’re performing some matrix operations and define the following matrix structure:

```c
typedef struct {
  float* data;
  size_t nrows;
  size_t ncols;
} matrix;
```

We need at least some initialization and cleanup code:

```c
enum matrix_err_code {
  MATRIX_SUCCESS,
  MATRIX_ERR_MEMORY_INSUFFICIENT,
  …
};

int matrix_alloc(matrix* ptr, size_t nrows, size_t ncols) {
  size_t size = nrows * ncols * sizeof(float);
  float* data = malloc(size);
  if (data == NULL) {
    return MATRIX_ERR_MEMORY_INSUFFICIENT;
  }
  ptr->data = data;
  ptr->nrows = nrows;
  ptr->ncols = ncols;
  return MATRIX_SUCCESS;
}

void matrix_dealloc(matrix* ptr) {
  if (ptr->data == NULL) {
    return;
  }
  free(ptr->data);
  ptr->data = NULL;
  ptr->nrows = 0;
  ptr->ncols = 0;
}
```

Now let’s write the multiplication function:

```c
int matrix_multiply(matrix* result, const matrix* lhs, const matrix* rhs) {
  int errcode;
  if (lhs->ncols != rhs->nrows) {
    return MATRIX_ERR_MISMATCHED_MATRIX_SIZE;
    // This error code needs to be added to enum matrix_err_code
  }
  errcode = matrix_alloc(result, lhs->nrows, rhs->ncols);
  if (errcode != MATRIX_SUCCESS) {
    return errcode;
  }
  // Perform matrix multiplication
  return MATRIX_SUCCESS;
}
```

Calling this function looks like this:

```c
matrix c;

// Zero initialize to simplify error handling and cleanup
memset(&c, 0, sizeof(matrix));

errcode = matrix_multiply(&c, &a, &b);
if (errcode != MATRIX_SUCCESS) {
  goto error_exit;
}
// Use multiplication result

error_exit:
matrix_dealloc(&c);
return errcode;
```

As you can see, we need to check for errors all over the place.


Can we do better in C++ without exceptions?

Technically yes, but things don’t get much better. Since C++ constructors can’t return error codes, you can’t perform operations that may fail inside constructors. Instead, you’d need a constructor that only zero-initializes members, followed by a separate `init()` function for real initialization.

Even though C++ supports operator overloading, you wouldn’t be able to use it here either, because you can’t return an error code from overloaded operators.

And this is just for a single level of function calls. If an error occurs far away from where it's handled, each layer needs to propagate the error code upward, cluttering your code and making it harder to read and maintain.



# Using Exceptions

With exceptions, you can safely perform initialization directly inside the constructor. Let’s say our matrix class has the following members:

```cpp
class matrix {
  …
private:
  float* data_;
  size_t nrows_;
  size_t ncols_;
};
```

The constructor can look like this:

```cpp
matrix::matrix(size_t nrows, size_t ncols) {
  data_  = new float[nrows * ncols];
  nrows_ = nrows;
  ncols_ = ncols;
}
```

The destructor is simple:

```cpp
matrix::~matrix() {
  delete[] data_;
}
```

Multiplication can be written like this:

```cpp
class matrix {
  …
  friend matrix operator*(const matrix&, const matrix&);
};

matrix operator*(const matrix& lhs, const matrix& rhs) {
  if (lhs.ncols_ != rhs.nrows_) {
    throw std::runtime_error("matrix sizes mismatch");
  }
  matrix result(lhs.nrows_, rhs.ncols_);
  // Perform matrix multiplication
  return result;
}
```

Using the multiplication function is simple too:

```cpp
matrix c = a * b;
```



At this point you might wonder: where’s the error handling? There’s only a single `throw`. Is this really equivalent to the C version?

Yes — and **even better**.

Using exceptions doesn’t mean you must always write `try` and `catch`. Code can still be *exception-safe* even without explicit `try` blocks.


What is "exception safety"? Exception safety means that if an exception occurs:

* No resources are leaked
* The system remains in a consistent state

Let’s analyze where errors might occur:

* **Memory allocation:** If `new` fails, C++ throws `bad_alloc`. Before this exception is caught, all stack-allocated objects are properly destroyed, automatically releasing resources.
* **Invalid matrix size:** If dimensions don’t match, we throw `runtime_error`. No `result` object gets created, and thus no `c` object either.
* **Failure during multiplication:** Same as above — if an allocation fails during multiplication, `result` won’t exist.
* **Failure while handling `a` and `b`:** They are local variables, so destructors will automatically clean them up if an exception occurs.

In short: as long as you organize your code properly and leverage RAII, your code can be shorter, cleaner, and safer. You can handle exceptions globally — often just for logging or user-facing error reporting.


# Downsides of Exceptions

Of course, exceptions aren’t perfect. The two main criticisms are:

1. **Exceptions violate C++’s “you don’t pay for what you don’t use” principle.** Even if you don’t throw exceptions, enabling exception support still increases binary size.
2. **Exceptions can be hard to spot.** It’s not always clear which code may throw and which exceptions might be thrown.

About point 1: There’s little developers can do here. Most C++ implementations sacrifice binary size for better runtime performance on the normal path (the "happy path"). As long as exceptions are not thrown, performance overhead is usually just a few percent — negligible in most applications.

About point 2: This is a more valid concern. Unlike Java, C++ does not perform compile-time checking of which exceptions might be thrown. Since C++17, dynamic exception specifications have been completely removed. The only thing you can declare now is whether a function is `noexcept`.

If a function marked `noexcept` throws an exception, C++ will call `std::terminate` and crash the program.

Because it’s often impossible to predict exceptions in generic code, C++ generally avoids requiring exception declarations.

How to deal with exceptions responsibly:

* **Write exception-safe code, especially inside templates.** Aim for strong exception safety where possible: if any third-party code throws, your objects remain unchanged, and no resources leak.
* **Document which exceptions your code might throw**, so users know what to expect without inspecting your implementation.
* **Mark functions as `noexcept` when they cannot throw.** This is especially important for move constructors, move assignment operators, and `swap()`. Destructors are automatically treated as `noexcept`.


# Why Use Exceptions?

Even though later we’ll discuss other error-handling approaches (without exceptions or error codes), exceptions are deeply integrated into C++ as its standard error-handling mechanism. The standard library itself uses exceptions — not only for runtime errors but even for certain logic errors.

For example, many standard containers provide an `at()` method (in addition to `operator[]`), which throws an exception if you access an invalid index:

```cpp
#include <iostream>   // std::cout/endl
#include <stdexcept>  // std::out_of_range
#include <vector>     // std::vector

using namespace std;

int main() 
{
  vector<int> v{1, 2, 3};

  try {
    v.at(3);
  }
  catch (const out_of_range& e) {
    cerr << e.what() << endl;
  }

}
```

Output:

```
// GCC
_M_range_check: __n (which is 3) >= this->size() (which is 3)
```

Most standard containers provide strong exception safety. For instance, `vector` will use copy constructors instead of move constructors if the move constructor isn’t guaranteed not to throw. If an exception happens mid-move, the element might be in a partially destroyed state, making strong exception safety impossible.

In short: if you’re using standard containers, even if you don’t actively use exceptions yourself, you still need to be prepared for exceptions (at least `bad_alloc`), unless you’re certain your target runtime environment cannot throw them — which might be true for some limited Linux configurations, and that’s partly why Google allows exception-less C++.


For logic errors, you can choose between exceptions and `assert()`:

* `assert()` works well during debugging but is usually disabled in production builds.
* Exceptions work in both debug and production, offering consistent behavior.

Since test coverage can’t always hit all code paths, relying solely on `assert()` is not enough. Exceptions offer a way to handle errors robustly in both environments.

---

