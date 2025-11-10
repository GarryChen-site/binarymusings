---
title: "Rust Type System & Generics: Complete Guide to Polymorphism"

description: "Master Rust's type system: strong typing, static analysis, type inference, generic data structures, and parametric polymorphism with examples."

summary: "Dive deep into Rust's powerful type system and generic programming. Learn about strong typing vs weak typing, static vs dynamic type checking, type inference rules, and how to build generic data structures and functions. Explore parametric polymorphism, monomorphization, lifetime constraints, and best practices for writing flexible, reusable Rust code."

date: 2025-08-31
series: ["Rust"]
weight: 1
tags: ["rust", "type-system", "generics", "polymorphism", "type-inference"]
author: ["Garry Chen"]
cover:
  image: images/rust-12-00.webp
  hiddenInList: true
  caption: "rust type system overview"

---



If you use a static language like C that doesn’t support generics, or a dynamic language like Python/Ruby/JavaScript, this part may be a bit challenging, so be ready to adjust your way of thinking; if you use static languages that support generics like C++/Java/Swift, you can compare Rust with them to see the similarities and differences.

Actually, in previous articles, we have already written quite a bit of Rust code and used various data structures, so I believe you already have a very rough impression of Rust’s type system. But what exactly is a type system? What can it be used for? When should we use it? Today, let’s find out.

As one of the core elements of a language, the type system largely shapes the user experience of the language as well as the safety of programs. Why is that? Because, in the world of machine code, there is no such thing as types—instructions only interact with immediate values or memory, and the data stored in memory is just a stream of bytes.

So, you could say **the type system is purely a tool**: the compiler performs static checks on data at compile time, or the language performs dynamic checks on data at runtime, to ensure that the data being processed by an operation is of the type the developer expects.

Now, can you understand why Rust’s type system is so strict when it comes to type issues (always throwing errors)?

## Basic concepts and classifications of type systems

Before specifically talking about Rust’s type system, let’s first clarify some concepts of type systems and reach a basic shared understanding.

As mentioned in this [article]({{< ref "rust-quickly-start-02.md" >}}), a type is a distinction of values—it includes information such as the length of the value in memory, alignment, and what operations the value can perform.

For example, the `u32` type is an unsigned 32-bit integer, its length is 4 bytes, alignment is also 4 bytes, and its value range is 0–4G; the `u32` type implements interfaces like addition, subtraction, multiplication, division, and comparisons, so you can do operations like 1 + 2 or i <= 3.

**The type system is essentially the system for defining, checking, and handling types**. So, depending on the stage at which type operations are performed, there are different classification standards, and accordingly different categories.

- By whether a type can be implicitly converted after being defined, we can divide into strongly typed and weakly typed. Rust does not allow automatic conversion between different types, so it is a strongly typed language, whereas C / C++ / JavaScript allow automatic conversion, making them weakly typed languages.

- By the timing of type checking—compile-time vs. runtime—we can divide into static type systems and dynamic type systems. Static type systems can be further divided into explicit static and implicit static. Rust / Java / Swift are explicit static languages, while Haskell is an implicit static language.

In type systems, polymorphism is a very important idea. **It means that when using the same interface, different types of objects will adopt different implementations**.

For dynamic type systems, polymorphism is implemented through [duck typing](https://en.wikipedia.org/wiki/Duck_typing); while for static type systems, polymorphism can be achieved through [parametric polymorphism](https://en.wikipedia.org/wiki/Parametric_polymorphism), [adhoc polymorphism](https://en.wikipedia.org/wiki/Ad_hoc_polymorphism), and [subtype polymorphism](https://en.wikipedia.org/wiki/Subtyping).

* Parametric polymorphism means that the type operated on by the code is a parameter that meets certain constraints, rather than a concrete type.

* Adhoc polymorphism means that the same behavior can have multiple different implementations. For example, addition can be 1 + 1, or "abc" + "cde", or matrix1 + matrix2, or even matrix1 + vector1. In object-oriented languages, adhoc polymorphism generally refers to function overloading.

* Subtype polymorphism means that at runtime, a subtype can be used as its parent type.

In Rust, parametric polymorphism is supported through generics, adhoc polymorphism is supported through traits, and subtype polymorphism can be supported using trait objects. We’ll talk about parametric polymorphism now, and the other two will be covered in later articles.

You can look at the following diagram to better clarify the relationships between these concepts:

![type system concepts](images/rust-12-01.webp)

## Rust Type System

Okay, now that we’ve mastered the basic concepts and classifications of type systems, let’s look at Rust’s type system.

According to the classifications just mentioned: at the definition stage, Rust does not allow implicit type conversions, which means Rust is a strongly typed language; at the checking stage, Rust uses a static type system, ensuring type correctness at compile time. Strong typing plus static typing makes Rust a type-safe language.

Actually, speaking of “type safety,” we often hear this term, but do you really know what it means?

From a memory perspective, **type safety means that code can only access the memory it is authorized to, and only in the allowed ways**.

For example, for an array of length 4 storing `u64` data: code accessing this array can only access within the 32 bytes of memory between the start and end address of the array, and accesses must be aligned by 8 bytes. Moreover, each element in the array can only perform operations allowed by the `u64` type. The compiler strictly checks the code to ensure this behavior. Let’s look at the diagram:

![rust array layout in memory](images/rust-12-02.webp)

So, weakly typed languages like C/C++ that allow implicit type conversion after definition are not memory safe, while strongly typed languages like Rust are type-safe, preventing developers from accidentally introducing an implicit conversion that leads to reading incorrect data or even out-of-bounds memory access.

On top of this, Rust further separates read and write permissions for memory access. So, **Rust’s memory safety is even stricter: code can only access memory it’s authorized to, in ways and with permissions it’s authorized for**.

To achieve this level of strict type safety, in Rust everything except definition statements like let / fn / static / const are expressions, and every expression has a type. So you can say that in Rust, types are everywhere.

You might be wondering, what’s the type of code like this?

``` rust
if has_work {
    do_something();
}
```

In Rust, for any scope—whether it’s an if / else / for loop, or a function—the return value of the last expression is the return value of the scope. If an expression or function doesn’t return any value, then it returns a unit `()`. unit is a type with only one value, whose value and type are both `()`.

So in the above if block, its type and return value are `()`. When placed inside a function with no return value, like this:

``` rust
fn work(has_work: bool) {
    if has_work {
        do_something();
    }
}
```


The logic of “types everywhere” in Rust still holds consistently.

The unit type has very broad applications. Besides being used as a return value, it is also heavily used in data structures. For example, `Result<(), Error>` means that in the returned error type, we only care about the error, not the success value. Another example: `HashSet` is actually a type alias for `HashMap<K, ()>`.

To briefly summarize: we’ve learned that Rust is a strongly typed / statically typed language, and that in code, types are everywhere.

As a statically typed language, Rust provides a large number of data types, but in practice, type annotations can be cumbersome. So Rust’s type system thoughtfully provides type inference.

Compared to dynamic type systems, one inconvenience of static type systems is that the same algorithm needs different implementations for different input data structures, even when these implementations don’t differ logically. To address this, Rust’s answer is generics (parametric polymorphism).

So next, we’ll first look at the basic data types Rust provides, then see how type inference works, and finally examine how Rust supports generics.


## Data Types

In Lecture 2, we introduced the definitions of primitive types and compound types. Today we will take a closer look at how these two categories of types are designed in Rust.

Rust’s primitive types include characters, integers, floating-point numbers, booleans, arrays, tuples, slices, pointers, references, functions, etc., as shown in the table.

![Rust Primitive Types](images/rust-12-03.webp)

On top of primitive types, the Rust standard library also supports a very rich set of compound types, let’s take a look at what we’ve already encountered:

![Rust compound types](images/rust-12-04.webp)

As we go further, we will constantly encounter new data types. I recommend consciously keeping a record of them — by the end, your list will have grown quite long.

In addition, based on Rust’s existing data types, you can also define your own compound types using struct and enum. We have already covered this in detail earlier, so I won’t repeat it here. You can look at the diagram below for review:

![programming data types](images/rust-02-01.webp)

## Type Inference

As a statically typed language, while it guarantees type safety at compile time, a big inconvenience is that code can become verbose, with type declarations required everywhere. Especially since Rust has so many data types, to reduce the burden on developers, Rust supports local type inference.

**Within a scope, Rust can infer the type of a variable from its usage context**, so we don’t always need explicit type annotations. For example, in this code we create a `BTreeMap`, then insert a key "hello" and value "world":

``` rust
use std::collections::BTreeMap;

fn main() {
    let mut map = BTreeMap::new();
    map.insert("hello", "world");
    println!("map: {:?}", map);
}

```

Here, the Rust compiler can infer from context that the type parameters of `BTreeMap<K, V>` are both string slices `&str`, so this code compiles. However, if you comment out the insert statement in line 5, the compiler will complain: “cannot infer type for type parameter K”.

Clearly, **Rust requires enough context to perform type inference**. In some cases the compiler cannot infer an appropriate type, for example when trying to filter even numbers from a list:
``` rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    let even_numbers = numbers
        .into_iter()
        .filter(|n| n % 2 == 0)
        .collect();

    println!("{:?}", even_numbers);
}

```


`collect` is a method of the [Iterator trait](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect), and it converts an iterator into a collection. Since many collection types, such as `Vec<T>`, `HashMap<K, V>`, etc., all implement Iterator, the compiler cannot infer from the context what type collect should actually return.

Therefore, this piece of code cannot compile. It will give the following error: “consider giving even_numbers a type”.

In this situation, we cannot rely on type inference to simplify the code; we must give `even_number`s` an explicit type. So, we can use a type annotation.

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    let even_numbers: Vec<_> = numbers
        .into_iter()
        .filter(|n| n % 2 == 0)
        .collect();

    println!("{:?}", even_numbers);
}
```

Note that here the compiler cannot infer the collection type, but it can infer the type of the collection’s elements from context. So we can write it as `Vec<_>`.

Besides giving the variable an explicit type, we can also make collect return a concrete type:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    let even_numbers = numbers
        .into_iter()
        .filter(|n| n % 2 == 0)
        .collect::<Vec<_>>();

    println!("{:?}", even_numbers);
}
```


You can see that after a generic function, we use `::<T>` to force the type to T. This syntax is called the turbofish. Let’s look at another example, parsing an IP address and port:

```rust
use std::net::SocketAddr;

fn main() {
    let addr = "127.0.0.1:8080".parse::<SocketAddr>().unwrap();
    println!("addr: {:?}, port: {:?}", addr.ip(), addr.port());
}
```

The turbofish syntax has many advantages. For example, when you want to directly return an expression in a function or match arm:

```rust
match data {
    Some(s) => v.parse::<User>()?,
    _ => return Err(...),
}
```

If the type `User` cannot be inferred from context, without turbofish you would have to first assign to a local variable with a type annotation, then return it — which makes the code redundant.

In some cases, **even when type information exists in context, you must still explicitly annotate constants and static variables**. For example:

```rust
const PI: f64 = 3.1415926;
static E: f32 = 2.71828;

fn main() {
    const V: u32 = 10;
    static V1: &str = "hello";
    println!("PI: {}, E: {}, V {}, V1: {}", PI, E, V, V1);
}

```

This is because `const` and `static` usually define global variables, which can be used across multiple contexts. For readability, explicit type annotations are required.


## Using Generics to Achieve Parametric Polymorphism

That’s it for defining and using types. As mentioned earlier, Rust uses generics to avoid requiring developers to implement separate algorithms for different types. In a static language without generics, coding can be painful. For example, think about `Vec<T>` — without generics, you’d have to implement a separate `Vec<T>` for each type `T`. That’s far too much work.

So let’s look at how Rust supports generics. Today we’ll cover parametric polymorphism, which includes generic data structures and generic functions. In the next lecture, we’ll discuss ad-hoc polymorphism and subtype polymorphism.


## Generic Data Structures

Rust has full support for generic (parameterized) data structures.

Throughout our learning so far, we’ve encountered many parameterized data types. These greatly improve code reuse and reduce redundancy. Almost all modern statically typed languages support parameterized types.

Let’s start with the simplest example, `Option<T>`:

```rust
enum Option<T> {
  Some(T),
  None,
}
```

You should be familiar with this. T can be any type. `Some(T)` represents a value, while `None` represents no value.

When defining this generic data structure, notice how it feels similar to defining a function:
* **A function abstracts parameters out of repeated code, making it reusable**. Calling the function with different arguments yields different results.
* **A generic type abstracts parameters out of repeated data structures**, and instantiating it with different type arguments yields different concrete types.

Now let’s look at a more complex generic structure, `Vec<T>`:

```rust
pub struct Vec<T, A: Allocator = Global> {
    buf: RawVec<T, A>,
    len: usize,
}

pub struct RawVec<T, A: Allocator = Global> {
    ptr: Unique<T>,
    cap: usize,
    alloc: A,
}
```

Vec has two parameters:
* T, the type of each element in the vector.
* A, which must satisfy the Allocator trait.

A has a default value [Global](https://doc.rust-lang.org/std/alloc/struct.Global.html), the default global allocator. This is why we only need to specify T when using `Vec<T>`.

When we discussed lifetime annotations, we saw that if a type contains borrowed data, it must explicitly annotate lifetimes. **In Rust, lifetimes are also part of generics**. A lifetime `'a` is just like `T`, representing “any lifetime.”

Consider the `Cow<T>` enum:

```rust
pub enum Cow<'a, B: ?Sized + 'a> where B: ToOwned,
{
    // borrowed data
    Borrowed(&'a B),
    // owned data
    Owned(<B as ToOwned>::Owned),
}
```


Cow (Clone-on-Write) is a very interesting and important data structure in Rust. It is similar to Option, in that when returning data, it provides a possibility: either return a borrowed piece of data (read-only), or return an owned piece of data (writable).

**For the owned data type B, the first is the lifetime constraint**. Here the lifetime of B is `'a`, so B must satisfy `'a`. This is the same as a generic constraint, written as `B: 'a`. When the internal type B of Cow has lifetime `'a`, Cow itself also has lifetime `'a`.

B also has two other constraints: `?Sized` and where `B: ToOwned`.

When expressing constraints on generic parameters, Rust allows two forms. One is similar to function parameter type declarations, using `:` to indicate constraints, and `+` to combine multiple constraints. The other is to use a `where` clause at the end of the definition to state the parameter constraints. Both methods are valid and can coexist.

`?Sized` is a special constraint syntax. `?` means the constraint following the question mark can be relaxed. Since Rust requires generic parameters to be `Sized` (types with fixed size) by default, here **`?Sized` allows types with variable size**.

[`ToOwned`](https://doc.rust-lang.org/std/borrow/trait.ToOwned.html) is a trait that can clone borrowed data into owned data. 

So the three constraints on B here are: lifetime `'a`, variable size `?Sized`, and implementing the `ToOwned` trait.

In the Cow enum, the meaning of `<B as ToOwned>::Owned` is: it performs a coercion on B to the `ToOwned` trait, and then accesses the `Owned` type defined inside the `ToOwned` trait.

Because in Rust, a subtype can be coerced into its supertype, and since B is constrained by ToOwned, it is a subtype of the ToOwned trait. Therefore, B can be safely coerced into ToOwned. Thus, B as ToOwned is valid here.

In the above `Vec` and `Cow` examples, constraints on generic parameters are placed in the struct or enum definition. However, **we can also add constraints progressively across different implementations**. For example:

```rust
use std::fs::File;
use std::io::{BufReader, Read, Result};

// Define a reader with a generic parameter R, initially with no constraints
struct MyReader<R> {
    reader: R,
    buf: String,
}

// In the new() function, no constraints are needed
impl<R> MyReader<R> {
    pub fn new(reader: R) -> Self {
        Self {
            reader,
            buf: String::with_capacity(1024),
        }
    }
}

// In process(), we require R to implement Read
impl<R> MyReader<R>
where
    R: Read,
{
    pub fn process(&mut self) -> Result<usize> {
        self.reader.read_to_string(&mut self.buf)
    }
}

fn main() {
    // On Windows, use another file path instead
    let f = File::open("/etc/hosts").unwrap();
    let mut reader = MyReader::new(BufReader::new(f));

    let size = reader.process().unwrap();
    println!("total size read: {}", size);
}
```

Adding constraints gradually ensures they only appear where necessary, giving the code maximum flexibility.



## Generic Functions

After understanding how generic data structures are defined and used, let’s look at generic functions—the idea is similar.
**When declaring a function, we don’t have to specify concrete parameter or return types; instead, we can use generic parameters**. For functions, this is a higher level of abstraction.

A simple example:

```rust
fn id<T>(x: T) -> T {
    return x;
}

fn main() {
    let int = id(10);
    let string = id("Tyr");
    println!("{}, {}", int, string);
}
```

Here, `id()` is a generic function. It accepts a parameter with a generic type and returns a generic type.

For generic functions, Rust performs [monomorphization](https://en.wikipedia.org/wiki/Monomorphization), which means that at compile time, it expands all uses of generic functions into concrete types, generating multiple functions. So, the above `id()` will compile into multiple processed versions:

```rust
fn id_i32(x: i32) -> i32 {
    return x;
}
fn id_str(x: &str) -> &str {
    return x;
}
fn main() {
    let int = id_i32(42);
    let string = id_str("Tyr");
    println!("{}, {}", int, string);
}

```

**The benefit of monomorphization is that generic function calls are statically dispatched at compile time**, directly mapped one-to-one. This preserves the flexibility of polymorphism without any performance loss, making them as efficient as ordinary function calls.

But as seen from the expanded code, monomorphization has an obvious downside: compilation is slow.**For each generic function, the compiler must find all the different concrete types used and compile each one individually**. This is one reason why Rust’s compilation speed is often criticized (another important factor is macros).

At the same time, the generated binary can be relatively large, because the binary code for a generic function actually exists in N copies.

Another subtle issue: **because of monomorphization, distributing code as a binary loses the generic information.** If I write a library providing the above `id()` function, and another developer uses the binary version of my library, the original generic function must still exist in the binary to be callable. But after monomorphization, the original generic information is discarded.


## Summary

Today we introduced some basic concepts of the type system and Rust’s type system.

We used a diagram to describe the main features of Rust’s type system, including its attributes, data structures, type inference, and generic programming:

![rust type system](images/rust-12-05.webp)

In terms of type definition, checking, and whether types can be inferred at compile time, Rust is **strongly typed** + **statically typed** + **explicitly typed**.

Since it is statically typed, you need to master the commonly used types firmly when writing code. To avoid the hassle of annotating types everywhere, Rust provides type inference.

In some cases, Rust cannot infer types from context. Then, we must explicitly annotate the type of variables, or use turbofish syntax to provide a concrete type for a generic function. An exception is constants or static variables: even if the type is obvious from the context, Rust still requires explicit type annotations.

For parametric polymorphism, Rust provides comprehensive support for generics. You can use and define **generic data structures**, and when declaring a function, you don’t need to specify exact parameter or return types, since generic parameters can replace them—i.e., **generic functions**. The concepts are similar: once data structures can be generic, functions naturally also need generics.

Additionally, lifetime annotations are also a part of generics. And for generic functions, monomorphization occurs at compile time, which contributes to slow compilation.
