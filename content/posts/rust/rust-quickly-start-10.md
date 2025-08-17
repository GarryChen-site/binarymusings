---
title: "Rust Lifetimes: 'static, Annotations and Elision Rules"

description: "Understand Rust lifetimes with examples: static vs dynamic, lifetime annotations in function signatures and structs, elision rules, borrow checks."

summary: "Learn how Rust models lifetimes for stack and heap, and how the compiler enforces constraints across function calls. Walk through 'static vs dynamic lifetimes, lifetime parameters and elision rules, fixing the classic max example, a strtok exercise, and annotating structs—plus when to annotate vs rely on inference."

date: 2025-08-17
series: ["Rust"]
weight: 1
tags: ["rust", "lifetimes", "lifetime-annotations", "borrow-checker", "references"]
author: ["Garry Chen"]
cover:
  image: images/rust-10-00.webp
  hiddenInList: true
  caption: "Rust lifetimes"

---



Previously, we mentioned that in any programming language, values on the stack have their own lifetimes, which match the lifetime of the stack frame they belong to. Rust takes this concept further by also introducing lifetimes for memory on the heap.

We know that in other languages, the lifetime of heap memory is uncertain or undefined. This means it’s either manually managed by the developer, or the language performs additional runtime checks. In Rust, unless you explicitly perform operations such as `Box::leak()`, `Box::into_raw()`, or `ManualDrop`, **the heap memory’s lifetime is generally bound to the lifetime of its associated stack allocation**.

Under this default behavior, the compiler can compare the lifetime of a value with that of its references in each function scope to ensure that “a reference’s lifetime does not outlive the value it points to.”

But have you ever wondered how the Rust compiler actually achieves this?


# The lifetime of a value

Before we go deeper, let’s define the possible lifetimes a value can have.

If a value’s lifetime **spans the entire duration of the process**, we call this a **static lifetime**.

When a value has a static lifetime, any reference to it also has a static lifetime. We denote such references with `'static`. For example, `&'static str` represents a string reference with a static lifetime.

Typically, global variables, static variables, and string literals all have static lifetimes. Heap memory we mentioned earlier will also have a static lifetime if we call `Box::leak` on it.

If a value is **defined within a scope—created** on the stack or heap—its lifetime is **dynamic**.

When this scope ends, the value’s lifetime ends as well. For dynamic lifetimes, we use lowercase letters or identifiers like `'a`, `'b`, or `'hello`. The specific name after `'` doesn’t matter—it just marks some finite lifetime. For example, `&'a str` and `&'b str` indicate that the two string references may have different lifetimes.

Summary diagram:

![lifetime_diagram](images/rust-10-01.webp)

* Memory allocated on the heap or stack has its own scope, and its lifetime is dynamic.
* Global variables, static variables, string literals, and code are compiled into the **BSS/Data/RoData/Text** segments of the executable and loaded into memory at startup. Their lifetime matches the lifetime of the process, so they are static.
* Function pointers also have static lifetimes because functions reside in the Text segment, and as long as the process is alive, their memory exists.

With these basics in mind, let’s see how the compiler recognizes lifetimes for values and references.


# How the compiler determines lifetimes

Let’s start with two simple examples.

* **Example 1**: `x` references a variable `y` created in an inner scope. Since `y`’s lifetime (`'b`) ends earlier than `x`’s lifetime (`'a`), the compiler errors when `x` tries to reference `y`.
* **Example 2**: `y` and `x` are in the same scope, and `x` references `y`. Here, `x`’s lifetime `'a` ends at the same time or earlier than `y`’s `'b`, so it’s allowed.

![lifetime_example](images/rust-10-02.webp)

These two small examples are easy to understand,let’s then look at a slightly more complex one.

The sample code in main() function creates two Strings, then passes them into the max() function to compare size.`max()` function accepts two string references, returns the reference of the bigger one.

```rust
fn main() {
    let s1 = String::from("Lindsey");
    let s2 = String::from("Rosie");

    let result = max(&s1, &s2);
    println!("bigger one: {}", result);
}

fn max(s1: &str, s2: &str) -> &str {
    if s1 > s2 {
        s1
    } else {
        s2
    }
}
```

This code fails to compile with a “missing lifetime specifier” error. In other words, when compiling `max()`, **the compiler cannot determine the relationship between the lifetimes of `s1`, `s2`, and the return value**.

Are you not very puzzled? From our developer perspective, this code is very intuitive — in `main()` function, `s1` and `s2` have the same lifetime, after their references are passed to max() function, no matter which one is returned, the lifetime will not exceed `s1` or `s2`. So this should be correct code, right?

Why did the compiler report an error and not allow it to compile? Let’s slightly extend this code, and you’ll understand the compiler’s confusion.

In the previous sample code, we create a new function `get_max()`, it accepts a string reference, then compares it with "Cynthia" this string literal. Earlier we mentioned, the lifetime of a string literal is static, while `s1` is dynamic — their lifetimes are clearly not the same.

```rust
fn main() {
    let s1 = String::from("Lindsey");
    let s2 = String::from("Rosie");

    let result = max(&s1, &s2);
    println!("bigger one: {}", result);

    let result = get_max(&s1);
    println!("bigger one: {}", result);
}

fn get_max(s1: &str) -> &str {
    max(s1, "Cynthia")
}

fn max(s1: &str, s2: &str) -> &str {
    if s1 > s2 {
        s1
    } else {
        s2
    }
}
```

When multiple parameters appear, and their lifetimes may not be the same, the return value’s lifetime is hard to determine. When compiling some function, the compiler does not know who will call this function, or how it will be called. So **the information the function itself carries is all the information the compiler can use at compile time**.

According to this, let’s look again at the sample code — when compiling the `max()` function, what is the relationship between parameters `s1` and `s2`’s lifetimes, and what is the relationship between the return value and parameters’ lifetimes, the compiler cannot determine.

At this time, we need to provide lifetime information in the function signature, that is lifetime annotations.When writing lifetime annotations, the parameters used are called lifetime parameters. Through lifetime annotations, we tell the compiler the constraints between the lifetimes of these references.

The way to describe lifetime parameters is the same as generic parameters, but only lowercase letters are used.Here, the two input parameters `s1`, `s2`, and the return value are all constrained by `'a`. **A lifetime parameter describes the relationship between parameter and parameter, and between parameter and return value — it does not change the original lifetime**.

After we add the lifetime parameter, as long as `s1` and `s2`’s lifetimes are greater than or equal to (outlive) `'a`, they meet the parameter constraint, and the return value’s lifetime similarly needs to be greater than or equal to `'a`.

When you run the above sample code, the compiler has already suggested you can modify the `max()` function like this:

```rust
fn max<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1 > s2 {
        s1
    } else {
        s2
    }
}
```

When `main()` calls `max()` function, s1 and s2 have the same lifetime `'a`, so it satisfies (`s1: &'a str, s2: &'a str`) constraint. When `get_max()` calls `max()`, "Cynthia" is static lifetime, it is greater than s1’s lifetime 'a, so it also can satisfy max()’s constraint requirement.



# Do Your References Need Extra Lifetime Annotations?

At this point, you might be wondering: *Why is it that in some of my previous code, many function parameters or return values used references, yet the compiler didn’t require me to explicitly annotate their lifetimes?*

This is because the compiler tries to minimize the burden on developers. In fact, any function that uses references needs lifetime annotations, but the compiler often infers them automatically to save you the trouble.

For example, consider this function `first()` which takes a string reference, finds the first word in it, and returns that word:

```rust
fn main() {
    let s1 = "Hello world";
    println!("first word of s1: {}", first(&s1));
}

fn first(s: &str) -> &str {
    let trimmed = s.trim();
    match trimmed.find(' ') {
        None => "",
        Some(pos) => &trimmed[..pos],
    }
}
```

Even though we didn’t annotate any lifetimes, the compiler automatically applies them based on a few simple rules:

1. All reference-type parameters have independent lifetimes `'a`, `'b`, etc.
2. If there’s only one reference-type parameter, its lifetime is applied to all return values.
3. If there are multiple reference-type parameters and one of them is `self`, its lifetime is applied to all return values.

Rule 3 applies to traits or custom data types, which we’ll discuss later.

For `first()`, using rules 1 and 2, the compiler infers:

```rust
fn first<'a>(s: &'a str) -> &'a str {
    let trimmed = s.trim();
    match trimmed.find(' ') {
        None => "",
        Some(pos) => &trimmed[..pos],
    }
}
```

As you can see, all references get valid lifetime annotations without conflicts.

Now compare that with the earlier example of returning the larger of two strings (`max()`). Why can’t the compiler handle that case automatically?

According to rule 1, we can annotate `s1` and `s2` with `'a` and `'b` respectively. But what about the return value? Should it have lifetime `'a` or `'b`? This is ambiguous, and the compiler can’t decide:

```rust
fn max<'a, 'b>(s1: &'a str, s2: &'b str) -> &'??? str
```

Only *we*, understanding the code logic, can correctly specify the constraints between parameters and return values so that it compiles.


# Lifetime Annotation Exercise

Let’s try implementing a string-splitting function `strtok()` to practice adding lifetime annotations.

If you’ve used C/C++, you’ve probably seen `strtok()`—it splits a string by a delimiter, returns the token, and advances the original string pointer to the next token.

In Rust, it’s not hard to implement. Since the input `s` needs to be a mutable reference to a string reference, its type is `&mut &str`:

```rust
pub fn strtok(s: &mut &str, delimiter: char) -> &str {
    if let Some(i) = s.find(delimiter) {
        let prefix = &s[..i];
        // Delimiter can be UTF-8, so we get its UTF-8 length
        let suffix = &s[(i + delimiter.len_utf8())..];
        *s = suffix;
        prefix
    } else {
        let prefix = *s;
        *s = "";
        prefix
    }
}

fn main() {
    let s = "hello world".to_owned();
    let mut s1 = s.as_str();
    let hello = strtok(&mut s1, ' ');
    println!("hello is: {}, s1: {}, s: {}", hello, s1, s);
}
```

If you run this, you’ll get a lifetime-related compile error. This is because after lifetime inference, `&mut &str` becomes `&'b mut &'a str`, and the returned `&str` can’t have a clearly inferred lifetime.

Solving the Problem, ask yourself: is the return value’s lifetime tied to the mutable reference `&mut`, or to the inner string reference `&str`?

Clearly, it’s the latter. So we annotate only the relevant part:

```rust
pub fn strtok<'b, 'a>(s: &'b mut &'a str, delimiter: char) -> &'a str { ... }
```

Since the return value’s lifetime is tied to the string reference, we can simplify and let the compiler infer the rest:

```rust
pub fn strtok<'a>(s: &mut &'a str, delimiter: char) -> &'a str { ... }
```

Final working version:

```rust
pub fn strtok<'a>(s: &mut &'a str, delimiter: char) -> &'a str {
    if let Some(i) = s.find(delimiter) {
        let prefix = &s[..i];
        let suffix = &s[(i + delimiter.len_utf8())..];
        *s = suffix;
        prefix
    } else {
        let prefix = *s;
        *s = "";
        prefix
    }
}

fn main() {
    let s = "hello world".to_owned();
    let mut s1 = s.as_str();
    let hello = strtok(&mut s1, ' ');
    println!("hello is: {}, s1: {}, s: {}", hello, s1, s);
}
```

To help you understand this function’s lifetime relationships better, I’ve drawn a diagram showing how each variable on the heap and stack relates to the others.

![function lifetime diagram](images/rust-10-03.webp)

**Tip:** If you find certain code difficult to reason about, draw a similar diagram showing heap and stack relationships—it makes the logic much clearer.


When handling lifetimes, the compiler automatically applies annotations according to certain rules. However, when inference conflicts arise, we need to annotate manually.

**The purpose of lifetime annotations is to establish a relationship (constraint) between parameters and return values**. When calling a function, the lifetime of the provided arguments must outlive the lifetime specified in the annotation.

Once every function has the correct annotations, the compiler can verify at call sites whether the references’ lifetimes match the function signature’s constraints. If they don’t match, it violates “a reference’s lifetime cannot exceed the lifetime of the value it points to,” and the compiler will error.


If you understand function lifetime annotations, data structure lifetime annotations work similarly.

Example:

```rust
struct Employee<'a, 'b> {
    name: &'a str,
    title: &'b str,
    age: u8,
}
```

Here, `Employee` contains string references for `name` and `title`. The lifetime of `Employee` must be less than or equal to the lifetimes of both `name` and `title`—otherwise, it would access invalid memory.

When using such a struct, the struct’s own lifetime must not exceed the lifetime of any of its referenced fields.


# Summary

Today we introduced the concepts of static lifetime and dynamic lifetime, as well as how the compiler identifies the lifetimes of values and references.

According to the ownership rules, the lifetime of a value can be determined — it can live until the owner leaves the scope; whereas the lifetime of a reference cannot exceed the lifetime of the value. In the same scope, this is obvious. However, **when a function call occurs, the compiler needs to determine, through the function’s signature, the constraints between the lifetimes of parameters and return values**.

In most cases, the compiler can, through the rules in the context, automatically add lifetime constraints.
If they cannot be added automatically, then the developer needs to manually add the constraints.
Generally, we only need to determine which parameter’s lifetime is related to the return value’s lifetime.
As for data structures, when there are references inside, we need to annotate the lifetimes for those references.





