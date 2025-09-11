---
title: "Go Visitor Pattern: Learn from kubectl Implementation"

description: "Master Go's Visitor pattern through kubectl's functional approach. Learn decorators, pipelines, and Kubernetes command-line architecture."

summary: "Discover how Kubernetes kubectl implements the Visitor design pattern in Go using functional programming."

date: 2025-07-04
series: ["Golang"]
weight: 1
tags: ["golang", "visitor-pattern", "kubernetes", "design-patterns", "kubectl"]
author: ["Garry Chen"]
cover:
  image: images/go-05-00.jpg
  hiddenInList: true
  caption: "Kubernetes"

---

This article mainly aims to discuss a programming pattern used in Kubernetes’s `kubectl` command — the **Visitor** pattern (note: in fact, `kubectl` mainly uses two patterns: **Builder** and **Visitor**). Originally, **Visitor** is an important design pattern in object-oriented programming (see the Wikipedia entry [Visitor Pattern](https://en.wikipedia.org/wiki/Visitor_pattern)). This pattern is a method of separating algorithms from the structure of the objects on which they operate. The practical result of this separation is the ability to add new operations to existing object structures without modifying those structures, which follows the **Open/Closed Principle**.

This article focuses on how `kubectl` implements this pattern using a **functional approach**.



## A Simple Example

Let’s begin with a basic example of the Visitor design pattern.

In our code, there’s a function type `Visitor`, and an interface `Shape`, which requires the use of a `Visitor` function as a parameter. 
Our example object types `Circle` and `Rectangle` implement the `Shape` interface’s `accept()` method, which is where the `Visitor` is passed in from the outside.

```go
package main

import (
    "encoding/json"
    "encoding/xml"
    "fmt"
)

type Visitor func(shape Shape)

type Shape interface {
    accept(Visitor)
}

type Circle struct {
    Radius int
}

func (c Circle) accept(v Visitor) {
    v(c)
}

type Rectangle struct {
    Width, Heigh int
}

func (r Rectangle) accept(v Visitor) {
    v(r)
}
```

Now we implement two Visitors — one for JSON serialization and one for XML serialization:

```go
func JsonVisitor(shape Shape) {
    bytes, err := json.Marshal(shape)
    if err != nil {
        panic(err)
    }
    fmt.Println(string(bytes))
}

func XmlVisitor(shape Shape) {
    bytes, err := xml.Marshal(shape)
    if err != nil {
        panic(err)
    }
    fmt.Println(string(bytes))
}
```

Here’s the code that uses the Visitor pattern:

```go
func main() {
    c := Circle{10}
    r := Rectangle{100, 200}
    shapes := []Shape{c, r}

    for _, s := range shapes {
        s.accept(JsonVisitor)
        s.accept(XmlVisitor)
    }
}
```

In fact, the purpose of this code is to **decouple the data structures from the algorithms**. The Strategy pattern can also achieve this and may look cleaner. However, in some cases, multiple Visitors are used to access **different parts of a data structure**. In such scenarios, the data structure is somewhat like a database, and each Visitor acts as a small application. `kubectl` is one such case.



## Kubernetes Background

Next, let’s go over some background knowledge:

* Kubernetes abstracts many types of **Resources**, such as: `Pod`, `ReplicaSet`, `ConfigMap`, `Volumes`, `Namespace`, `Roles`, etc. These various types make up Kubernetes’s **data model**. (Click [Kubernetes Resources Map](https://github.com/kubernauts/practical-kubernetes-problems/blob/master/images/k8s-resources-map.png) to see how complex it is.)
* `kubectl` is a command-line client tool in Kubernetes. Operators use it to manage the cluster. `kubectl` communicates with the Kubernetes **API Server**, which in turn talks to each node’s **kubelet** to control them.
* The main job of `kubectl` is to process user inputs (CLI arguments, YAML files, etc.), organize them into structured data, and send it to the API Server.
* Related source code is found at: `src/k8s.io/cli-runtime/pkg/resource/visitor.go` ([source link](https://github.com/kubernetes/kubernetes/blob/cea1d4e20b4a7886d8ff65f34c6d4f95efcb4742/staging/src/k8s.io/cli-runtime/pkg/resource/visitor.go)).

`kubectl`’s code is fairly complex, but its core principle is simple: it **uses the Builder pattern** to parse command-line input and YAML into Resources, and then **uses the Visitor pattern** to iterate and process those Resources.



## kubectl Implementation

### Visitor Pattern Definition

First, `kubectl` mainly processes a structure called `Info`. Here's the related definition:

```go
type VisitorFunc func(*Info, error) error

type Visitor interface {
    Visit(VisitorFunc) error
}

type Info struct {
    Namespace   string
    Name        string
    OtherThings string
}

func (info *Info) Visit(fn VisitorFunc) error {
    return fn(info, nil)
}
```

We can see:

* A function type `VisitorFunc` is defined.
* A `Visitor` interface requires a method `Visit(VisitorFunc) error` (just like the `Shape` interface from the earlier example).
* The `Info` struct implements the `Visit()` method of the Visitor interface by **simply calling the passed-in function**.

Let’s Define Several Visitor Types

### Name Visitor

This visitor accesses the `Name` and `Namespace` fields in the `Info` struct:

```go
type NameVisitor struct {
    visitor Visitor
}

func (v NameVisitor) Visit(fn VisitorFunc) error {
    return v.visitor.Visit(func(info *Info, err error) error {
        fmt.Println("NameVisitor() before call function")
        err = fn(info, err)
        if err == nil {
            fmt.Printf("==> Name=%s, NameSpace=%s\n", info.Name, info.Namespace)
        }
        fmt.Println("NameVisitor() after call function")
        return err
    })
}
```

What we see here:

* `NameVisitor` is a struct that **wraps another Visitor** — this implies polymorphism.
* Its `Visit()` method calls the `Visit()` of the inner Visitor, acting like a **decorator**.



### OtherThings Visitor

This visitor accesses the `OtherThings` field:

```go
type OtherThingsVisitor struct {
    visitor Visitor
}

func (v OtherThingsVisitor) Visit(fn VisitorFunc) error {
    return v.visitor.Visit(func(info *Info, err error) error {
        fmt.Println("OtherThingsVisitor() before call function")
        err = fn(info, err)
        if err == nil {
            fmt.Printf("==> OtherThings=%s\n", info.OtherThings)
        }
        fmt.Println("OtherThingsVisitor() after call function")
        return err
    })
}
```


### Log Visitor

```go
type LogVisitor struct {
    visitor Visitor
}

func (v LogVisitor) Visit(fn VisitorFunc) error {
    return v.visitor.Visit(func(info *Info, err error) error {
        fmt.Println("LogVisitor() before call function")
        err = fn(info, err)
        fmt.Println("LogVisitor() after call function")
        return err
    })
}
```


### Code that Uses the Visitors

Let’s look at how to use the above code:

```go
func main() {
    info := Info{}
    var v Visitor = &info
    v = LogVisitor{v}
    v = NameVisitor{v}
    v = OtherThingsVisitor{v}

    loadFile := func(info *Info, err error) error {
        info.Name = "Linus"
        info.Namespace = "Linux"
        info.OtherThings = "Talk is cheap, show me the code."
        return nil
    }

    v.Visit(loadFile)
}
```

What we can observe:

* Visitors are **nested one inside another**.
* `loadFile` simulates reading data from a file.
* The final call `v.Visit(loadFile)` triggers the full chain.

Expected output:

```
LogVisitor() before call function
NameVisitor() before call function
OtherThingsVisitor() before call function
==> OtherThings=We are running as remote team.
OtherThingsVisitor() after call function
==> Name=Hao Chen, NameSpace=MegaEase
NameVisitor() after call function
LogVisitor() after call function
```

Effects of the above code:

* Decouples **data** and **processing logic**.
* Uses **decorator pattern**.
* Achieves a **pipeline style** of processing.



### Visitor Decorator

Now let’s refactor the above using the decorator pattern:

```go
type DecoratedVisitor struct {
    visitor    Visitor
    decorators []VisitorFunc
}

func NewDecoratedVisitor(v Visitor, fn ...VisitorFunc) Visitor {
    if len(fn) == 0 {
        return v
    }
    return DecoratedVisitor{v, fn}
}

func (v DecoratedVisitor) Visit(fn VisitorFunc) error {
    return v.visitor.Visit(func(info *Info, err error) error {
        if err != nil {
            return err
        }
        if err := fn(info, nil); err != nil {
            return err
        }
        for i := range v.decorators {
            if err := v.decorators[i](info, nil); err != nil {
                return err
            }
        }
        return nil
    })
}
```

Explanation:

* `DecoratedVisitor` stores all `VisitorFunc` functions.
* `NewDecoratedVisitor` takes any number of `VisitorFunc` and constructs the object.
* `Visit()` loops through and executes each function.

Now the usage becomes:

```go
info := Info{}
var v Visitor = &info
v = NewDecoratedVisitor(v, NameVisitor, OtherVisitor)

v.Visit(LoadFile)
```

Isn’t that simpler than before?

Note: `DecoratedVisitor` itself **can also be treated as a Visitor**.


All of the above patterns and logic are present in `kubectl`'s source code. If you understand the logic here, you’ll likely be able to read and comprehend the `kubectl` source code too.


---