---
title: "Go Inversion of Control Guide: Struct Embedding & Design"

description: "Learn Go's Inversion of Control with struct embedding, method override, and dependency inversion. Build reusable control logic patterns."

summary: "Master Go's Inversion of Control using struct embedding, polymorphism, and dependency inversion. Learn to separate control logic from business logic with practical examples including undo functionality and reusable design patterns."

date: 2025-07-02
series: ["Golang"]
weight: 1
tags: ["golang", "design-patterns", "struct-embedding", "inversion-of-control", "dependency-injection"]
author: ["Garry Chen"]
cover:
  image: images/go-04-00.webp
  hiddenInList: true
  caption: "Control"

---


**[Inversion of Control (IoC)](https://en.wikipedia.org/wiki/Inversion_of_control)** – Inversion of Control is a method of software design. Its main idea is to separate control logic from business logic. Don't write control logic inside business logic, because this will make control logic depend on business logic. Instead, it should be the other way around: make business logic depend on control logic. This kind of programming style can effectively reduce program complexity and improve code reuse.

We won’t talk about object-oriented design patterns here. Let’s take a look at an example in Go that uses the `embed` structure.

## Embedding and Delegation

### Struct Embedding
In Go, we can easily embed one struct into another struct. As shown below:

```go
type Widget struct {
    X, Y int
}
type Label struct {
    Widget        // Embedding (delegation)
    Text   string // Aggregation
}
```

In the example above, we embed `Widget` into `Label`, so we can use it like this:

```go
label := Label{Widget{10, 10}, "State:"}

label.X = 11
label.Y = 12
```

If a name conflict occurs in the `Label` struct, it must be resolved. For example, if member `X` is duplicated, `label.X` refers to `Label`’s own `X`, while `label.Widget.X` refers to the embedded one.

With this kind of embedding, we can design the structure layer by layer, like UI components. For example, we can create two new structs: `Button` and `ListBox`:

```go
type Button struct {
    Label // Embedding (delegation)
}

type ListBox struct {
    Widget          // Embedding (delegation)
    Texts  []string // Aggregation
    Index  int      // Aggregation
}
```


### Method Override

Then, we define two interfaces: `Painter` for drawing components and `Clicker` for click events:

```go
type Painter interface {
    Paint()
}
 
type Clicker interface {
    Click()
}
```

Of course:

* For `Label`, it only has `Painter`, not `Clicker`.
* For `Button` and `ListBox`, they both have `Painter` and `Clicker`.

Here are some implementations:

```go
func (label Label) Paint() {
  fmt.Printf("%p:Label.Paint(%q)\n", &label, label.Text)
}
```

Since this interface can be brought into the new struct through `Label` embedding, we can override it in `Button`:

```go
func (button Button) Paint() { // Override
    fmt.Printf("Button.Paint(%s)\n", button.Text)
}
func (button Button) Click() {
    fmt.Printf("Button.Click(%s)\n", button.Text)
}
```

```go
func (listBox ListBox) Paint() {
    fmt.Printf("ListBox.Paint(%q)\n", listBox.Texts)
}
func (listBox ListBox) Click() {
    fmt.Printf("ListBox.Click(%q)\n", listBox.Texts)
}
```

Here, we need to especially emphasize that the `Button.Paint()` interface can be inherited from `Label` via embedding. If `Button.Paint()` is not implemented, it will call `Label.Paint()`. Therefore, defining `Paint()` inside `Button` is essentially an **override**.


### Embedded Struct Polymorphism

Through the following program, we can see how the whole polymorphism works:

```go
button1 := Button{Label{Widget{10, 70}, "OK"}}
button2 := NewButton(50, 70, "Cancel")
listBox := ListBox{Widget{10, 40}, 
[]string{"AL", "AK", "AZ", "AR"}, 0}

for _, painter := range []Painter{label, listBox, button1, button2} {
    painter.Paint()
}
 
for _, widget := range []interface{}{label, listBox, button1, button2} {
  widget.(Painter).Paint()
  if clicker, ok := widget.(Clicker); ok {
    clicker.Click()
  }
  fmt.Println() // print a empty line 
}
```

We can see that we can use **interfaces** for polymorphism, and also use the **generic `interface{}`** for polymorphism, but it requires a **type assertion**.



## Inversion of Control

Let’s look at another example. We have a data structure that stores integers, as shown below:

```go
type IntSet struct {
    data map[int]bool
}
func NewIntSet() IntSet {
    return IntSet{make(map[int]bool)}
}
func (set *IntSet) Add(x int) {
    set.data[x] = true
}
func (set *IntSet) Delete(x int) {
    delete(set.data, x)
}
func (set *IntSet) Contains(x int) bool {
    return set.data[x]
}
```

This implements three operations: `Add()`, `Delete()`, and `Contains()`. The first two are write operations; the last one is a read operation.



### Implementing Undo Functionality

Now we want to implement an **Undo** feature. We can wrap `IntSet` into an `UndoableIntSet`, as shown below:

```go
type UndoableIntSet struct { // Poor style
    IntSet    // Embedding (delegation)
    functions []func()
}
 
func NewUndoableIntSet() UndoableIntSet {
    return UndoableIntSet{NewIntSet(), nil}
}

func (set *UndoableIntSet) Add(x int) { // Override
    if !set.Contains(x) {
        set.data[x] = true
        set.functions = append(set.functions, func() { set.Delete(x) })
    } else {
        set.functions = append(set.functions, nil)
    }
}

func (set *UndoableIntSet) Delete(x int) { // Override
    if set.Contains(x) {
        delete(set.data, x)
        set.functions = append(set.functions, func() { set.Add(x) })
    } else {
        set.functions = append(set.functions, nil)
    }
}

func (set *UndoableIntSet) Undo() error {
    if len(set.functions) == 0 {
        return errors.New("No functions to undo")
    }
    index := len(set.functions) - 1
    if function := set.functions[index]; function != nil {
        function()
        set.functions[index] = nil // For garbage collection
    }
    set.functions = set.functions[:index]
    return nil
}
```

In the above code, we can see:

* We embed `IntSet` in `UndoableIntSet`, then override its `Add()` and `Delete()` methods.
* The `Contains()` method is not overridden, so it’s brought into `UndoableIntSet`.
* In the overridden `Add()`, we record a `Delete()` operation.
* In the overridden `Delete()`, we record an `Add()` operation.
* The newly added `Undo()` method performs the undo operation.

Using this approach to extend new functionality onto existing code is a good choice. It balances reusing original functionality with adding new capabilities. However, the biggest problem with this approach is that `Undo` functionality is actually **control logic**, not **business logic**. Therefore, reusing the `Undo` function is problematic—because it embeds a lot of `IntSet`-specific business logic.


### Dependency Inversion

Now let’s look at another method:

We first declare a function interface that represents the function signature acceptable for `Undo` control:

```go
type Undo []func()
```

With the above protocol, our undo control logic can be written like this:

```go
func (undo *Undo) Add(function func()) {
  *undo = append(*undo, function)
}

func (undo *Undo) Undo() error {
  functions := *undo
  if len(functions) == 0 {
    return errors.New("No functions to undo")
  }
  index := len(functions) - 1
  if function := functions[index]; function != nil {
    function()
    functions[index] = nil // For garbage collection
  }
  *undo = functions[:index]
  return nil
}
```

Don’t be surprised here—`Undo` is just a type. It doesn’t have to be a struct. Being a function array is perfectly acceptable.

Then, we embed `Undo` into our `IntSet`, and use the above method inside `Add()` and `Delete()` to complete the functionality:

```go
type IntSet struct {
    data map[int]bool
    undo Undo
}
 
func NewIntSet() IntSet {
    return IntSet{data: make(map[int]bool)}
}

func (set *IntSet) Undo() error {
    return set.undo.Undo()
}
 
func (set *IntSet) Contains(x int) bool {
    return set.data[x]
}

func (set *IntSet) Add(x int) {
    if !set.Contains(x) {
        set.data[x] = true
        set.undo.Add(func() { set.Delete(x) })
    } else {
        set.undo.Add(nil)
    }
}
 
func (set *IntSet) Delete(x int) {
    if set.Contains(x) {
        delete(set.data, x)
        set.undo.Add(func() { set.Add(x) })
    } else {
        set.undo.Add(nil)
    }
}
```

This **is** inversion of control. The control logic `Undo` no longer depends on the business logic `IntSet`; instead, the business logic `IntSet` depends on `Undo`.

What it depends on is actually just a **protocol**, which is an array of no-parameter functions. As we can also see, our `Undo` code is now reusable.

---

