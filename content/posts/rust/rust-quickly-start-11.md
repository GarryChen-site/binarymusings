---
title: "Rust Memory Management: Value Lifecycle & Drop Trait"

description: "Complete guide to Rust memory management: value creation, usage patterns, and destruction. Understand stack vs heap layouts and the Drop trait."

summary: "Follow a value's complete journey in Rust memory management: from creation with optimal struct layouts, through usage with Copy/Move semantics, to destruction with the Drop trait. Learn memory layouts, alignment rules, and automatic resource cleanup in this comprehensive deep dive."

date: 2025-08-24
series: ["Rust"]
weight: 1
tags: ["rust", "memory-management", "drop-trait", "value-lifecycle", "memory-layout"]
author: ["Garry Chen"]
cover:
  image: images/rust-11-00.webp
  hiddenInList: true
  caption: "Rust Memory Management Overview"

---



Since our first look into Rust, we’ve been learning about ownership and lifetimes. By now, you should have a solid understanding of the core ideas behind Rust’s memory management.

Through the single-ownership model, Rust solves the problem of heap memory being too flexible and difficult to safely and efficiently free. It avoids both the heavy mental burden and potential errors of manual memory management, as well as the efficiency costs of global mechanisms like tracing GC or ARC.

That said, the ownership model also introduces many new concepts—from Move/Copy/Borrow semantics to lifetime management—which can make learning a bit challenging.

But have you noticed? Most of these newly introduced concepts, including Copy semantics and value lifetimes, already exist implicitly in other languages. Rust simply makes them clearer, and more explicitly defines their scope of use.

Today, following the same line of thought as before, we’ll first organize and summarize the basics of Rust’s memory management. Then we’ll embark on the “fantastic journey” of a value—seeing what happens to it in memory, from creation to destruction—bringing together everything we’ve learned so far.

At this point, you might feel a little impatient—“Why are we talking about memory again today?” The reason is that memory management is at the core of any programming language, as important as inner strength in martial arts. Only when we truly understand how data is created, stored, and destroyed in memory can we read code and analyze problems with confidence and ease.

## Memory Management

[Heaps and stacks]({{< ref "rust-quickly-start-01.md" >}}) are the main occasions for the use of memory in code.

Stack memory allocation and deallocation are both highly efficient because they are determined at compile time. However, this means the stack cannot safely hold values of dynamic size or values whose lifetimes extend beyond the frame. To compensate for this limitation, we need memory that can be manipulated freely at runtime—the heap.

The heap is flexible enough, but managing the lifetime of heap-allocated data has long been a headache for programming languages.

C adopts an undefined approach, leaving it entirely up to the developer. C++ improves on this by introducing smart pointers, offering a mix of manual and automatic management. Java and .NET take full control of heap memory with garbage collection (GC), marking the beginning of the “managed” era of heap memory. Managed code is code that runs under a runtime system, which ensures safe access to heap memory.

The history of heap memory lifecycle management can be summarized as shown in the diagram:

![heap memory lifecycle management](images/rust-11-01.webp)

Rust’s creators took a fresh look at heap memory lifetimes. They found that **most heap usage stems from the need for dynamic sizing, while only a smaller portion requires extended lifetimes**. Therefore, by default, Rust ties the lifetime of heap memory to the lifetime of the stack memory that uses it. For the few cases where heap memory needs to outlive the stack frame, Rust provides the leaked mechanism, allowing memory to persist beyond its frame lifetime.

Here’s a comparison summary diagram:

![language lifecycle comparison](images/rust-11-02.webp)

With this foundation, let’s explore how Rust manages memory through the creation, use, and destruction of values.

The goal is that, after today’s discussion, when you look at a Rust data structure, you’ll be able to picture its memory layout: which fields live on the stack, which live on the heap, and roughly how large it is.

## Value Creation

When we create a value for a data structure and assign it to a variable, depending on the value’s nature, it may be created on the stack or on the heap.

As a [quick]({{< ref "rust-quickly-start-01.md" >}}) [review]({{< ref "rust-quickly-start-02.md" >}}): in theory, any value with a size determinable at compile time is placed on the stack. This includes Rust’s primitive types (like characters, arrays, and tuples), as well as developer-defined fixed-size structs and enums.

If a data structure’s size cannot be determined, or if its size is known but it requires a longer lifetime, then it’s better placed on the heap. Let’s now examine the memory layouts of several key Rust data structures when created: struct, enum, Vec\<T\>, and String.

### Struct
When laying out data in memory, Rust **rearranges fields based on their alignment to achieve optimal memory size and access efficiency**. For example, a struct with fields A, B, and C might be laid out as A, C, B in memory.

![](images/rust-11-03.webp)

Why does the Rust compiler do this?

Let’s first look at how C handles struct layouts. Consider two structs, `S1` and `S2`, each with three fields: `a`, `b`, and `c`. Fields `a` and `c` are `u8` (1 byte each), and `b` is `u16` (2 bytes). In `S1`, the order is `a`, `b`, `c`. In `S2`, the order is `a`, `c`, `b`.

What are the sizes of `S1` and `S2`?

```c
#include <stdio.h>

struct S1 {
    u_int8_t a;
    u_int16_t b;
    u_int8_t c;
};

struct S2 {
    u_int8_t a;
    u_int8_t c;
    u_int16_t b;
};

void main() {
    printf("size of S1: %d, S2: %d", sizeof(struct S1), sizeof(struct S2));
}
```

The correct answer is: 6 and 4.

Why is `S1` 6 bytes even though the fields only take up 4 bytes? **Because CPUs experience performance penalties when accessing misaligned memory**. To avoid this, the compiler inserts padding to align fields properly.

Here’s how C handles struct alignment:
1. Each field’s alignment equals its size.
2. Each field must start at an offset aligned to its size; if it doesn’t, padding is added.
3. The struct’s alignment equals the largest field’s alignment, and its total size is rounded up to a multiple of that alignment.

Applying these rules to `S1`:
* `a` (`u8`) has alignment 1.
* `b` (`u16`) has alignment 2. Since `a` takes only 1 byte, `b` starts at offset 1, which isn’t aligned, so the compiler adds 1 byte of padding. `b` then starts at offset 2.
* `c` (`u8`) is fine with no padding.

![C struct alignment and padding](images/rust-11-04.webp)

The total is 5 bytes, but the struct must be a multiple of 2 (the max alignment), so it rounds up to 6.

Thus, poor struct layout wastes memory: `S1` takes 50% more space than `S2` for the same data.

Best practice in C is to carefully order fields for efficient alignment. But this is tedious, especially with nested structs. Rust solves this automatically by reordering fields for you.

For the same example in Rust, both `S1` and `S2` are 4 bytes:

```rust
use std::mem::{align_of, size_of};

struct S1 {
    a: u8,
    b: u16,
    c: u8,
}

struct S2 {
    a: u8,
    c: u8,
    b: u16,
}

fn main() {
    println!("sizeof S1: {}, S2: {}", size_of::<S1>(), size_of::<S2>());
    println!("alignof S1: {}, S2: {}", align_of::<S1>(), align_of::<S2>());
}

```

You can also compare this visually in diagrams showing C vs. Rust behavior.

![C vs. Rust struct layout](images/rust-11-05.webp)

Although Rust defaults to optimizing struct layout, you can force C-style layouts with the `#[repr]` attribute, making Rust structs compatible with C code.

Now that we understand struct layouts in Rust (`tuples` behave similarly), let’s move on to enums. 

### enum

We’ve already talked about enum before — in Rust it is a tagged union. Its size is the size of the tag plus the size of the largest variant.

In the [basic syntax section]({{< ref "rust-quickly-start-03.md" >}}), when we defined enum data structures, we briefly mentioned two examples: `Option<T>` and `Result<T, E>`. Option is the simplest enum representing “some value / no value.” Result represents either a successful return value or an error value. We’ll go deeper into them later, but for now let’s just focus on how they are laid out in memory.

According to the three alignment rules we discussed earlier, the memory after the tag will be aligned to its required alignment. So for `Option<u8>`, its size is 1 + 1 = 2 bytes, while for `Option<f64>`, its size is 8 + 8 = 16 bytes. In general, on a 64-bit CPU, the maximum size of an enum is the size of the largest variant plus 8, because the maximum alignment on a 64-bit CPU is 64 bits (8 bytes). 

The diagram below shows the layout of `enum`, `Option<T>`, and `Result<T, E>`.

![Enum layout in Rust](images/rust-11-06.webp)

It’s worth noting that the Rust compiler applies extra optimizations to enums to make certain common structures more compact in memory. Let’s write a piece of code to better understand the sizes of different data structures:

``` rust
use std::collections::HashMap;
use std::mem::size_of;

enum E {
    A(f64),
    B(HashMap<String, String>),
    C(Result<Vec<u8>, String>),
}

// A declarative macro that prints sizes of data structures,
// their size when wrapped in Option, and in Result.
macro_rules! show_size {
    (header) => {
        println!(
            "{:<24} {:>4}    {}    {}",
            "Type", "T", "Option<T>", "Result<T, io::Error>"
        );
        println!("{}", "-".repeat(64));
    };
    ($t:ty) => {
        println!(
            "{:<24} {:4} {:8} {:12}",
            stringify!($t),
            size_of::<$t>(),
            size_of::<Option<$t>>(),
            size_of::<Result<$t, std::io::Error>>(),
        )
    };
}

fn main() {
    show_size!(header);
    show_size!(u8);
    show_size!(f64);
    show_size!(&u8);
    show_size!(Box<u8>);
    show_size!(&[u8]);

    show_size!(String);
    show_size!(Vec<u8>);
    show_size!(HashMap<String, String>);
    show_size!(E);
}

```

This code uses a declarative macro `show_size!`. Don’t worry too much about the macro itself for now. When you run this, you’ll see that `Option` combined with reference-like data structures such as `&u8`, `Box`, `Vec`, or `HashMap` takes **no additional space**. That’s very interesting.


``` rust
Type                        T    Option<T>    Result<T, io::Error>
----------------------------------------------------------------
u8                          1        2           24
f64                         8       16           24
&u8                         8        8           24
Box<u8>                     8        8           24
&[u8]                      16       16           24
String                     24       24           32
Vec<u8>                    24       24           32
HashMap<String, String>    48       48           56
E                          56       56           64

```

For Option, the tag only has two cases: 0 or 1. Tag = 0 means None, tag = 1 means Some.

Normally, if you combine it with a reference, the tag still takes 1 bit, but on a 64-bit CPU, references are aligned to 8 bytes. With padding, that would make it 16 bytes — wasteful.

Rust solves this cleverly: since the first field of a reference-like type is a pointer, and pointers can never be 0, **we can reuse the pointer value. If it’s 0, it means None. Otherwise, it means Some**. This optimization saves memory.


## Vec\<T\> and String

From the results above, we also see that `String` and `Vec<u8>` both occupy 24 bytes. If you look at the implementation of [String](https://doc.rust-lang.org/src/alloc/string.rs.html#279-281), you’ll see it’s essentially just a `Vec<u8>` under the hood.

A [Vec\<T\>](https://doc.rust-lang.org/src/alloc/vec/mod.rs.html#294-302) is a fat pointer consisting of three words: a pointer to the heap memory, the capacity of the heap allocation, the current length of the data.

Like this:

![Vec<T> memory layout](images/rust-11-07.webp)


Many dynamically sized data structures follow this pattern: **the stack holds a fat pointer, which points to the heap where the actual data lives**. We saw the same with Rc.

If you’re curious about memory layouts of other types, check out [cheats.rs](https://cheats.rs/#data-layout), a Rust language cheat sheet that’s very handy to browse anytime. For example, it shows the memory layout of reference types.

![Memory layout of reference types](images/rust-11-08.webp)

Now the value has been created, and we understand its memory layout. Next, let’s look at what happens during usage.

## Value Usage

When we talked about ownership, we learned that in Rust, if a value does not implement the `Copy` trait, it will be moved (Move) when assigned, passed as a parameter, or returned from a function.

**In fact, under the hood, both `Copy` and `Move` are shallow, bitwise memory copies**. The only difference is that `Copy` allows you to continue accessing the original variable, whereas `Move` does not. Let’s look at the diagram.

![Memory layout of `Copy` and `Move`](images/rust-11-09.webp)

In our intuition, memory copying feels heavy and inefficient. And yes, if every call on a critical path requires copying hundreds of kilobytes of data (like a large array), that’s inefficient.

However, if what you’re copying is just a primitive type (Copy) or a fat pointer on the stack (Move), and no heap memory is being duplicated (i.e., no deep copy), then the efficiency is very high. We don’t need to worry about performance loss from assignment or passing values in these cases.

So, whether Copy or Move, their efficiency is very high.

There is one exception worth noting: passing large arrays by value on the stack can hurt performance because it copies the entire array.Therefore, **we generally recommend not putting large arrays on the stack**. If you must, it’s better to pass them by reference instead of by value.

During value usage, aside from Move, you also need to be aware of dynamic growth. In Rust, collection types automatically expand as you use them.

Take `Vec<T>` as an example: when the existing heap capacity is full and you continue adding elements, the vector automatically reallocates and grows. Sometimes, if elements are added and removed frequently, the collection keeps a lot of unused capacity, wasting memory. In such cases, you might want to call methods like `shrink_to_fit` to optimize memory usage.


## Value Destruction

At this point, the value’s journey is more than halfway done—we’ve covered creation and usage, and now let’s talk about destruction.

Earlier, we mentioned that when an owner leaves scope, its value is dropped. But how does Rust actually drop it in code?

This is handled by the `Drop` trait. The `Drop` trait is like a destructor in object-oriented programming. **When a value is to be freed, its `drop()` method is automatically called**. For example, if a variable `greeting` is a string, when it goes out of scope, `drop()` is called automatically: the heap memory holding "hello world" is freed, and then the stack memory is released.

![Memory layout during drop](images/rust-11-10.webp)

If the value is a complex data structure, like a struct, then when the struct’s `drop()` is called, it will recursively call `drop()` on each of its fields. If a field is itself a complex structure or collection, the process continues until everything is properly released.

Example:

![Memory layout during drop of a complex struct](images/rust-11-11.webp)

If a `student` struct contains fields like `name`, `age`, and `scores`, where `name` is a `String` and `scores` is a `HashMap`, both of these need their own drops. Since the `HashMap`’s keys are also `String`, those keys are dropped first. The full release order is: drop the `HashMap` keys, drop the `HashMap`’s heap table, then drop the struct’s stack memory.


## Heap Memory Release

Because ownership ensures that each value has only one owner, heap memory release in Rust is simple: just call the `Drop` trait's `drop()` method. No extra concerns. This makes freeing values both safe and effortless—something unique to Rust.

Rust’s design in memory management reminds me of an ant colony. **Each ant follows simple, rigid rules, but together they form an efficient and error-free system**.

In contrast, in other languages, values are flexible and references can bounce around, making the system harder to analyze. Compilers alone can’t always decide if values in various scopes can be safely freed. This leads to two extremes: like in C/C++, developers must handle freeing manually, or like in Java, a separate system like GC is required to manage safe memory release.

In Rust, for most custom data structures, you don’t need to implement the `Drop` trait yourself—the compiler’s default behavior is enough. But if you want custom cleanup logic, you can implement `Drop` manually. Even if both your custom `drop()` and the system’s `drop()` target the same field, the compiler ensures it is only dropped once.

## Releasing Other Resources

While `Drop` is often used for heap memory, it can also release any resource: sockets, files, locks, etc. Rust provides RAII support for all resources.

For example, creating a file and writing "hello world":

``` rust
use std::fs::File;
use std::io::prelude::*;

fn main() -> std::io::Result<()> {
    let mut file = File::create("foo.txt")?;
    file.write_all(b"hello world")?;
    Ok(())
}

```

When file goes out of scope, not only is its memory freed, but the OS file descriptor is also closed automatically.

In other languages (Java, Python, Go), you must explicitly close the file to avoid resource leaks. Even with GC, non-memory resources are not automatically released.

Rust, however, through ownership, ensures the compiler knows exactly when a value leaves scope, so all its resources—including memory—can be safely freed.

You might think, “Isn’t it just skipping a `close()` call?” But in real-world code, much complexity comes from error handling. Managing `close()` calls across different contexts increases complexity. While Python’s with or Go’s defer help, they’re not perfect.

**When multiple variables and exceptions stack up, the risk of resource leaks grows quickly**. Many deadlocks and leaks arise this way.

From `Drop`, we see again how solving problems from first principles leads to elegant solutions. Just a few simple ownership rules automatically solve the hard problem of safe resource release.


## Summary

We further explored Rust’s memory management. Building on ownership and lifetimes, we looked at the full journey of a value: creation, usage, and destruction. We studied memory layouts at creation, size and alignment; we saw how values are moved or dynamically grow during usage; and we learned how values are destroyed with Drop.

Understanding data layout in memory—what’s on the stack and what’s on the heap—greatly helps us analyze code structure and efficiency.

