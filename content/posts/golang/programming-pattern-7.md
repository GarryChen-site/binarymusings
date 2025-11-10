---
title: "Go Generics Guide: Type-Safe Programming & Data Structures"

description: "Master Go generics with type-safe programming, generic data structures, and functional patterns. Build reusable code with type constraints."

summary: "Comprehensive guide to Go generics: learn type parameters, constraints, and build generic data structures like stacks and linked lists. Explore functional programming patterns with generic Map, Reduce, and Filter operations for type-safe, reusable code."

date: 2025-07-07
series: ["Golang"]
weight: 1
tags: ["golang", "generics", "type-safety", "data-structures", "functional-programming"]
author: ["Garry Chen"]
cover:
  image: images/go-07-00.jpg
  hiddenInList: true
  caption: "Golang Generics"

---

## First Look

Let’s start with a simple example:

```go
package main

import "fmt"

func print[T any](arr []T) {
  for _, v := range arr {
    fmt.Print(v)
    fmt.Print(" ")
  }
  fmt.Println("")
}

func main() {
  strs := []string{"Hello", "World",  "Generics"}
  decs := []float64{3.14, 1.14, 1.618, 2.718}
  nums := []int{2, 4, 6, 8}

  print(strs)
  print(decs)
  print(nums)
}
```

In this example, we have a `print()` function designed to print elements of an array. Without generics, you would need to write separate versions for `int`, `float`, `string`, and custom `struct` types. But thanks to generics, we can declare a type parameter with `[T any]` (similar to C++'s `typename T`), and use `T` throughout the function.

This generic `print()` function works for `int`, `float64`, and `string`.

With this ability, we can now write standard algorithms, like a search function:

```go
func find[T comparable](arr []T, elem T) int {
  for i, v := range arr {
    if v == elem {
      return i
    }
  }
  return -1
}
```

Notice we used `[T comparable]` instead of `[T any]`. The `comparable` constraint ensures that the type supports the `==` operator; otherwise, the code won’t compile. This `find()` function works with `int`, `float64`, and `string`.

From these two examples, Go’s generics are basically ready to use, though there are a few limitations:

1. `fmt.Printf()` still uses `%v` for all types; Go doesn't support something like C++'s `iostream` operator overloading.
2. Go doesn’t support operator overloading, so you can’t use generic arithmetic or comparison operators like `==` in custom ways.
3. The `find()` algorithm works with arrays, but not with other data structures like hash tables, trees, graphs, or linked lists. There’s no generic iterator abstraction like in C++ STL (which often uses operator overloading like `++`).

Still, this is a great step forward. Let’s see what we can do with it.


## Data Structures

### Stack

One major benefit of generics is creating type-independent data structures. Here’s how we implement a generic stack using a slice.

```go
type stack[T any] []T
```

It’s just a slice of type `T`. Now let’s implement methods like `push()`, `pop()`, `top()`, `len()`, and `print()` — very similar to C++’s STL stack.

```go
func (s *stack[T]) push(elem T) {
  *s = append(*s, elem)
}

func (s *stack[T]) pop() {
  if len(*s) > 0 {
    *s = (*s)[:len(*s)-1]
  } 
}

func (s *stack[T]) top() *T {
  if len(*s) > 0 {
    return &(*s)[len(*s)-1]
  }
  return nil
}

func (s *stack[T]) len() int {
  return len(*s)
}

func (s *stack[T]) print() {
  for _, elem := range *s {
    fmt.Print(elem, " ")
  }
  fmt.Println("")
}
```

The above example is still relatively simple, but during the implementation process, there was a bit of a hang-up when the stack is empty — in that case, `top()` either has to return an error or return a null value. Because before, the "null" value we returned was either `0` for int, or `""` for string. However, under the generic type `T`, that value becomes hard to handle. In other words, beyond type generics, we also need some sort of "value generics" (Note: in C++, if you try to call `top()` on an empty stack, you’ll get a segmentation fault), so here we return a pointer, allowing us to check whether the pointer is nil.

Stack Usage:

```go
func main() {
  ss := stack[string]{}
  ss.push("Hello")
  ss.push("Linus")
  ss.push("Torvalds")
  ss.print()
  fmt.Printf("stack top is - %v\n", *(ss.top()))
  ss.pop()
  ss.pop()
  ss.print()

  ns := stack[int]{}
  ns.push(10)
  ns.push(20)
  ns.print()
  ns.pop()
  ns.print()
  *ns.top() += 1
  ns.print()
  ns.pop()
  fmt.Printf("stack top is - %v\n", ns.top())
}
```


### Doubly Linked List

Here’s a generic implementation of a doubly linked list with methods:

* `add()` – insert at the head
* `push()` – insert at the tail
* `del()` – delete a node (requires `comparable`)
* `print()` – traverse and print from head

```go
type node[T comparable] struct {
  data T
  prev *node[T]
  next *node[T]
}

type list[T comparable] struct {
  head, tail *node[T]
  len int
}

func (l *list[T]) isEmpty() bool {
  return l.head == nil && l.tail == nil
}

func (l *list[T]) add(data T) {
  n := &node[T]{data: data, next: l.head}
  if l.isEmpty() {
    l.head = n
    l.tail = n
  }
  l.head.prev = n
  l.head = n
}

func (l *list[T]) push(data T) {
  n := &node[T]{data: data, prev: l.tail}
  if l.isEmpty() {
    l.head = n
    l.tail = n
  }
  l.tail.next = n
  l.tail = n
}

func (l *list[T]) del(data T) {
  for p := l.head; p != nil; p = p.next {
    if data == p.data {
      if p == l.head {
        l.head = p.next
      }
      if p == l.tail {
        l.tail = p.prev
      }
      if p.prev != nil {
        p.prev.next = p.next
      }
      if p.next != nil {
        p.next.prev = p.prev
      }
      return
    }
  }
}

func (l *list[T]) print() {
  if l.isEmpty() {
    fmt.Println("the link list is empty.")
    return
  }
  for p := l.head; p != nil; p = p.next {
    fmt.Printf("[%v] -> ", p.data)
  }
  fmt.Println("nil")
}
```
 List Usage:

```go
func main(){
  var l = list[int]{}
  l.add(1)
  l.add(2)
  l.push(3)
  l.push(4)
  l.add(5)
  l.print() //[5] -> [2] -> [1] -> [3] -> [4] -> nil
  l.del(5)
  l.del(1)
  l.del(4)
  l.print() //[2] -> [3] -> nil
}
```


## Functional Generics

### Generic Map

```go
func gMap[T1 any, T2 any](arr []T1, f func(T1) T2) []T2 {
  result := make([]T2, len(arr))
  for i, elem := range arr {
    result[i] = f(elem)
  }
  return result
}
```

In the above map function, I used two types – `T1` and `T2`.

* `T1` – is the type of the data to be processed
* `T2` – is the type of the processed data
* `T1` and `T2` can be the same or different.

We also have a function parameter – `func(T1) T2` – which means it takes in a `T1` type and returns a `T2` type.

Then, the entire function returns a `[]T2`.

Usage:

```go
nums := []int{0,1,2,3,4,5,6,7,8,9}
squares := gMap(nums, func(elem int) int {
  return elem * elem
})
print(squares) // [0 1 4 9 16 25 36 49 64 81]

strs := []string{"hello", "world", "golang", "generics"}
upstrs := gMap(strs, func(s string) string {
  return strings.ToUpper(s)
})
print(upstrs) // ["HELLO" "WORLD" "GOLANG" "GENERICS"]

```



### Generic Reduce

```go
func gReduce[T1 any, T2 any](arr []T1, init T2, f func(T2, T1) T2) T2 {
  result := init
  for _, elem := range arr {
    result = f(result, elem)
  }
  return result
}
```

The function is quite simple to implement, but it feels not very elegant.

* It also has two types `T1` and `T2`, the former is the input data type, and the latter is the output data type.
* Because we want to combine data into one, we need an initial value `init`, which is of type `T2`.
* The custom function `func(T2, T1) T2` passes this `init` value to the user, and then the user processes it and returns it back.

Below is a usage example — summing an array.

```go
nums := []int{0,1,2,3,4,5,6,7,8,9}
sum := gReduce(nums, 0, func(result, elem int) int {
  return result + elem
})
fmt.Printf("Sum = %d\n", sum)
```


### Generic Filter

The filter function is mainly used for filtering, to filter out some data that meets certain conditions (filter in) or does not meet the conditions (filter out). Below is a related code example.

```go
func gFilter[T any](arr []T, in bool, f func(T) bool) []T {
  result := []T{}
  for _, elem := range arr {
    choose := f(elem)
    if (in && choose) || (!in && !choose) {
      result = append(result, elem)
    }
  }
  return result
}

func gFilterIn[T any](arr []T, f func(T) bool) []T {
  return gFilter(arr, true, f)
}

func gFilterOut[T any](arr []T, f func(T) bool) []T {
  return gFilter(arr, false, f)
}
```

Here, the user needs to provide a `bool` function. We will pass the data to the user, and the user only needs to tell us yes or no, and then we will return a filtered array back to the user.

For example, if we want to filter out all the odd numbers in the array.

```go
nums := []int{0,1,2,3,4,5,6,7,8,9}
odds := gFilterIn(nums, func(elem int) bool {
  return elem % 2 == 1
})
print(odds)
```


## Business Example

Define an employee struct and data:

```go
type Employee struct {
  Name     string
  Age      int
  Vacation int
  Salary   float32
}

var employees = []Employee{
  {"Hao", 44, 0, 8000.5},
  {"Bob", 34, 10, 5000.5},
  {"Alice", 23, 5, 9000.0},
  {"Jack", 26, 0, 4000.0},
  {"Tom", 48, 9, 7500.75},
  {"Marry", 29, 0, 6000.0},
  {"Mike", 32, 8, 4000.3},
}
```

Then, we want to sum up all employees’ salaries uniformly, and we can use the previous reduce function.

```go
total_pay := gReduce(employees, 0.0, func(result float32, e Employee) float32 {
  return result + e.Salary
})
fmt.Printf("Total Salary: %0.2f\n", total_pay)
```


Our function — this `gReduce` — is a bit verbose. We still need to pass in an initial value, and in the user's own function, they still need to deal with `result`. Let’s define a better version.

Generally speaking, when we use reduce, it's mostly for summing or counting. So, can we define it in a more straightforward way? For example, the `CountIf()` function below is much cleaner than the above Reduce.

```go
func gCountIf[T any](arr []T, f func(T) bool) int {
  cnt := 0
  for _, elem := range arr {
    if f(elem) {
      cnt++
    }
  }
  return cnt
}
```

When we do summing, we can also write a generic Sum.

* It processes data of type `T` and returns a result of type `U`.
* Then, the user only needs to give me a `U`-typed value to be summed, based on the data of type `T`.

The code is as follows:

```go
type Sumable interface {
  type int, int8, int16, int32, int64,
       uint, uint8, uint16, uint32, uint64,
       float32, float64
}

func gSum[T any, U Sumable](arr []T, f func(T) U) U {
  var sum U
  for _, elem := range arr {
    sum += f(elem)
  }
  return sum
}
```

In the code above, we used an interface called `Sumable`, which restricts the `U` type to only those types listed in `Sumable`, which are integer or float types. This support can make our generic code more robust.

Therefore, we can accomplish the following things.


1. **Count employees older than 40:**

```go
old := gCountIf(employees, func(e Employee) bool {
  return e.Age > 40
})
fmt.Printf("old people(>40): %d\n", old)
```

2. **Count employees with salary > 6000:**

```go
high_pay := gCountIf(employees, func(e Employee) bool {
  return e.Salary >= 6000
})
fmt.Printf("High Salary people(>6k): %d\n", high_pay)
```

3. **Sum salaries of employees under 30:**

```go
younger_pay := gSum(employees, func(e Employee) float32 {
  if e.Age < 30 {
    return e.Salary
  }
  return 0
})
fmt.Printf("Total Salary of Young People: %0.2f\n", younger_pay)
```

4. **Sum total vacation days:**

```go
total_vacation := gSum(employees, func(e Employee) int {
  return e.Vacation
})
fmt.Printf("Total Vacation: %d day(s)\n", total_vacation)
```

5. **Filter employees with no vacation:**

```go
no_vacation := gFilterIn(employees, func(e Employee) bool {
  return e.Vacation == 0
})
print(no_vacation)
// {Hao 44 0 8000.5} {Jack 26 0 4000} {Marry 29 0 6000}
```

---

