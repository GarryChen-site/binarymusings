---
title: "Go Decorator Pattern Guide: Pipelines & HTTP Middleware"

description: "Learn Go decorator pattern for function wrapping, HTTP middleware, and pipeline processing. Build reusable higher-order functions."

summary: "Master Go's decorator pattern with practical examples: function timing decorators, HTTP middleware chains, and pipeline processing. Learn to build reusable higher-order functions and explore generic decorators for clean, modular code architecture."

date: 2025-07-09
series: ["Golang"]
weight: 1
tags: ["golang", "decorator-pattern", "middleware", "pipelines", "higher-order-functions"]
author: ["Garry Chen"]
cover:
  image: images/go-08-00.webp
  hiddenInList: true
  caption: ""

---



## Simple Example

Let’s start with an example:

```go
package main

import "fmt"

func decorator(f func(s string)) func(s string) {
    return func(s string) {
        fmt.Println("Started")
        f(s)
        fmt.Println("Done")
    }
}

func Hello(s string) {
    fmt.Println(s)
}

func main() {
    decorator(Hello)("Hello, World!")
}
```

As we can see, we used a higher-order function `decorator()`. When calling it, we first pass in the `Hello()` function. It returns an anonymous function that not only runs its own code but also calls the passed-in `Hello()` function.

This approach is similar to Python’s decorators. However, Go does not support Python-style `@decorator` syntactic sugar. So the syntax isn’t quite as elegant. Of course, if you want to make the code more readable, you can do:

```go
hello := decorator(Hello)
hello("Hello")
```

Let’s look at another example that calculates execution time:

```go
package main

import (
    "fmt"
    "reflect"
    "runtime"
    "time"
)

type SumFunc func(int64, int64) int64

func getFunctionName(i interface{}) string {
    return runtime.FuncForPC(reflect.ValueOf(i).Pointer()).Name()
}

func timedSumFunc(f SumFunc) SumFunc {
    return func(start, end int64) int64 {
        defer func(t time.Time) {
            fmt.Printf("--- Time Elapsed (%s): %v ---\n",
                getFunctionName(f), time.Since(t))
        }(time.Now())

        return f(start, end)
    }
}

func Sum1(start, end int64) int64 {
    var sum int64 = 0
    if start > end {
        start, end = end, start
    }
    for i := start; i <= end; i++ {
        sum += i
    }
    return sum
}

func Sum2(start, end int64) int64 {
    if start > end {
        start, end = end, start
    }
    return (end - start + 1) * (end + start) / 2
}

func main() {
    sum1 := timedSumFunc(Sum1)
    sum2 := timedSumFunc(Sum2)

    fmt.Printf("%d, %d\n", sum1(-10000, 10000000), sum2(-10000, 10000000))
}
```

A few notes on the above code:

1. There are two `Sum` functions. `Sum1()` performs a loop-based sum, while `Sum2()` uses a mathematical formula. (Note: `start` and `end` can be negative.)
2. The code uses Go’s reflection to get the function name.
3. The decorator function is `timedSumFunc()`.

Output after running:

```
$ go run time.sum.go
--- Time Elapsed (main.Sum1): 3.557469ms ---
--- Time Elapsed (main.Sum2): 291ns ---
49999954995000, 49999954995000
```


## HTTP Example

Let’s look at an example of handling HTTP requests.

Here’s a simple HTTP server:

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "strings"
)

func WithServerHeader(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithServerHeader()")
        w.Header().Set("Server", "HelloServer v0.0.1")
        h(w, r)
    }
}

func hello(w http.ResponseWriter, r *http.Request) {
    log.Printf("Received Request %s from %s\n", r.URL.Path, r.RemoteAddr)
    fmt.Fprintf(w, "Hello, World! "+r.URL.Path)
}

func main() {
    http.HandleFunc("/v1/hello", WithServerHeader(hello))
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}
```

In this code, we use a decorator pattern. The function `WithServerHeader()` takes in a `http.HandlerFunc` and returns a modified version. This allows us to add a `Server` header to the HTTP response.

We can write more such functions — for example, to set an auth cookie, check for an auth cookie, log debug information, etc.:

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "strings"
)

func WithServerHeader(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithServerHeader()")
        w.Header().Set("Server", "HelloServer v0.0.1")
        h(w, r)
    }
}

func WithAuthCookie(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithAuthCookie()")
        cookie := &http.Cookie{Name: "Auth", Value: "Pass", Path: "/"}
        http.SetCookie(w, cookie)
        h(w, r)
    }
}

func WithBasicAuth(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithBasicAuth()")
        cookie, err := r.Cookie("Auth")
        if err != nil || cookie.Value != "Pass" {
            w.WriteHeader(http.StatusForbidden)
            return
        }
        h(w, r)
    }
}

func WithDebugLog(h http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        log.Println("--->WithDebugLog")
        r.ParseForm()
        log.Println(r.Form)
        log.Println("path", r.URL.Path)
        log.Println("scheme", r.URL.Scheme)
        log.Println(r.Form["url_long"])
        for k, v := range r.Form {
            log.Println("key:", k)
            log.Println("val:", strings.Join(v, ""))
        }
        h(w, r)
    }
}

func hello(w http.ResponseWriter, r *http.Request) {
    log.Printf("Received Request %s from %s\n", r.URL.Path, r.RemoteAddr)
    fmt.Fprintf(w, "Hello, World! "+r.URL.Path)
}

func main() {
    http.HandleFunc("/v1/hello", WithServerHeader(WithAuthCookie(hello)))
    http.HandleFunc("/v2/hello", WithServerHeader(WithBasicAuth(hello)))
    http.HandleFunc("/v3/hello", WithServerHeader(WithBasicAuth(WithDebugLog(hello))))
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}
```


## Pipeline of Multiple Decorators

When using multiple decorators, you end up nesting them layer by layer, which can look messy — especially when there are many decorators. So let’s refactor this.

We start by writing a utility function that loops through and applies each decorator:

```go
type HttpHandlerDecorator func(http.HandlerFunc) http.HandlerFunc

func Handler(h http.HandlerFunc, decors ...HttpHandlerDecorator) http.HandlerFunc {
    for i := range decors {
        d := decors[len(decors)-1-i] // apply in reverse order
        h = d(h)
    }
    return h
}
```

Now, we can use it like this:

```go
http.HandleFunc("/v4/hello", Handler(hello,
                WithServerHeader, WithBasicAuth, WithDebugLog))
```

This code is much more readable — and the pipeline structure becomes clear.


## Generic Decorators

However, there's still a small problem with using the decorator pattern in Go — it's hard to make it *generic*. Just like the execution-time example above, the decorator's code is tightly coupled with the function signature of what it decorates, so it’s not truly reusable. If this can’t be resolved, the decorator pattern in Go becomes somewhat limited in practicality.

Unlike Python or Java — where Python is dynamically typed and Java has a powerful virtual machine — Go is a statically typed language. This means all types must be resolved at compile time, or the code won’t compile.

Here’s an attempt to write a generic decorator using Go’s generics:

```go
func Decorator3[T1, T2, T3, R any](fn func(T1, T2, T3) R) func(T1, T2, T3) R {
	return func(a T1, b T2, c T3) R {
		fmt.Println("before")
		out := fn(a, b, c)
		fmt.Println("after")
		return out
	}
}

func Decorator2[T1, T2, R any](fn func(T1, T2) R) func(T1, T2) R {
	return func(a T1, b T2) R {
		fmt.Println("before")
		out := fn(a, b)
		fmt.Println("after")
		return out
	}
}
```

Let’s see how this works. Suppose we have two functions we want to decorate:

```go
func foo(a, b, c int) int {
    fmt.Printf("%d, %d, %d\n", a, b, c)
    return a + b + c
}

func bar(a, b string) string {
    fmt.Printf("%s, %s\n", a, b)
    return a + b
}
```

Then, we can apply the decorators like this:

```go
decoratedFoo := Decorator3(foo)
result := decoratedFoo(1, 2, 3)
fmt.Println("Result of decorated foo:", result)

decoratedBar := Decorator2(bar)
resultBar := decoratedBar("Hello, ", "World!")
fmt.Println("Result of decorated bar:", resultBar)
```

This approach makes it possible to write reusable decorators for functions with specific arity (number of arguments), but you’ll still need to write a separate decorator for each function signature — which can’t be avoided due to Go’s static typing constraints.

---

