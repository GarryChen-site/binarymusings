---
title: "Rust Web Scraper Tutorial: HTTP Requests & Markdown Conversion"

description: "Build your first Rust project: a web scraper that fetches HTML content and converts it to Markdown. Learn Rust basics through hands-on coding."

summary: "Get hands dirty with Rust by building a practical web scraper from scratch. Learn to make HTTP requests with reqwest, convert HTML to Markdown, and handle file operations. This beginner-friendly tutorial covers essential Rust syntax, project setup with Cargo, and real-world application development while building a useful tool."

date: 2025-07-28
series: ["Rust"]
weight: 1
tags: ["rust", "web-scraper", "http-requests", "rust-beginner", "hands-on-tutorial"]
author: ["Garry Chen"]
cover:
  image: images/rust-03-00.webp
  hiddenInList: true
  caption: ""

---

The best shortcut to learning a language is to immerse yourself in its environment. As programmers, we value "getting our hands dirty." Learning directly from code provides the most intuitive experience.

Today, we'll cover many basic Rust concepts. I've carefully crafted code examples to help you understand. **I highly recommend typing these examples yourself, thinking about why they're written this way, and experiencing the execution and output process**.

You can use any editor to write Rust code. Personally, I prefer VS Code because it's free, powerful, and fast. Here are the plugins I installed for Rust in VS Code, which you can follow:
- **rust-analyzer**: This compiles and analyzes your Rust code in real-time, highlighting errors and annotating types. You can also use the official Rust plugin as an alternative.
- **crates**: Helps you check if your project's dependencies are up to date.
- **even better toml**: Rust uses TOML for project configuration. This plugin provides syntax highlighting and error checking for TOML files.
- **rust test lens**: Allows you to quickly run specific Rust tests.

## Your First Practical Rust Program

Now that you have the tools and environment ready, let's write a slightly useful Rust program. Even though we haven't introduced any Rust syntax yet, it won't stop us from writing a program. Running it will give you a basic understanding of Rust's features, key syntax, and ecosystem, which we'll analyze in detail later.

This program's requirement is simple: send an HTTP request to the Rust homepage, convert the obtained HTML to Markdown, and save it. Using JavaScript or Python, this might be around ten lines of code with the right dependencies. Let's see how to do it in Rust.

First, generate a new project with `cargo new scrape_url`. By default, this command creates an executable project `scrape_url` with the entry point at `src/main.rs`. In the `Cargo.toml` file, add the following dependencies:

```toml
[dependencies]
reqwest = {version = "*", features = ["blocking"]}
html2md = "*"
```

`Cargo.toml` is Rust's project configuration file, following the TOML syntax. We've added two dependencies: `reqwest` and `html2md`. `reqwest` is an HTTP client similar to Python's `requests`; `html2md` converts HTML text to Markdown.

Next, in `src/main.rs`, add the following code to the `main()` function:

```rust
use std::fs;

fn main() {
  let url = "https://www.rust-lang.org/";
  let output = "rust.md";
  
  println!("Fetching url: {}", url);
  let body = reqwest::blocking::get(url).unwrap().text().unwrap();

  println!("Converting html to markdown...");
  let md = html2md::parse_html(&body);

  fs::write(output, md.as_bytes()).unwrap();
  println!("Converted markdown has been saved in {}.", output);
}
```

Save the file, navigate to the project directory in the command line, and run `cargo run`. After a slightly lengthy compilation, the program will run, and you'll see the following output in the command line:

```
Fetching url: https://www.rust-lang.org/
Converting html to markdown...
Converted markdown has been saved in rust.md.
```

Additionally, a `rust.md` file will be created in the current directory. Opening it reveals the content of the Rust homepage.

Bingo! Your first Rust program has successfully run!

From this not-so-long piece of code, you can sense some basic Rust characteristics: 

First, Rust uses a tool called `cargo` to manage projects, similar to Node.js's `npm` or Golang's `go module`. It handles dependency management and development tasks like compiling, running, testing, and code formatting.

Secondly, Rust's overall syntax is reminiscent of C/C++. Functions are enclosed in curly braces `{}`, statements are separated by semicolons `;`, and member functions or variables of structures are accessed using the dot `.` operator. Namespaces or static functions of objects are accessed using the double colon `::` operator. To simplify references to functions or data types within a namespace, you can use the `use` keyword, such as `use std::fs`. Additionally, the entry point of an executable is the `main()` function.

Moreover, you'll notice that although **Rust is a strongly-typed language, the compiler supports type inference**, making the coding experience feel similar to scripting languages.

![the type of the variable will be hinted automatically](images/rust-03-01.webp)

Finally, Rust supports macro programming, with many fundamental features like `println!()` encapsulated as macros, making it easier for developers to write concise code.

Although the example didn't show it, Rust also has the following characteristics:

- Variables in Rust are immutable by default. If you want to modify a variable's value, you need to explicitly use the `mut` keyword.
- Aside from a few statements like `let`, `static`, `const`, and `fn`, most of Rust's code consists of expressions. This means `if`, `while`, `for`, and `loop` all return a value, and the last expression in a function is its return value, similar to functional programming languages.
- Rust supports interface-oriented programming and generic programming.
- Rust has a rich set of data types and a powerful standard library.
- Rust provides extensive control flow mechanisms, including pattern matching.

Next, to quickly get started with Rust, let's go over the basics of Rust development.

## Basic Syntax and Fundamental Data Types

First, let's see how to define variables, functions, and data structures in Rust.

### Variables and Functions

As mentioned earlier, Rust supports type inference. When the compiler can infer the type, the variable type can generally be omitted, but constants (`const`) and static variables (`static`) must have their types declared.

When defining variables, you can add the `mut` keyword to make a variable mutable. **The default immutability of variables** is an important feature that adheres to the Principle of Least Privilege, helping us write robust and correct code. If you use `mut` but do not modify the variable, the Rust compiler will kindly warn you to remove the unnecessary `mut`.

In Rust, functions are first-class citizens and can be passed as parameters or returned as values. Here's an example where a function is used as a parameter:

```rust
fn apply(value: i32, f: fn(i32) -> i32) -> i32 {
  f(value)
}

fn square(value: i32) -> i32 {
  value * value
}

fn cube(value: i32) -> i32 {
  value * value * value
}

fn main() {
  println!("Square of 2: {}", apply(2, square));
  println!("Cube of 2: {}", apply(2, cube));
}
```

Here, `fn(i32) -> i32` is the type of the second parameter of the `apply` function. It indicates that the function takes another function as a parameter, which must take one `i32` argument and return an `i32`.

Both the argument types and return types of Rust functions must be explicitly defined. If there is no return value, it can be omitted, and it returns `unit`. If you need to return early from a function, use the `return` keyword; otherwise, the last expression in the function is its return value. If you add `;` after the last expression, it implicitly returns `unit`.

```rust
fn pi() -> f64 {
  3.1415926
}

fn not_pi() {
  3.1415926;
}

fn main() {
  let is_pi = pi();
  let is_unit1 = not_pi();
  let is_unit2 = {
    pi();
  };
  
  println!("is_pi: {:?}, is_unit1: {:?}, is_unit2: {:?}", is_pi, is_unit1, is_unit2);
}
```

### Data Structures

Now that we know how to define functions, let's see how to define data structures in Rust.

Data structures are the core components of a program. When modeling complex problems, we need to define custom data structures. Rust is very powerful, allowing you to define structures with `struct`, tagged unions with `enum`, and tuple types like in Python.

For example, we can define data structures for a chat service as follows:

```rust
#[derive(Debug)]
enum Gender {
  Unspecified = 0,
  Female = 1,
  Male = 2,
}

#[derive(Debug, Copy, Clone)]
struct UserId(u64);

#[derive(Debug, Copy, Clone)]
struct TopicId(u64);

#[derive(Debug)]
struct User {
  id: UserId,
  name: String,
  gender: Gender,
}

#[derive(Debug)]
struct Topic {
  id: TopicId,
  name: String,
  owner: UserId,
}


#[derive(Debug)]
enum Event {
  Join((UserId, TopicId)),
  Leave((UserId, TopicId)),
  Message((UserId, TopicId, String)),
}

fn main() {
    let alice = User { id: UserId(1), name: "Alice".into(), gender: Gender::Female };
    let bob = User { id: UserId(2), name: "Bob".into(), gender: Gender::Male };
    
    let topic = Topic { id: TopicId(1), name: "rust".into(), owner: UserId(1) };
    let event1 = Event::Join((alice.id, topic.id));
    let event2 = Event::Join((bob.id, topic.id));
    let event3 = Event::Message((alice.id, topic.id, "Hello world!".into()));
    
    println!("event1: {:?}, event2: {:?}, event3: {:?}", event1, event2, event3);
}
```

A brief explanation:

- `Gender`: An enumeration type. In Rust, `enum` can define C-like enums.
- `UserId`/`TopicId`: Special forms of `struct` called tuple structs. Their fields are anonymous and can be accessed by index, suitable for simple structures.
- `User`/`Topic`: Standard structs that can combine any types.
- `Event`: A standard tagged union that defines three kinds of events: `Join`, `Leave`, and `Message`. Each event has its own data structure.

When defining data structures, we usually add traits to introduce additional behavior. In Rust, the behavior of data is defined through traits, which we'll cover in detail later. For now, you can think of traits as defining the interfaces that data structures can implement, similar to interfaces in Java.

We generally use the `impl` keyword to implement traits for data structures. However, Rust provides derive macros to greatly simplify the definition of some standard interfaces. For example, `#[derive(Debug)]` implements the `Debug` trait for a data structure, providing debugging capabilities so it can be printed with `println!` using `{:?}`.

When defining `UserId`/`TopicId`, we also used the `Copy` and `Clone` derive macros. `Clone` allows a data structure to be copied, while `Copy` allows the data structure to be automatically byte-copied when passed as a parameter.

A Simple Overview of Rust for Defining Variables, Functions, and Data Structures

![basic syntax](images/rust-03-02.webp)

## Control Flow

The basic control flows in a program are as follows, and we should already be familiar with them. Let's see how to use them in Rust.

**Sequential execution** is just executing the code line by line. During execution, when a function is encountered, a function call occurs. In a **function call**, code execution jumps to the function's context and continues until the function returns.

Rust's loops are similar to those in most languages, supporting infinite loops with `loop`, conditional loops with `while`, and iterator loops with `for`. Loops can be terminated early with `break` or skip to the next iteration with `continue`.

When certain conditions are met, there will be jumps. Rust supports conditional jumps, pattern matching, error jumps, and asynchronous jumps.

- **Conditional jumps**: the familiar `if/else`.
- **Pattern matching**: branching based on matching expressions or part of a value's content.
- **Error jumps**: Rust can terminate the current function early and return an error to the higher-level function when a called function returns an error.
- **Asynchronous jumps**: when an `async` function encounters `await`, the current context may be blocked, and execution may jump to another asynchronous task until `await` is no longer blocking.

Using the Fibonacci sequence as an example, we can demonstrate the basic control flows with `if`, `loop`, `while`, and `for`.

```rust
fn fib_loop(n: u8) {
    let mut a = 1;
    let mut b = 1;
    let mut i = 2u8;
    
    loop {
        let c = a + b;
        a = b;
        b = c;
        i += 1;
        
        println!("next val is {}", b);
        
        if i >= n {
            break;
        }
    }
}

fn fib_while(n: u8) {
    let (mut a, mut b, mut i) = (1, 1, 2);
    
    while i < n {
        let c = a + b;
        a = b;
        b = c;
        i += 1;
        
        println!("next val is {}", b);
    }
}

fn fib_for(n: u8) {
    let (mut a, mut b) = (1, 1);
    
    for _i in 2..n {
        let c = a + b;
        a = b;
        b = c;
        println!("next val is {}", b);
    }
}

fn main() {
    let n = 10;
    fib_loop(n);
    fib_while(n);
    fib_for(n);
}
```


It's important to note that Rust's `for` loop can be used with any data structure that implements the `IntoIterator` trait.

During execution, `IntoIterator` generates an iterator, and the `for` loop continuously retrieves values from the iterator until it returns `None`. Therefore, the `for` loop is essentially syntactic sugar. The compiler expands it to use a `loop` to access the iterator until it returns `None`.

In the `fib_for` function, you might notice syntax like `2..n`, which Python developers will recognize as a range operation. The range `2..n` includes all values where `2 <= x < n`. Like Python, Rust allows you to omit the start or end index of a range, for example:

```rust
let arr = [1, 2, 3];
assert_eq!(arr[..], [1, 2, 3]);
assert_eq!(arr[0..=1], [1, 2]);
```

However, unlike Python, Rust ranges do not support negative numbers. Therefore, you cannot use code like `arr[1..-1]`. This is because the start and end indices of a range are of the `usize` type, which cannot be negative.

Below is a summary of Rust's main control flow mechanisms.

![control flow](images/rust-03-03.webp)

## Pattern Matching

Rust's pattern matching is inspired by functional programming languages, making it powerful, elegant, and efficient. It can be used to match part or all of a `struct` or `enum`. For example, with the `Event` data structure defined earlier, you can match it as follows:

```rust
fn process_event(event: &Event) {
    match event {
        Event::Join((uid, _tid)) => println!("user {:?} joined", uid),
        Event::Leave((uid, tid)) => println!("user {:?} left {:?}", uid, tid),
        Event::Message((_, _, msg)) => println!("broadcast: {}", msg),
    }
}
```

In this code, you can see that it's possible to directly match and assign values from within the `enum`, which can save several lines of code compared to languages that only support simple pattern matching, such as JavaScript or Python.

Besides using the `match` keyword for pattern matching, you can also use `if let` or `while let` for simpler matches. If you only care about `Event::Message` from the previous example, you can write it like this:

```rust
fn process_message(event: &Event) {
    if let Event::Message((_, _, msg)) = event {
        println!("broadcast: {}", msg);
    }
}
```

Rust's pattern matching is a crucial language feature, widely used in state machine processing, message processing, and error handling.

## Error Handling

Rust does not follow the exception handling model used by predecessors like C++/Java. Instead, it borrows from Haskell, **encapsulating errors in the `Result<T, E>` type and providing the `?` operator to propagate errors conveniently**. The `Result<T, E>` type is a generic data structure where `T` represents the result type of successful execution, and `E` represents the error type.

In the `scrape_url` project we started today, many calls already use the `Result<T, E>` type. Here’s a demonstration of the code, using the `unwrap()` method to focus only on successful results. If an error occurs, the entire program terminates.

```rust
use std::fs;

fn main() {
    let url = "https://www.rust-lang.org/";
    let output = "rust.md";

    println!("Fetching url: {}", url);
    let body = reqwest::blocking::get(url).unwrap().text().unwrap();

    println!("Converting html to markdown...");
    let md = html2md::parse_html(&body);

    fs::write(output, md.as_bytes()).unwrap();
    println!("Converted markdown has been saved in {}.", output);
}
```

To propagate errors, replace all `unwrap()` calls with the `?` operator, and let the `main()` function return a `Result<T, E>`:

```rust
use std::fs;

// main function now returns a Result
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = "https://www.rust-lang.org/";
    let output = "rust.md";

    println!("Fetching url: {}", url);
    let body = reqwest::blocking::get(url)?.text()?;

    println!("Converting html to markdown...");
    let md = html2md::parse_html(&body);

    fs::write(output, md.as_bytes())?;
    println!("Converted markdown has been saved in {}.", output);

    Ok(())
}
```

## Organizing Rust Projects

As the scale of Rust code grows, it becomes impractical to contain all code in a single file. Multiple files or directories may need to work together, and you can use `mod` to organize the code. 

In the project’s entry file (`lib.rs` or `main.rs`), use `mod` to declare other code files to load. If a module contains many parts, it can be placed in a directory with a `mod.rs` file that includes other files of that module. This file functions similarly to Python’s `__init__.py`. After this setup, you can import the module using `mod + directory name`.

![organizing code](images/rust-03-04.webp)



In Rust, a project is also called a crate. A crate can be an executable project or a library, created with `cargo new <name> --lib`. When code in a crate changes, the crate needs to be recompiled.

Within a crate, unit tests and integration tests are also included.

Rust’s unit tests are usually placed in the same file as the code being tested, using conditional compilation with `#[cfg(test)]` to ensure the test code is only compiled in the test environment. Here’s an example of a unit test:

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
        assert_eq!(2 + 2, 4);
    }
}
```

Integration tests are generally placed in the `tests` directory, parallel to `src`. Unlike unit tests, integration tests can only test the public interface of the crate and are compiled into separate executable files.

To run test cases within a crate, use `cargo test`.

When the codebase continues to grow, placing all code in one crate becomes inefficient, as any code change necessitates recompiling the entire crate. This can be mitigated by **using a workspace**.

A workspace can contain one or more crates, and only the modified crates need to be recompiled. To build a workspace, create a `Cargo.toml` file in a directory, including all crates in the workspace, and then use `cargo new` to generate the corresponding crates.

![creating a workspace](images/rust-03-05.webp)

## Summary

We’ve briefly reviewed the basic concepts of Rust. We covered defining variables with `let/let mut`, creating functions with `fn`, and defining complex data structures using `struct` and `enum`. We also learned about Rust's basic control flow, how pattern matching works, and how to handle errors.

Finally, considering the issue of code scalability, we introduced how to organize Rust code using `mod`, `crate`, and `workspace`.


---