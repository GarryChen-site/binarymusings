---
title: "Go Functional Programming: Map, Reduce & Filter Patterns"

description: "Master Go's Map, Reduce, and Filter patterns for data processing. Learn functional programming techniques with practical examples."

summary: "Learn essential functional programming patterns in Go: Map, Reduce, and Filter operations for elegant data processing. Discover how to separate business logic from control logic with practical employee data examples and explore the path toward generic programming."

date: 2025-07-06
series: ["Golang"]
weight: 1
tags: ["golang", "functional-programming", "map-reduce", "data-processing", "programming-patterns"]
author: ["Garry Chen"]
cover:
  image: images/go-06-00.webp
  hiddenInList: true
  caption: "map-reduce"

---


In this article, we’ll learn about three very important operations in functional programming: **Map**, **Reduce**, and **Filter**. These three operations make data processing very convenient and flexible — and in most programs, we’re essentially just working with data. Especially in scenarios that involve statistics, Map/Reduce/Filter are commonly used techniques.

Let’s look at some examples first:


## Basic Examples

### Map Example

In the code below, we’ve written two `Map` functions. Each function takes two parameters:

1. A string array `[]string`, representing the data to be processed.
2. A function: either `func(s string) string` or `func(s string) int`.

```go
func MapStrToStr(arr []string, fn func(s string) string) []string {
    var newArray = []string{}
    for _, it := range arr {
        newArray = append(newArray, fn(it))
    }
    return newArray
}

func MapStrToInt(arr []string, fn func(s string) int) []int {
    var newArray = []int{}
    for _, it := range arr {
        newArray = append(newArray, fn(it))
    }
    return newArray
}
```

The logic of both functions is very similar: they iterate over the array, call the function on each element, and build a new array from the results.

So we can use them like this:

```go
var list = []string{"Linus", "Torvalds", "Linux Foundation"}

x := MapStrToStr(list, func(s string) string {
    return strings.ToUpper(s)
})
fmt.Printf("%v\n", x)
// [LINUS, TORVALDS, LINUX FOUNDATION]

y := MapStrToInt(list, func(s string) int {
    return len(s)
})
fmt.Printf("%v\n", y)
// [5, 8, 18]
```

You can see that when we passed a function to `MapStrToStr()` that converts strings to uppercase, the resulting array is all uppercase. When we passed a function to `MapStrToInt()` that calculates the length, the resulting array contained the lengths of the strings.


### Reduce Example

```go
func Reduce(arr []string, fn func(s string) int) int {
    sum := 0
    for _, it := range arr {
        sum += fn(it)
    }
    return sum
}

var list = []string{"Linus", "Torvalds", "Linux Foundation"}

x := Reduce(list, func(s string) int {
    return len(s)
})
fmt.Printf("%v\n", x)
// 31
```


### Filter Example

```go
func Filter(arr []int, fn func(n int) bool) []int {
    var newArray = []int{}
    for _, it := range arr {
        if fn(it) {
            newArray = append(newArray, it)
        }
    }
    return newArray
}

var intset = []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

out := Filter(intset, func(n int) bool {
    return n%2 == 1
})
fmt.Printf("%v\n", out)
// [1, 3, 5, 7, 9]

out = Filter(intset, func(n int) bool {
    return n > 5
})
fmt.Printf("%v\n", out)
// [6, 7, 8, 9, 10]
```

The diagram below is a metaphor that vividly illustrates the semantics of Map-Reduce. It’s very useful in data processing.

![map-reduce](images/go-06-01.png)

## Business Example

From the examples above, you might already understand that Map/Reduce/Filter are just **control logic**. The real **business logic** is defined by the data and the function you pass in. That’s right — this is a classic example of **separating business logic from control logic**, a well-known design principle.

Let’s now look at an example with real business meaning to reinforce what it means to separate business and control logic.



### Employee Information

First, we define an `Employee` object and some sample data:

```go
type Employee struct {
    Name     string
    Age      int
    Vacation int
    Salary   int
}

var list = []Employee{
    {"Hao", 44, 0, 8000},
    {"Bob", 34, 10, 5000},
    {"Alice", 23, 5, 9000},
    {"Jack", 26, 0, 4000},
    {"Tom", 48, 9, 7500},
    {"Marry", 29, 0, 6000},
    {"Mike", 32, 8, 4000},
}
```


### Related Reduce/Filter Functions

```go
func EmployeeCountIf(list []Employee, fn func(e *Employee) bool) int {
    count := 0
    for i := range list {
        if fn(&list[i]) {
            count += 1
        }
    }
    return count
}

func EmployeeFilterIn(list []Employee, fn func(e *Employee) bool) []Employee {
    var newList []Employee
    for i := range list {
        if fn(&list[i]) {
            newList = append(newList, list[i])
        }
    }
    return newList
}

func EmployeeSumIf(list []Employee, fn func(e *Employee) int) int {
    var sum = 0
    for i := range list {
        sum += fn(&list[i])
    }
    return sum
}
```

Quick explanation:

* `EmployeeCountIf` and `EmployeeSumIf` are for counting and summing values that meet a condition. They are like Filter + Reduce combined.
* `EmployeeFilterIn` is a Filter function based on a condition.



### Custom Statistics Examples

1. **Count how many employees are over 40:**

```go
old := EmployeeCountIf(list, func(e *Employee) bool {
    return e.Age > 40
})
fmt.Printf("Old people: %d\n", old)
// Old people: 2
```

2. **Count how many employees earn more than 6000:**

```go
high_pay := EmployeeCountIf(list, func(e *Employee) bool {
    return e.Salary >= 6000
})
fmt.Printf("High Salary people: %d\n", high_pay)
// High Salary people: 4
```

3. **List employees who haven’t taken vacation:**

```go
no_vacation := EmployeeFilterIn(list, func(e *Employee) bool {
    return e.Vacation == 0
})
fmt.Printf("People with no vacation: %v\n", no_vacation)
// People with no vacation: [{Hao 44 0 8000} {Jack 26 0 4000} {Marry 29 0 6000}]
```

4. **Calculate total salary of all employees:**

```go
total_pay := EmployeeSumIf(list, func(e *Employee) int {
    return e.Salary
})
fmt.Printf("Total Salary: %d\n", total_pay)
// Total Salary: 43500
```

5. **Calculate total salary for employees under 30:**

```go
younger_pay := EmployeeSumIf(list, func(e *Employee) int {
    if e.Age < 30 {
        return e.Salary
    }
    return 0
})
```


## Generic Map-Reduce

As you can see, all the above Map-Reduce examples had to be written differently depending on the data types. Even though the code looks quite similar, the lack of type generality means we had to repeat ourselves. This naturally leads to the idea of **generic programming**.

---
