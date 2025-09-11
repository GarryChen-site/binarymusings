---
title: "Go Programming Patterns: Slices, Interfaces & Performance"

description: "Go slice memory management, interface design patterns, and performance optimization. Learn best practices with examples and gotchas."

summary: "Essential Go programming patterns including slice memory management, interface design, time handling, and performance optimization techniques for better Go applications."

date: 2025-06-28

series: ["Golang"]
weight: 1
tags: ["Golang", "Programming Patterns", "Slice Management", "Interface Design", "Performance"]
author: ["Garry Chen"]
cover:
  image: images/go-01-00.jpg
  hiddenInList: true
  caption: "Slice"

---


## Slice

First, let’s discuss **Slice**. In Go, a slice is not an array but a struct, defined as follows:

```go
type slice struct {
    array unsafe.Pointer // pointer to the underlying array where data is stored  
    len   int            // how large the length is  
    cap   int            // how large the capacity is  
}
```

Illustrated graphically, an empty slice looks like this:

![empty slice](images/go-01-01.webp)

Anyone familiar with C/C++ knows that using a pointer to an array in a struct leads to shared data! Now let’s look at some operations on a slice:

```go
foo = make([]int, 5)
foo[3] = 42
foo[4] = 100

bar := foo[1:4]
bar[1] = 99
```

For the code above:

1. First, create a slice `foo` with length and capacity both equal to 5.
2. Then assign to the elements at indices 3 and 4 of the array pointed to by `foo`.
3. Next, slice `foo` to assign to `bar`, then modify `bar[1]`.

![shared memory](images/go-01-02.webp)

From the diagram, we see that because `foo` and `bar` share memory, modifications via either one affect the other.

Now let’s look at an example using `append()`:

```go
a := make([]int, 32)
b := a[1:16]
a = append(a, 1)
a[2] = 42
```

In this snippet, slicing `a[1:16]` assigns to `b`, so `a` and `b` share memory. Calling `append()` on `a` causes it to reallocate memory, causing `a` and `b` to no longer share, as shown in the following diagram:

![append example](images/go-01-03.webp)

From the diagram, you can see that `append()` increases `a`’s capacity to 64 and its length to 33. It’s important to note: **when `cap` is insufficient, `append()` reallocates to increase capacity; if capacity is sufficient, it does not reallocate!**


Another example:

```go
func main() {
    path := []byte("AAAA/BBBBBBBBB")
    sepIndex := bytes.IndexByte(path, '/')

    dir1 := path[:sepIndex]
    dir2 := path[sepIndex+1:]

    fmt.Println("dir1 =>", string(dir1)) // prints: dir1 => AAAA
    fmt.Println("dir2 =>", string(dir2)) // prints: dir2 => BBBBBBBBB

    dir1 = append(dir1, "suffix"...)

    fmt.Println("dir1 =>", string(dir1)) // prints: dir1 => AAAAsuffix
    fmt.Println("dir2 =>", string(dir2)) // prints: dir2 => uffixBBBB
}
```

In this example, `dir1` and `dir2` share memory. Even though `dir1` is appended, because its capacity is sufficient, it extends into `dir2`’s memory region. The illustration below shows changes in `cap` and `len` for both.

![reallocation example](images/go-01-04.webp)

To fix this, change one line:

```go
dir1 := path[:sepIndex]
```

to:

```go
dir1 := path[:sepIndex:sepIndex]
```

The new code uses the **Full Slice Expression**, where the final parameter is the **Limited Capacity**. Subsequent `append()` operations will then reallocate memory, preventing data overlap.



## Deep Equality Comparison

When comparing objects that may be built-in types, arrays, structs, maps… copying a struct and comparing fields for equality requires *deep comparison*, not just shallow. For that, Go provides reflection with `reflect.DeepEqual()`. Here are some examples:

```go
import (
    "fmt"
    "reflect"
)

func main() {
    v1 := data{}
    v2 := data{}
    fmt.Println("v1 == v2:", reflect.DeepEqual(v1, v2))
    // prints: v1 == v2: true

    m1 := map[string]string{"one": "a", "two": "b"}
    m2 := map[string]string{"two": "b", "one": "a"}
    fmt.Println("m1 == m2:", reflect.DeepEqual(m1, m2))
    // prints: m1 == m2: true

    s1 := []int{1, 2, 3}
    s2 := []int{1, 2, 3}
    fmt.Println("s1 == s2:", reflect.DeepEqual(s1, s2))
    // prints: s1 == s2: true
}
```

## Interface Programming
Below, let’s look at some code containing two methods for printing a struct—one uses a function, the other uses a *method*.

```go
func PrintPerson(p *Person) {
    fmt.Printf("Name=%s, Sexual=%s, Age=%d\n",
        p.Name, p.Sexual, p.Age)
}

func (p *Person) Print() {
    fmt.Printf("Name=%s, Sexual=%s, Age=%d\n",
        p.Name, p.Sexual, p.Age)
}

func main() {
    var p = Person{
        Name:    "John Doe",
        Sexual:  "Male",
        Age:     44,
    }

    PrintPerson(&p)
    p.Print()
}
```

Which style do you prefer? In Go, the “method” style uses a *receiver*. This approach is encapsulation—`PrintPerson()` is tightly coupled with `Person`, so fitting them together makes sense. More importantly, it enables interface-based abstraction, which is essential for *polymorphism*.


Consider the following code:

```go
type Country struct {
    Name string
}

type City struct {
    Name string
}

type Printable interface {
    PrintStr()
}

func (c Country) PrintStr() {
    fmt.Println(c.Name)
}
func (c City) PrintStr() {
    fmt.Println(c.Name)
}

c1 := Country{"USA"}
c2 := City{"Los Angeles"}
c1.PrintStr()
c2.PrintStr()
```

Here, we define a `Printable` interface, and both `Country` and `City` implement the `PrintStr()` method to print their own `Name`. But the code in each is essentially identical. Can we DRY it up?


One way is embedding a struct:

```go
type WithName struct {
    Name string
}

type Country struct {
    WithName
}

type City struct {
    WithName
}

type Printable interface {
    PrintStr()
}

func (w WithName) PrintStr() {
    fmt.Println(w.Name)
}

c1 := Country{WithName{"USA"}}
c2 := City{WithName{"Los Angeles"}}
c1.PrintStr()
c2.PrintStr()
```

We introduced a `WithName` struct and reuse its `PrintStr()` method. But initialization now becomes slightly messy.


A cleaner approach is:

```go
type Country struct {
    Name string
}

type City struct {
    Name string
}

type Stringable interface {
    ToString() string
}

func (c Country) ToString() string {
    return "Country = " + c.Name
}

func (c City) ToString() string {
    return "City = " + c.Name
}

func PrintStr(p Stringable) {
    fmt.Println(p.ToString())
}

d1 := Country{"USA"}
d2 := City{"Los Angeles"}
PrintStr(d1)
PrintStr(d2)
```

Here, we define a `Stringable` interface and a function `PrintStr(p Stringable)`. This decouples the *business types* (`Country`, `City`) and the *control logic* (`PrintStr`). Any type implementing `ToString()` can be printed, following the golden OOP principle: **“Program to an interface, not an implementation.”**

This pattern appears throughout Go’s standard library, e.g., `io.Reader` and `ioutil.ReadAll`. As long as you implement:

```go
Read(p []byte) (n int, err error)
```

you can pass your type to `ioutil.ReadAll`.



## Interface Completeness Check

Go’s compiler doesn’t strictly require that a type implement *all* methods of an interface unless you use it. Here’s an example:

```go
type Shape interface {
    Sides() int
    Area() int
}

type Square struct {
    len int
}

func (s *Square) Sides() int {
    return 4
}

func main() {
    s := Square{len: 5}
    fmt.Printf("%d\n", s.Sides())
}
```

`Square` implements `Sides()` but not `Area()`. The code compiles, though it’s not rigorous. To enforce full implementation, do this:

```go
var _ Shape = (*Square)(nil)
```

This declares an unused var, assigning a `*Square` pointer to a `Shape`. If `Square` fails to implement all `Shape` methods, the compiler errors:

> Square does not implement Shape (missing Area method)


This enforces interface compliance.


## Time Management

Time handling is complex—time zones, formats, precision, etc. Always reuse existing libraries rather than roll your own. In Go, use `time.Time` and `time.Duration`:

* `flag` package supports `time.ParseDuration`
* JSON (`encoding/json`) encodes/decodes `time.Time` in RFC3339 format
* `database/sql` maps DATETIME/TIMESTAMP to `time.Time`
* YAML library (e.g. `gopkg.in/yaml.v2`) supports `time.Time`, `time.Duration`, and RFC3339 format

When exchanging data externally, stick to [RFC3339](https://datatracker.ietf.org/doc/html/rfc3339). For global, cross-timezone applications, store everything in UTC on all servers.


## Performance Tips

Go is high-performance, but you should still care. Here are some practical tips:

* Use `strconv.Itoa()` instead of `fmt.Sprintf()` for integer-to-string—about twice as fast.
* Avoid converting `string` to `[]byte`—this hurts performance.
* Pre-allocate sufficient `capacity` for slices when using `append()` inside loops to avoid reallocation and 2× growth waste.
* Use `strings.Builder` or `bytes.Buffer` instead of `+` or `+=` for building strings: 1000× faster.
* Use goroutines with `sync.WaitGroup` to parallelize slice operations.
* Avoid heap allocations in hot paths—use `sync.Pool` to reuse objects and reduce GC pressure.
* Use lock-free atomic ops (`sync/atomic`) instead of `mutex` where possible.
* Buffer I/O heavily with `bufio.NewWriter()` / `bufio.NewReader()`—I/O is slow.
* Pre-compile regexes with `regexp.Compile()` for loops—gains two orders of magnitude in speed.
* For high-performance serialization, consider protobuf or `msgp` instead of JSON (JSON uses reflection).
* For `map` keys, `int` keys are faster than `string` keys due to cheaper comparisons.


