---
title: "Go Error Handling: Modern Patterns & Best Practices"

description: "Go error handling with modern Go 1.13+ patterns, error wrapping, sentinel errors, and techniques to avoid 'if err != nil' hell."

summary: "Go error handling patterns including modern error wrapping, sentinel errors, resource cleanup with defer, and techniques to avoid 'if err != nil' hell."

date: 2025-06-29
series: ["Golang"]
weight: 1
tags: ["Golang", "Error Handling", "Go 1.13", "Error Wrapping", "Best Practices"]
author: ["Garry Chen"]
cover:
  image: images/go-02-00.jpg
  hiddenInList: true
  caption: "Error"

---

**Error Handling has always been a problem that programming must face. If error handling is done well, the code’s stability will be very good. Different languages have different ways to handle errors. Go language is the same. In this article, we will discuss Go’s error entry and exit points, especially the maddening `if err != nil`.**

Before formally discussing how to deal with Go code flooded with `if err != nil`, I want to first talk about error handling in programming. This allows everyone to understand error handling at a higher level.



# C's error checking

First, we know the most direct way to handle errors is through error codes. This is also the traditional way—the common approach in procedural languages. For example, in C, basically, functions indicate error or not via return values, and then use the global `errno` variable along with an `errstr` array to tell you *why* the error occurred.

Why this design? The reasoning is simple—not only can some errors be shared, but more importantly, it’s a compromise. For example: `read()`, `write()`, `open()` — their return values actually return business-logic values. That is, the return value has two meanings: one is the successful value, such as `open()` returning a file handle pointer `FILE *`, and the other is an error `NULL`. This means the caller doesn’t know why it failed and must check `errno` to get the error cause in order to handle it properly.

In general, this error-handling method works fine in most cases. But there are exceptional cases. Let’s look at this C function:

```c
int atoi(const char *str)
```

This function converts a string to an integer. But the issue arises: if the input string is illegal (not numeric), e.g., `"ABC"`, or the integer overflows, what should it return? If it returns an error, **any** number it returns could conflict with normal results. For example, returning `0` conflicts with converting the string `"0"`. That makes it impossible to distinguish error from valid result. You might say: shouldn’t we check `errno`? In principle yes, but according to the C99 standard we find this description:

> **7.20.1** The functions `atof`, `atoi`, `atol`, and `atoll` need not affect the value of the integer expression `errno` on an error. If the value of the result cannot be represented, the behavior is undefined.

Functions like `atoi()`, `atof()`, `atol()`, or `atoll()` do *not* set `errno`, and moreover, if the result can’t be represented, the behavior is undefined. Later, libc introduced a new function `strtol()` that *does* set the global variable `errno` on error:

```c
long val = strtol(in_str, &endptr, 10);  // 10 means decimal
// if cannot convert
if (endptr == str) {
    fprintf(stderr, "No digits were found\n");
    exit(EXIT_FAILURE);
}
// if integer overflow
if (errno == ERANGE && (val == LONG_MAX || val == LONG_MIN)) {
    fprintf(stderr, "ERROR: number out of range for LONG\n");
    exit(EXIT_FAILURE);
}
// if other error
if (errno != 0 && val == 0) {
    perror("strtol");
    exit(EXIT_FAILURE);
}
```

Although `strtol()` solves the `atoi()` problem, it still feels clumsy and unnatural.

Why? Because this “return value + errno” pattern has problems:

1. Programmers can easily forget to check the return value, introducing bugs.
2. Function interfaces become impure—normal values and error codes are mixed, muddying semantics.

So libraries started distinguishing them. For example, Windows system calls use `HRESULT`, a unified error return type—making clear whether the function call succeeded or failed. But that forces function *inputs* and *outputs* to go through parameters, thus introducing the notion of input parameters vs. output parameters.

Yet this complicates parameter semantics—some are inputs, some outputs—and it still doesn’t prevent ignoring success or failure results.



# Java's error handling

Java uses `try-catch-finally` exception handling, which is a big step up from C. Throwing and catching exceptions brings these benefits:

* Function signatures clearly separate *input* (params), *output* (return value), *error* semantics.
* Normal logic is separated from error handling and resource cleanup, improving readability.
* Exceptions cannot be ignored (explicit catch required to ignore).
* In OOP languages like Java, exceptions are objects, so you can catch them polymorphically.
* Compared to status return codes, exceptions let nested or chained calls, e.g.:

  ```java
  int x = add(a, div(b, c));
  Pizza p = new PizzaBuilder().setSize(sz).setPrice(p)...;
  ```



# Go’s error handling

Go supports multiple return values, so you can separate *business* results and *control* on errors. Many Go functions return `(result, err)`, therefore:

* Parameters are clearly inputs, and returning distinct results and errors clarifies function semantics.
* If you want to ignore the error, you must explicitly do so, e.g., `_`.
* Since `error` is an interface with only `Error() string`, you can define custom error types.
* If a function can return multiple error types, you can switch on type (traditional approach):

  ```go
  if err != nil {
    switch err.(type) {
    case *json.SyntaxError:
      ...
    case *ZeroDivisionError:
      ...
    case *NullPointerError:
      ...
    default:
      ...
    }
  }
  ```

* **Modern approach (Go 1.13+)** uses `errors.Is()` and `errors.As()` for better error checking:

  ```go
  import "errors"
  
  // Check for specific error values
  if errors.Is(err, io.EOF) {
      // handle EOF
  }
  
  // Check for specific error types
  var syntaxErr *json.SyntaxError

  if errors.As(err, &syntaxErr) {
      // handle json.SyntaxError
  }
  ```

In essence, Go’s error handling is return-value checking—but it incorporates many benefits of exceptions, like extensibility through types.

# Resource Cleanup

After an error occurs, you need to clean up resources. Different programming languages use different patterns:

* **C** – Uses `goto fail;` style cleanup in a central location.
* **C++** – Commonly uses the RAII pattern: wrap resources in object proxies, clean in the destructor.
* **Java** – Uses `finally` blocks.
* **Go** – Uses the `defer` keyword.

Here’s an example of resource cleanup in Go:

```go
func Close(c io.Closer) {
    err := c.Close()
    if err != nil {
        log.Fatal(err)
    }
}

func main() {
    r, err := Open("a")
    if err != nil {
        log.Fatalf("error opening 'a'\n")
    }
    defer Close(r) // Use defer to close the file when the function returns.

    r, err = Open("b")
    if err != nil {
        log.Fatalf("error opening 'b'\n")
    }
    defer Close(r) // Use defer again
}
```


# Error Check Hell

Yes, the notorious Go pattern `if err != nil` can overwhelm your code—but there are better ways. Observe this maddening example:

```go
func parse(r io.Reader) (*Point, error) {
    var p Point

    if err := binary.Read(r, binary.BigEndian, &p.Longitude); err != nil {
        return nil, err
    }
    if err := binary.Read(r, binary.BigEndian, &p.Latitude); err != nil {
        return nil, err
    }
    if err := binary.Read(r, binary.BigEndian, &p.Distance); err != nil {
        return nil, err
    }
    if err := binary.Read(r, binary.BigEndian, &p.ElevationGain); err != nil {
        return nil, err
    }
    if err := binary.Read(r, binary.BigEndian, &p.ElevationLoss); err != nil {
        return nil, err
    }
}
```

It’s painful.



You can clean it up using a closure:

```go
func parse(r io.Reader) (*Point, error) {
    var p Point
    var err error
    read := func(data interface{}) {
        if err != nil {
            return
        }
        err = binary.Read(r, binary.BigEndian, data)
    }

    read(&p.Longitude)
    read(&p.Latitude)
    read(&p.Distance)
    read(&p.ElevationGain)
    read(&p.ElevationLoss)

    if err != nil {
        return &p, err
    }
    return &p, nil
}
```

This consolidates the repeated error logic, but you now have a local `err` and a closure—still not ideal.


Inspired by `bufio.Scanner`. Go’s standard pattern (seen in `bufio.Scanner`) uses a struct to collect errors and methods:

```go
scanner := bufio.NewScanner(input)
for scanner.Scan() {
    token := scanner.Text()
    // process token
}
if err := scanner.Err(); err != nil {
    // handle the error
}
```

No `if err != nil` in the loop body—check once after. Let’s adapt that pattern:

```go
type Reader struct {
    r   io.Reader
    err error
}

func (r *Reader) read(data interface{}) {
    if r.err == nil {
        r.err = binary.Read(r.r, binary.BigEndian, data)
    }
}

func parse(input io.Reader) (*Point, error) {
    var p Point
    r := Reader{r: input}

    r.read(&p.Longitude)
    r.read(&p.Latitude)
    r.read(&p.Distance)
    r.read(&p.ElevationGain)
    r.read(&p.ElevationLoss)

    if r.err != nil {
        return nil, r.err
    }
    return &p, nil
}
```


With the above technology, our "[Fluent Interface](https://martinfowler.com/bliki/FluentInterface.html)" is easy to handle. As follows:

```go
package main

import (
    "bytes"
    "encoding/binary"
    "fmt"
)

// byte slice is missing one byte (Weight)
var b = []byte{0x48, 0x61, 0x6f, 0x20, 0x43, 0x68, 0x65, 0x6e, 0x00, 0x00, 0x2c}
var r = bytes.NewReader(b)

type Person struct {
    Name   [10]byte
    Age    uint8
    Weight uint8
    err    error
}

func (p *Person) read(data interface{}) {
    if p.err == nil {
        p.err = binary.Read(r, binary.BigEndian, data)
    }
}

func (p *Person) ReadName() *Person {
    p.read(&p.Name)
    return p
}
func (p *Person) ReadAge() *Person {
    p.read(&p.Age)
    return p
}
func (p *Person) ReadWeight() *Person {
    p.read(&p.Weight)
    return p
}
func (p *Person) Print() *Person {
    if p.err == nil {
        fmt.Printf("Name=%s, Age=%d, Weight=%d\n", p.Name, p.Age, p.Weight)
    }
    return p
}

func main() {
    p := Person{}
    p.ReadName().ReadAge().ReadWeight().Print()
    fmt.Println(p.err)  // EOF error
}
```

This "fluent interface" pattern can greatly clean error handling for repeated operations on the same object. But for **multiple business objects**, you'll still need `if err != nil`.



# Wrapping Errors

Don’t just return `err`—wrap it to preserve context:

Use `fmt.Errorf`:

```go
if err != nil {
    return fmt.Errorf("something failed: %v", err)
}
```

Or define a custom wrapper type:

```go
type authorizationError struct {
    operation string
    err       error  // original error
}

func (e *authorizationError) Error() string {
    return fmt.Sprintf("authorization failed during %s: %v", e.operation, e.err)
}
```

Better yet, implement an `Unwrap()` method so callers can unwrap (Go 1.13+ style):

```go
func (e *authorizationError) Unwrap() error {
    return e.err
}
```

## Modern Error Wrapping (Go 1.13+)

**The recommended approach** is using built-in error wrapping:

```go
if err != nil {
    return fmt.Errorf("read failed: %w", err)
}
```

The `%w` verb wraps the error, preserving the original error for unwrapping.

## Modern Error Checking (Go 1.13+)

Instead of type switching, use `errors.Is()` and `errors.As()`:

```go
import "errors"

// Check for specific error values
if errors.Is(err, io.EOF) {
    // handle EOF specifically
}

// Check for specific error types
var syntaxErr *json.SyntaxError

if errors.As(err, &syntaxErr) {
    // handle json.SyntaxError specifically
    fmt.Printf("JSON syntax error at byte offset %d", syntaxErr.Offset)
}

// Check for custom error types
var authErr *authorizationError

if errors.As(err, &authErr) {
    fmt.Printf("authorization failed during %s", authErr.operation)
}
```

## Sentinel Errors Pattern

Define package-level error variables for common errors:

```go
var (
    ErrNotFound = errors.New("not found")
    ErrInvalidInput = errors.New("invalid input")
    ErrTimeout = errors.New("operation timed out")
)

func FindUser(id string) (*User, error) {
    if id == "" {
        return nil, fmt.Errorf("user lookup failed: %w", ErrInvalidInput)
    }
    // ... lookup logic
    if !found {
        return nil, fmt.Errorf("user %s: %w", id, ErrNotFound)
    }
    return user, nil
}

// Usage
user, err := FindUser("123")
if err != nil {
    if errors.Is(err, ErrNotFound) {
        // handle not found case
    } else if errors.Is(err, ErrInvalidInput) {
        // handle invalid input
    } else {
        // handle other errors
    }
}
```

## Third-Party Libraries (Legacy)

Before Go 1.13, the `github.com/pkg/errors` library was popular:

```go
import "github.com/pkg/errors"

if err != nil {
    return errors.Wrap(err, "read failed")
}

switch err := errors.Cause(err).(type) {
case *MyError:
    // handle specifically
default:
    // unknown error
}
```

**Note:** While `github.com/pkg/errors` still works, the built-in error wrapping in Go 1.13+ is now preferred for new projects.



---
