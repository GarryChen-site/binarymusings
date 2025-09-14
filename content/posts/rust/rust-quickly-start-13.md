---
title: "Rust Traits Guide: Polymorphism & Dynamic Dispatch Tutorial"

description: "Master Rust traits for ad-hoc and subtype polymorphism. Learn trait objects, dynamic dispatch, vtables, and object safety in this guide."

summary: "Comprehensive guide to Rust traits covering polymorphism implementation, trait objects for dynamic dispatch, associated types, generic traits, and object safety rules. Learn how to build flexible APIs with practical examples including parser traits, operator overloading, and service patterns."

date: 2025-09-14
series: ["Rust"]
weight: 1
tags: ["rust", "traits", "polymorphism", "dynamic-dispatch", "trait-objects"]
author: ["Garry Chen"]
cover:
  image: images/rust-13-00.webp
  hiddenInList: true
  caption: "Rust Traits Design"

---




Through the [previous article]({{< ref "rust-quickly-start-12.md" >}}), we have gained an understanding of the essence of Rust’s type system.As a tool for defining, checking, and handling types, the type system ensures that the data types used in operations are exactly what we expect.

With Rust’s powerful support for generics, we can conveniently define and use generic data structures and generic functions, and use them to handle parametric polymorphism, making input and output types more flexible and improving code reusability.

Today, we will continue with two other kinds of polymorphism: ad-hoc polymorphism and subtype polymorphism. We will see what problems they can solve, how to implement them, and how to use them.

If you don’t quite remember the definitions of these two polymorphisms, let’s briefly review: Ad-hoc polymorphism includes operator overloading; it refers to one behavior having multiple different implementations. Treating a subtype as a parent type, such as using Cat as Animal, belongs to subtype polymorphism.

In Rust, the implementation of both kinds of polymorphism is related to traits, so we need to first understand what traits are before seeing how to use them to handle these two types of polymorphism.


## What is a trait?

A trait in Rust is an interface that **defines the behaviors a type implementing it must provide**. If you map it to languages you are familiar with: `interface` in Java, `protocol` in Swift, `type class` in Haskell.

When developing complex systems, we often emphasize the separation of interfaces and implementations. This is good design practice: it isolates the caller from the implementer; as long as both follow the interface, internal changes on one side won’t affect the other.

A trait works like this: it extracts behaviors from data structures so that they can be shared among multiple types. It can also be used as a constraint in generic programming, requiring parameterized types to satisfy certain behaviors.


## Basic traits

Let’s look at how a basic trait is defined. Here is [std::io::Write](https://doc.rust-lang.org/std/io/trait.Write.html) from the standard library as an example. You can see that this trait defines a set of method interfaces:

```rust
pub trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;
    fn write_vectored(&mut self, bufs: &[IoSlice<'_>]) -> Result<usize> { ... }
    fn is_write_vectored(&self) -> bool { ... }
    fn write_all(&mut self, buf: &[u8]) -> Result<()> { ... }
    fn write_all_vectored(&mut self, bufs: &mut [IoSlice<'_>]) -> Result<()> { ... }
    fn write_fmt(&mut self, fmt: Arguments<'_>) -> Result<()> { ... }
    fn by_ref(&mut self) -> &mut Self where Self: Sized { ... }
}
```

These methods are also called associated functions. **In a trait, methods can have default implementations**. For this Write trait, you only need to implement write and flush; all others have default implementations.

If you compare traits to a parent class, and types implementing traits to subclasses, then default methods are like methods in a parent class that can be overridden but don’t have to be.


When defining methods above, we saw two special keywords frequently: `Self` and `self`.
* `Self` represents the current type. For example, if `File` implements `Write`, then `Self` refers to `File` inside the implementation.
* `self` as the first parameter is shorthand for `self: Self`. So `&self` = `self: &Self`, `&mut self` = `self: &mut Self`.


Just talking about definitions is hard to fully grasp, so let’s build a BufBuilder structure that implements the Write trait, and explain through code:

```rust
use std::fmt;
use std::io::Write;

struct BufBuilder {
    buf: Vec<u8>,
}

impl BufBuilder {
    pub fn new() -> Self {
        Self {
            buf: Vec::with_capacity(1024),
        }
    }
}

// Implement Debug trait to print strings
impl fmt::Debug for BufBuilder {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}", String::from_utf8_lossy(&self.buf))
    }
}

impl Write for BufBuilder {
    fn write(&mut self, buf: &[u8]) -> std::io::Result<usize> {
        // Append buf to the end of BufBuilder
        self.buf.extend_from_slice(buf);
        Ok(buf.len())
    }

    fn flush(&mut self) -> std::io::Result<()> {
        // No need to flush because we operate in memory
        Ok(())
    }
}

fn main() {
    let mut buf = BufBuilder::new();
    buf.write_all(b"Hello world!").unwrap();
    println!("{:?}", buf);
}
```

From the code, we can see: we implemented `write` and `flush`. Other methods use the default implementation. Thus, `BufBuilder` fully implements the `Write` trait. If we skip `write` or `flush`, the Rust compiler will give an error—you can try this yourself.

Once a data structure implements a trait, all methods inside the trait can be used. For example, here we called `buf.write_all()`.

How was `write_all()` called? Let’s look at its signature:

```rust
fn write_all(&mut self, buf: &[u8]) -> Result<()>
```

It takes two parameters: `&mut self` — the mutable reference to buf, and `&[u8]` — here it’s b"Hello world!"


## Basic trait exercise

Okay, after understanding the basic definition and usage of traits, let’s try defining a trait to reinforce it.

Suppose we want to create a string parser that can parse a part of a string into a certain type, then we can define the trait like this: it has a method called `parse`, which takes a string reference and returns `Self`.

```rust
pub trait Parse {
  fn parse(s: &str) -> Self;
}
```

This `parse` method is a static method of the trait because its first parameter has nothing to do with self, so when calling it, we need to use `T::parse(str)`.

Let’s try implementing `parse` for the `u8` data structure, for example: “123abc” will be parsed to the integer 123, while “abcd” will be parsed to 0.

To achieve this, we need to introduce a new library [Regex](https://github.com/rust-lang/regex) to extract the required content using regular expressions. In addition, we also need to use the `str::parse` function to convert a string containing numbers into numbers.

The complete code is as follows

```rust
use regex::Regex;
pub trait Parse {
    fn parse(s: &str) -> Self;
}

impl Parse for u8 {
    fn parse(s: &str) -> Self {
        let re: Regex = Regex::new(r"^[0-9]+").unwrap();
        if let Some(captures) = re.captures(s) {
            // Take the first match and convert the captured digits into u8
            captures
                .get(0)
                .map_or(0, |s| s.as_str().parse().unwrap_or(0))
        } else {
            0
        }
    }
}

#[test]
fn parse_should_work() {
    assert_eq!(u8::parse("123abcd"), 123);
    assert_eq!(u8::parse("1234abcd"), 0);
    assert_eq!(u8::parse("abcd"), 0);
}

fn main() {
    println!("result: {}", u8::parse("255 hello world"));
}
```

This implementation is not difficult. If you are interested, you can also try implementing this Parse trait for `f64`, for example, “123.45abcd” needs to be parsed to 123.45.

While implementing `f64`, do you feel that except for the type and the regex used for capturing, the whole code is basically duplicated from the above code? As developers, we want Don’t Repeat Yourself (DRY), so writing such code feels awkward and uncomfortable. Is there a better way?

Yes! The previous lecture introduced generic programming, so **when implementing a trait, we can also use generic parameters**, but note that we need to impose certain constraints on the generic parameters.

* First, not all types can be parsed from strings. In the example, we can only handle numeric types, and the type must also be able to be handled by `str::parse`.

Specifically, `str::parse` is a generic function that returns any type implementing the `FromStr` trait, **so the first constraint on the generic parameter here is that it must implement the `FromStr` trait**.

* Second, when the above code fails to parse the string correctly, it directly returns 0 to indicate failure. But after using generic parameters, we can’t return 0, because 0 might not be a value of the type satisfying the generic parameter. What to do?

Actually, the purpose of returning 0 is to return a default value when processing fails. In the Rust standard library, there is a `Default` trait, and most types implement this trait to provide default values for data structures. So **another constraint for the generic parameter is `Default`**.

Okay, with the basic idea in mind, let’s look at the code

```rust
use std::str::FromStr;

use regex::Regex;
pub trait Parse {
    fn parse(s: &str) -> Self;
}

// We constrain T to implement both FromStr and Default
// So that we can use methods of these two traits
impl<T> Parse for T
where
    T: FromStr + Default,
{
    fn parse(s: &str) -> Self {
        let re: Regex = Regex::new(r"^[0-9]+(\.[0-9]+)?").unwrap();
        // Generate a closure to create a default value, mainly to simplify subsequent code
        // Default::default() returns a type inferred as Self from the context
        // And we have constrained Self (i.e., T) to implement the Default trait
        let d = || Default::default();
        if let Some(captures) = re.captures(s) {
            captures
                .get(0)
                .map_or(d(), |s| s.as_str().parse().unwrap_or(d()))
        } else {
            d()
        }
    }
}

#[test]
fn parse_should_work() {
    assert_eq!(u32::parse("123abcd"), 123);
    assert_eq!(u32::parse("123.45abcd"), 0);
    assert_eq!(f64::parse("123.45abcd"), 123.45);
    assert_eq!(f64::parse("abcd"), 0f64);
}

fn main() {
    println!("result: {}", u8::parse("255 hello world"));
}
```

By implementing a trait with constrained generic parameters, we have one piece of code implementing the Parse trait for `u32` / `f64` and other types, very concise. However, do you feel there’s still a problem with this code? When the string cannot be parsed correctly, we return a default value, but shouldn’t we return an error instead?

Yes. **Returning a default value here confuses cases like parsing “0abcd”—we can’t tell whether the parsed 0 means failure or if it was indeed supposed to be 0**.

So a better approach is for the parse function to return a `Result<T, E>`:

```rust
pub trait Parse {
    fn parse(s: &str) -> Result<Self, E>;
}
```

But the `E` in `Result` is tricky: the error type to be returned is not determined when the trait is defined. Different implementers can use different error types. It’s best if the trait’s author can leave this flexibility to the implementers. What to do?

Since traits allow internal methods (associated functions), can we further include associated types? The answer is yes.

## Traits with associated types

Rust allows traits to contain associated types internally. When implementing them, like associated functions, we also need to implement the associated types. Let’s see how to add an associated type to the `Parse` trait:

```rust
pub trait Parse {
    type Error;
    fn parse(s: &str) -> Result<Self, Self::Error>;
}
```

With the associated type `Error`, the `Parse` trait can now return reasonable errors when failures occur. Here’s the modified code:

```rust
use std::str::FromStr;

use regex::Regex;
pub trait Parse {
    type Error;
    fn parse(s: &str) -> Result<Self, Self::Error>
    where
        Self: Sized;
}

impl<T> Parse for T
where
    T: FromStr + Default,
{
    // Define the associated type Error as String
    type Error = String;
    fn parse(s: &str) -> Result<Self, Self::Error> {
        let re: Regex = Regex::new(r"^[0-9]+(\.[0-9]+)?").unwrap();
        if let Some(captures) = re.captures(s) {
            // When an error occurs we return Err(String)
            captures
                .get(0)
                .map_or(Err("failed to capture".to_string()), |s| {
                    s.as_str()
                        .parse()
                        .map_err(|_err| "failed to parse captured string".to_string())
                })
        } else {
            Err("failed to parse string".to_string())
        }
    }
}

#[test]
fn parse_should_work() {
    assert_eq!(u32::parse("123abcd"), Ok(123));
    assert_eq!(
        u32::parse("123.45abcd"),
        Err("failed to parse captured string".into())
    );
    assert_eq!(f64::parse("123.45abcd"), Ok(123.45));
    assert!(f64::parse("abcd").is_err());
}

fn main() {
    println!("result: {:?}", u8::parse("255 hello world"));
}
```

In the above code, we allow users to defer deciding the error type until implementing the trait. Traits with associated types are more flexible and more abstract than ordinary traits.

Parameters or return values in trait methods can all be expressed with associated types, and when implementing traits with associated types, you only need to provide the specific type for the associated type.


## Traits Supporting Generics

So far, we have step by step learned about the definition and usage of basic traits, as well as more complex and flexible traits with associated types. Combining this with generics introduced in the previous lecture, you might wonder: can trait definitions themselves also support generics?

For example, suppose we want to define a `Concat` trait that allows data structures to be concatenated together. Naturally, we hope that `String` can be concatenated with another `String`, with `&str`, or even with any data structure convertible to `String`. At this point, we need traits themselves to support generics.

Let’s look at how operators are overloaded in the standard library. Take the `std::ops::Add` trait for example, which provides addition:

```rust
pub trait Add<Rhs = Self> {
    type Output;
    #[must_use]
    fn add(self, rhs: Rhs) -> Self::Output;
}
```

This trait has a generic parameter `Rhs` representing the right-hand operand in addition, used as the second parameter of `add`. Here `Rhs` defaults to `Self`, meaning that if you use the `Add` trait without providing a generic parameter, the right operand and left operand must be of the same type.

Let’s define a complex number type and try using this trait:

```rust
use std::ops::Add;

#[derive(Debug)]
struct Complex {
    real: f64,
    imagine: f64,
}

impl Complex {
    pub fn new(real: f64, imagine: f64) -> Self {
        Self { real, imagine }
    }
}

// Implementation for Complex
impl Add for Complex {
    type Output = Self;

    // Note that the first parameter of add is self, which will move the ownership.
    fn add(self, rhs: Self) -> Self::Output {
        let real = self.real + rhs.real;
        let imagine = self.imagine + rhs.imagine;
        Self::new(real, imagine)
    }
}

fn main() {
    let c1 = Complex::new(1.0, 1f64);
    let c2 = Complex::new(2 as f64, 3.0);
    println!("{:?}", c1 + c2);

    // This line would cause a compile error because c1 and c2 have been moved
    // println!("{:?}", c1 + c2); 
}
```

Complex numbers have real and imaginary parts; adding two complex numbers means adding their real parts and imaginary parts respectively to get a new complex number. Notice that `add` takes `self` as its first parameter, moving ownership. So after calling `c1 + c2`, by ownership rules, `c1` and `c2` can no longer be used.

This makes `Add` convenient for types implementing `Copy` like `u32` or `f64`, but for `Complex`, after one addition the original values are gone. Can we implement `Add` for references to `Complex` instead?

Yes, we can:

```rust
// If you don’t want to move ownership, you can implement add for &Complex, so that you can do &c1 + &c2. 
impl Add for &Complex {
    // Note that the return value should no longer be Self, because at this point Self is &Complex.
    type Output = Complex;

    fn add(self, rhs: Self) -> Self::Output {
        let real = self.real + rhs.real;
        let imagine = self.imagine + rhs.imagine;
        Complex::new(real, imagine)
    }
}

fn main() {
    let c1 = Complex::new(1.0, 1f64);
    let c2 = Complex::new(2 as f64, 3.0);
    println!("{:?}", &c1 + &c2);
    println!("{:?}", c1 + c2);
}

```

Now we can do `&c1 + &c2` without moving ownership.

So far, we’ve only used default generic parameters. You might ask, what’s the point of generics then?

Here’s a practical example: now we want to add a complex number to a real number so that only the real part changes while the imaginary part stays the same:

```rust
impl Add<f64> for &Complex {
    type Output = Complex;

    fn add(self, rhs: f64) -> Self::Output {
        let real = self.real + rhs;
        Complex::new(real, self.imagine)
    }
}

fn main() {
    let c1 = Complex::new(1.0, 1f64);
    let c2 = Complex::new(2 as f64, 3.0);
    println!("{:?}", &c1 + &c2);
    println!("{:?}", &c1 + 5.0);
    println!("{:?}", c1 + c2);
}
```
By using `Add`, we implemented addition of `Complex` and `f64`. **So generic traits allow multiple implementations of the same trait for the same type when needed**.


This small example is not very practical, so let’s look at a generic trait that might actually be used in real-world work, and you’ll see how powerful this feature really is.

[tower::Service](https://docs.rs/tower/0.4.8/tower/trait.Service.html) is a third-party library that defines a sophisticated, classic trait for handling requests and returning responses. It’s used in many well-known third-party networking libraries, such as [tonic](https://docs.rs/tonic/0.5.2/tonic/) for handling gRPC.

Look at the definition of Service:

```rust
// The Service trait allows an implementation of a service to handle multiple different Requests
pub trait Service<Request> {
    type Response;
    type Error;
    // The Future type is constrained by the Future trait
    type Future: Future;
    fn poll_ready(
        &mut self, 
        cx: &mut Context<'_>
    ) -> Poll<Result<(), Self::Error>>;
    fn call(&mut self, req: Request) -> Self::Future;
}
```

This trait allows a `Service` to handle multiple different `Requests`. When we use this trait in web development, each Method+URL can be defined as a Service, with its Request being the input type.

Note that for a specific Request type, there will only be one kind of Response, so here Response uses an associated type rather than a generic. If it were possible to return multiple Responses, then you should use a generic `Service<Request, Response>`.

![trait in web](images/rust-13-01.webp)

## Trait “Inheritance”

In Rust, one trait can “inherit” the associated types and associated functions of another trait. For example, `trait B: A` means any type T that implements trait B must also implement trait A. In other words, **trait B can use the associated types and methods of trait A when defined**.

This “inheritance” is very helpful for extending trait capabilities. Many common traits use trait inheritance to provide additional abilities, such as [AsyncWriteExt](https://docs.rs/tokio/1.10.0/tokio/io/trait.AsyncWriteExt.html) in the tokio library and [StreamExt](https://docs.rs/futures/0.3.16/futures/stream/trait.StreamExt.html) in the futures library.

Take `StreamExt` as an example: because all the methods in `StreamExt` have default implementations, and all types that implement the `Stream` trait also implement `StreamExt`:

```rust
impl<T: ?Sized> StreamExt for T where T: Stream {}
```

So if you implement the `Stream` trait, you can directly use the methods in `StreamExt`, which is very convenient.

Alright, at this point, traits are basically covered. Let’s do a simple summary, a trait is an abstraction of the same behavior across different data structures. Besides basic traits, 
* when behavior is associated with specific data — such as the `Parse` trait we defined for string parsing — we introduced traits with associated types, deferring the definition of data types related to the behavior until the trait implementation.
* For the same type and the same trait behavior, there can be different implementations. For example, the `From` trait we’ve used extensively before — here we can use generic traits.

You can say that Rust’s traits are like a Swiss Army knife, covering all kinds of scenarios where you need to define interfaces.

Ad-hoc polymorphism is different implementations of the same behavior. So in fact, **by defining traits and implementing them for different types, we have already achieved ad-hoc polymorphism**.

The `Add` trait we discussed earlier is a classic example of ad-hoc polymorphism: the same addition operation is handled differently depending on the data involved. The `Service` trait is a less obvious example of ad-hoc polymorphism: the same web request behavior is handled with different code depending on the URL.


## How to do subtype polymorphism?

Strictly speaking, subtype polymorphism is the domain of object-oriented languages. **If an object A is a subclass of object B, then an instance of A can appear in any context where an instance of B is expected**. For example, cats and dogs are both animals; if a function interface requires an animal to be passed in, passing in either a cat or a dog is allowed.

Although Rust has no parent and child classes, the relationship between a trait and the types that implement the trait is similar, so Rust can also do subtype polymorphism. Look at an example:

```rust
struct Cat;
struct Dog;

trait Animal {
    fn name(&self) -> &'static str;
}

impl Animal for Cat {
    fn name(&self) -> &'static str {
        "Cat"
    }
}

impl Animal for Dog {
    fn name(&self) -> &'static str {
        "Dog"
    }
}

fn name(animal: impl Animal) -> &'static str {
    animal.name()
}

fn main() {
    let cat = Cat;
    println!("cat: {}", name(cat));
}
```

Here `impl Animal` is shorthand for `T: Animal`, so the definition of the name function is equivalent to:

```rust
fn name<T: Animal>(animal: T) -> &'static str;
```

As mentioned in the previous article, such generic functions will be monomorphized based on the specific types used, compiled into multiple instances — this is static dispatch.

Static dispatch is fine and efficient, but many times, the type may be hard to decide at compile time. For example, to write a formatting tool — which is common in IDEs — we can define a `Formatter` interface, then create a series of implementations:

```rust
pub trait Formatter {
    fn format(&self, input: &mut String) -> bool;
}

struct MarkdownFormatter;
impl Formatter for MarkdownFormatter {
    fn format(&self, input: &mut String) -> bool {
        input.push_str("\nformatted with Markdown formatter");
        true
    }
}

struct RustFormatter;
impl Formatter for RustFormatter {
    fn format(&self, input: &mut String) -> bool {
        input.push_str("\nformatted with Rust formatter");
        true
    }
}

struct HtmlFormatter;
impl Formatter for HtmlFormatter {
    fn format(&self, input: &mut String) -> bool {
        input.push_str("\nformatted with HTML formatter");
        true
    }
}
```

First, which formatting method to use can only be determined after opening the file and analyzing the file contents, so we can’t give a specific type at compile time. Second, a file might have one or more formatting tools. For example, a Markdown file with Rust code inside might need both `MarkdownFormatter` and `RustFormatter`.

Here, if we use a `Vec<T>` to provide all the formatting tools needed, how should the type of the `formatters` parameter be determined in the following function?

```rust
pub fn format(input: &mut String, formatters: Vec<???>) {
    for formatter in formatters {
        formatter.format(input);
    }
}
```

Normally, the type inside a `Vec<>` container needs to be consistent, but here we can’t give a single consistent type.

So we need a way to tell the compiler: here, we need and only need any data type that implements the `Formatter` interface. In Rust, this type is called a Trait Object, represented as `&dyn Formatter` or `Box<dyn Formatter>`.

Here, the `dyn` keyword is only to help us better distinguish between ordinary types and trait types. When reading code, seeing `dyn` tells us what follows is a trait.

So the above code can be written as:

```rust
pub fn format(input: &mut String, formatters: Vec<&dyn Formatter>) {
    for formatter in formatters {
        formatter.format(input);
    }
}
```

This allows us to construct a list of `Formatters` at runtime and pass it to the `format` function for file formatting — this is dynamic dispatch.

Look at the final code calling the formatting tools:

```rust
pub trait Formatter {
    fn format(&self, input: &mut String) -> bool;
}

struct MarkdownFormatter;
impl Formatter for MarkdownFormatter {
    fn format(&self, input: &mut String) -> bool {
        input.push_str("\nformatted with Markdown formatter");
        true
    }
}

struct RustFormatter;
impl Formatter for RustFormatter {
    fn format(&self, input: &mut String) -> bool {
        input.push_str("\nformatted with Rust formatter");
        true
    }
}

struct HtmlFormatter;
impl Formatter for HtmlFormatter {
    fn format(&self, input: &mut String) -> bool {
        input.push_str("\nformatted with HTML formatter");
        true
    }
}

pub fn format(input: &mut String, formatters: Vec<&dyn Formatter>) {
    for formatter in formatters {
        formatter.format(input);
    }
}

fn main() {
    let mut text = "Hello world!".to_string();
    let html: &dyn Formatter = &HtmlFormatter;
    let rust: &dyn Formatter = &RustFormatter;
    let formatters = vec![html, rust];
    format(&mut text, formatters);

    println!("text: {}", text);
}
```

This implementation is simple, right? Having learned this much, you might feel a bit burdened because yet another Rust term appears. Don’t worry — although Trait Objects are unique to Rust, the concept itself is not new. Why? Let’s look at its mechanism.


## The mechanism of Trait Objects

When you need to use the Formatter trait for dynamic dispatch, you can assign a reference of a concrete type to `&Formatter` as in the example below:

![underlying logic of a Trait Object](images/rust-13-02.webp)

After assigning the reference of `HtmlFormatter` to `Formatter`, a Trait Object will be generated. In the diagram above, you can see that **the underlying logic of a Trait Object is a fat pointer**. One pointer points to the data itself, the other points to the virtual function table (vtable).

The vtable is a static table. Rust will generate a table for the trait implementation of the type that uses the trait object at compile time and put it in the executable file (generally in the TEXT or RODATA section). Looking at the diagram will help you understand:

![vtable visual](images/rust-13-03.webp)

In this table, there is information about the concrete type, such as:
* size,
* alignment,
* a series of function pointers:
* all the methods supported by this interface, such as `format()`,
* the `drop` trait for the concrete type, which is used to release all resources when the Trait Object is dropped.

So when at runtime we execute `formatter.format()`, the formatter can find the corresponding function pointer from the vtable and execute the concrete operation.

Therefore, **there is nothing mysterious about Rust’s Trait Objects; they are just a variant of the familiar vtable in C++/Java**. 

Here’s a side note: in C++/Java, the pointer to the vtable is placed in the class structure at compile time, whereas in Rust, it’s placed in the Trait Object. This is why Rust can easily perform dynamic dispatch on primitive types, while C++/Java cannot.

In fact, Rust doesn’t distinguish between primitive types and composite types — all types are equal in Rust.

However, when you use trait objects, you need to pay attention to object safety. Only object-safe traits can be used as trait objects, and the official documentation discusses this in detail.

So what kind of trait is not object-safe?

**If all the methods in a trait return Self or carry generic parameters, then the trait cannot produce a trait object**.

Returning `Self` is not allowed because when a trait object is created, the original type is erased, so we don’t know who `Self` actually is. For example, the `Clone` trait has only one method `clone()`, which returns `Self`, so it cannot produce a trait object.

Carrying generic parameters is not allowed because in Rust, types with generics are monomorphized at compile time, while trait objects are runtime constructs — the two cannot be combined.

For example, the `From` trait, because the entire trait carries generics, and naturally each method contains generics, cannot produce trait objects. If a trait has only some methods returning `Self` or using generic parameters, then those methods cannot be called on a trait object.


## Summary
Today we gave a complete introduction to how traits are defined and used, including the most basic traits, traits with associated types, and generic traits. We also reviewed doing static dispatch via traits and using trait objects for dynamic dispatch.

Today’s content is quite a lot. You can also use the diagram below to review the main points of this lesson:
![summary of traits](images/rust-13-04.webp)

Traits, as an abstraction over the same behavior across different data structures, allow us **during development to first determine the system’s behavior based on user needs, abstract these behaviors into traits, and then gradually decide what data structures to use and how to implement these traits for those data structures**.

So, traits are the core element when you do Rust development. When to use which trait depends on the requirements.

But requirements are often not so clear, especially because we need to translate user needs into system design needs. This translation ability relies on enough source code reading, thinking, and rich experience accumulated bit by bit. **Because no matter how powerful Rust’s traits are, they are only like a Swiss Army knife; what makes it fully effective is the person holding it**.

For example, previous articles, we used traits to decouple the system and enhance its extensibility. You can briefly review it. For instance, in this [article]({{< ref "rust-quickly-start-05.md" >}}), the Engine trait and SpecTransform trait used ordinary traits:

``` rust

// Engine trait: We can add more engines in the future, and only need to replace the engine in the main process
pub trait Engine {
    // Apply a series of ordered processing steps to the engine according to specs
    fn apply(&mut self, specs: &[Spec]);
    // Generate the target image from the engine, note that we use self here, not a reference to self
    fn generate(self, format: ImageOutputFormat) -> Vec<u8>;
}

// SpecTransform: If we add more specs in the future, we only need to implement this trait
pub trait SpecTransform<T> {
    // Apply the transform to the image using the op
    fn transform(&mut self, op: T);
}
```

This [article]({{< ref "rust-quickly-start-06.md" >}}) `Fetch`/`Load` trait used traits with associated types:

``` rust

// Rust's async trait is not yet stable, so we can use the async_trait macro
#[async_trait]
pub trait Fetch {
    type Error;
    async fn fetch(&self) -> Result<String, Self::Error>;
}

pub trait Load {
    type Error;
    fn load(self) -> Result<DataSet, Self::Error>;
}
```


