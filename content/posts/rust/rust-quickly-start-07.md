---
title: "Rust Ownership and Lifetimes Explained: Clear Examples"

description: "Understand Rust ownership, lifetimes and the borrow checker with stack vs heap basics, references vs moves, and fixes for typical compile errors."

summary: "Demystify Rust's ownership and lifetimes with a step-by-step, stack-first model. See how variables move and borrow across function calls, why the borrow checker complains, and how to resolve errors using references, moves, and explicit lifetimes through clear, practical examples."

date: 2025-08-08
series: ["Rust"]
weight: 1
tags: ["rust", "ownership", "lifetimes", "borrow-checker", "memory-safety"]
author: ["Garry Chen"]
cover:
  image: images/rust-07-00.webp
  hiddenInList: true
  caption: "ownership and lifetimes in Rust"

---

As your codebase grows, it seems the compiler starts to work against you. Some code that feels fine results in inexplicable compilation errors.

So today, let’s return to rationality and tackle the hardest challenge in learning Rust: ownership and lifetimes. Why start with this topic? Because **ownership and lifetimes are the main differences between Rust and other programming languages, and they form the foundation for understanding other concepts in Rust**.

Many beginners in Rust struggle at this point. They continue learning with only a partial understanding, which makes everything harder. When they try to write actual code, they easily stumble and face compilation errors, leading to a loss of confidence in Rust.

The reason ownership and lifetimes are so challenging to grasp isn't just because they address memory safety in a unique way; another significant factor is that current materials are not beginner-friendly. They often jump straight into explaining Copy/Move semantics without clarifying why they are used.

So in this article, we’ll take a different approach, starting from the behavior of a variable using the stack. We’ll explore the rationale behind Rust's design choices regarding ownership and lifetimes to help you fundamentally resolve these compilation issues.

## What Happens to Variables During Function Calls

First, let’s examine what happens to variables during function calls in most programming languages we're familiar with, and the issues that arise.

Consider this code where the `main()` function defines a dynamic array `data` and a value `v`, then passes them to the `find_pos` function to check if `v` exists in `data`. If it does, it returns the index of `v` in `data`; if not, it returns `None`.

```rust
fn main() {
    let data = vec![10, 42, 9, 8];
    let v = 42;
    if let Some(pos) = find_pos(data, v) {
        println!("Found {} at {}", v, pos);
    }
}

fn find_pos(data: Vec<u32>, v: u32) -> Option<usize> {
    for (pos, item) in data.iter().enumerate() {
        if *item == v {
            return Some(pos);
        }
    }
    
    None
}
```

This code is straightforward. It’s worth emphasizing that **the dynamic array is placed on the heap since its size cannot be determined at compile time, and there is a "fat pointer" on the stack pointing to the heap memory, containing its length and capacity**.

When `find_pos()` is called, the local variables `data` and `v` in the `main()` function are passed as parameters to `find_pos()`, placing them in the parameter area of `find_pos()`. 

![kernel stack diagram showing function call and parameter passing](images/rust-07-01.webp)

According to the conventions of most programming languages, the heap memory now has two references. Moreover, each time `data` is passed as a parameter, another reference is created for the heap memory.

However, what these references actually do is unclear, and we cannot impose any restrictions on them; it’s also challenging to determine when the heap memory can be released, especially when multiple call stacks are involved. It depends on when the last reference goes out of scope. Thus, this seemingly simple function call poses significant challenges for memory management.

For the issue of multiple references to heap memory, let’s first look at how most languages handle it:

* **C/C++ requires developers to manage it manually**, which is quite cumbersome. This demands high discipline when writing code, adhering to best practices summarized by others. However, mistakes can lead to memory safety issues, whether it be memory leaks or using freed memory, causing program crashes.

* **Languages like Java use tracing garbage collection (GC)**, which periodically scans the heap to check if data is still referenced by anyone, managing heap memory for developers. While this is a solution, the stop-the-world (STW) problem that GC introduces limits its usage scenarios and incurs performance costs.

* **Objective-C/Swift employs Automatic Reference Counting (ARC)**, which automatically adds code to maintain reference counts at compile time, relieving developers of some heap memory management burdens. However, it also incurs non-negligible runtime performance overhead.

Current solutions mainly approach the issue from the perspective of managing references, each with its drawbacks. Reflecting on the function call process we just outlined, the fundamental issue is that heap memory can be referenced freely. So, can we restrict the behavior of references themselves from another angle?

## Rust’s Solution

This idea opens new avenues, and Rust takes a unique approach.

Before Rust, references were casual, implicitly created, and lacked defined permissions—like pointers flying around in C or pass-by-reference everywhere in Java, which are both readable and writable, allowing broad access. Rust decided to restrict developers’ ability to reference freely.

As developers, we often find that **appropriate restrictions can unleash boundless creativity and productivity**. This is evident in various development frameworks like React and Ruby on Rails, which impose certain limitations on how developers can use the language but significantly enhance productivity.

Now that we have the idea, how can we implement restrictions on data reference behavior?

To answer this question, we first need to address: Who truly owns the data, or who holds the ultimate power over values? Should this power be shared or require exclusivity?

## Ownership and Move Semantics

Let’s start by addressing the question of whether the ultimate power over values can be shared or needs to be exclusive. Generally, we might agree that a value is best owned by a single owner, as shared ownership inevitably leads to ambiguity in usage and release, reverting us back to the old ways of tracing garbage collection (GC) or Automatic Reference Counting (ARC).

So, how can we ensure exclusivity? The actual implementation can be challenging because many scenarios need to be considered. For instance, a variable being assigned to another variable, passed as a parameter to a function, or returned from a function can all potentially create non-unique ownership. What can be done?

To this end, Rust provides the following rules:

1. **A value can only be owned by one variable, which is called its owner**.
2. **At any given time, a value can have only one owner**, meaning no two variables can own the same value. Therefore, in the cases we discussed—variable assignment, parameter passing, function returns—the old owner transfers the ownership of the value to the new owner to maintain single ownership.
3. **When the owner goes out of scope, the value will be dropped**, and memory will be freed.

These three rules are straightforward, focusing on ensuring single ownership. The second rule regarding ownership transfer is known as Move semantics, a concept Rust borrowed from C++.

The term scope introduced in the third rule is a new concept; it refers to a block of code. In Rust, a block of code enclosed in curly braces constitutes a scope. For example, if a variable is defined within an `if {}` block, the variable's scope ends when the `if` statement finishes, leading to its value being dropped. Similarly, variables defined within a function will be dropped when exiting that function.

Under the constraints of these three ownership rules, we can see how the reference issue at the beginning is resolved:

![diagram showing ownership transfer and invalidation of original variable](images/rust-07-02.webp)

The `data` in the `main()` function becomes invalid after being moved to `find_pos()`, and the compiler ensures that subsequent code in `main()` cannot access this variable, thus maintaining a unique reference to the heap memory.

You might have a small question: the parameter `v` passed to `find_pos()` is also moved, right? Why is it not marked in gray in the diagram? Let’s set this question aside for now; by the end of this article, you’ll have the answer.

Now, let’s write some code to deepen our understanding of ownership.

In this code, we first create an immutable data variable `data`, then assign `data` to `data1`. According to the ownership rules, after assignment, the value pointed to by `data` is moved to `data1`, making `data` itself inaccessible. Subsequently, `data1` is passed as a parameter to the `sum()` function, rendering `data1` also inaccessible in the `main()` function.

However, the subsequent code still tries to access `data1` and `data`, so this code should result in two errors.

``` rust
fn main() {
    let data = vec![1, 2, 3, 4];
    let data1 = data;
    println!("sum of data1: {}", sum(data1));
    println!("data1: {:?}", data1); // error1
    println!("sum of data: {}", sum(data)); // error2
}

fn sum(data: Vec<u32>) -> u32 {
    data.iter().fold(0, |acc, x| acc + x)
}
```

At runtime, the compiler indeed catches these two errors and clearly informs us that we cannot use variables that have already been moved.

![error messages indicating use of moved value](images/rust-07-03.webp)

If we want to pass `data1` to `sum()` while still allowing `main()` to access `data`, what can we do?

We can call `data.clone()` to create a copy of `data` for `data1`. This way, there will be two independent copies of `vec![1,2,3,4]` on the heap, which can be released independently, as shown in the diagram below:

![diagram showing cloning of data to create independent copies](images/rust-07-04.webp)

As we can see, **the ownership rules address the issue of who truly has the power over the data, eliminating multiple references to the data on the heap, which is its greatest advantage**.

However, this can complicate the code, especially for simple data types that only exist on the stack. To avoid the situation where access is lost after ownership transfer, we would need to manually copy the data, which can be cumbersome and inefficient.

Rust considers this and provides two solutions:

1. If you do not want the ownership of a value to be transferred, outside of Move semantics, Rust offers [Copy semantics](https://doc.rust-lang.org/std/marker/trait.Copy.html). If a data structure implements the Copy trait, it will use Copy semantics. This means that when you assign or pass a parameter, the value will be automatically copied bitwise (shallow copy).

2. If you do not want the ownership of a value to be transferred and cannot use Copy semantics, you can **"borrow" the data**. We will discuss "borrowing" in detail in the next article.

For now, let's look at the first solution we are discussing today: Copy semantics.


## Copy Semantics and the Copy Trait

Types that conform to Copy semantics will automatically perform a bitwise copy when you assign or pass parameters. This is straightforward to understand, but how is it implemented in Rust specifically?

Looking closely at the errors the compiler provided in the previous code, you’ll notice it complained that the type `Vec<u32>` didn’t implement the `Copy` trait, which means it couldn’t be copied when assigned or passed to a function and instead defaulted to using Move semantics. After a Move, the original variable `data` could no longer be accessed, which is why the error occurred.

![error message indicating Vec<u32> does not implement Copy trait](images/rust-07-05.webp)

In other words, when you attempt to move a value, if the type of that value has implemented the `Copy` trait, it will automatically use Copy semantics to create a copy. Otherwise, it will use Move semantics to transfer ownership.

Here, I’ll make a quick note: While learning Rust, you can try to modify your code based on the detailed error messages the compiler gives, so that it compiles correctly. In this process, you can also use Stack Overflow to search for error messages and further explore concepts you're unfamiliar with. I highly recommend that you use `rustc --explain E0382` to dive deeper into error code `E0382` from the example above for a more thorough explanation.

Now, returning to the main topic: What data structures implement the `Copy` trait in Rust? You can quickly check if a data structure implements the `Copy` trait with the following code:

```rust
fn is_copy<T: Copy>() {}

fn types_impl_copy_trait() {
    is_copy::<bool>();
    is_copy::<char>();

    // all iXX and uXX, usize/isize, fXX implement Copy trait
    is_copy::<i8>();
    is_copy::<u64>();
    is_copy::<i64>();
    is_copy::<usize>();

    // function (actually a pointer) is Copy
    is_copy::<fn()>();

    // raw pointer is Copy
    is_copy::<*const String>();
    is_copy::<*mut String>();

    // immutable reference is Copy
    is_copy::<&[Vec<u8>]>();
    is_copy::<&String>();

    // array/tuple with values which is Copy is Copy
    is_copy::<[u8; 4]>();
    is_copy::<(&str, &str)>();
}

fn types_not_impl_copy_trait() {
    // unsized or dynamic sized type is not Copy
    is_copy::<str>();
    is_copy::<[u8]>();
    is_copy::<Vec<u8>>();
    is_copy::<String>();

    // mutable reference is not Copy
    is_copy::<&mut String>();

    // array / tuple with values that not Copy is not Copy
    is_copy::<[Vec<u8>; 4]>();
    is_copy::<(String, u32)>();
}

fn main() {
    types_impl_copy_trait();
    types_not_impl_copy_trait();
}
```

I recommend running this code yourself and carefully reading the compiler errors to reinforce your understanding. Here’s a summary:

- **Primitive types**, including functions, immutable references, and raw pointers, implement `Copy`.
- **Arrays** and **tuples**, if their internal data structures implement `Copy`, will also implement `Copy`.
- **Mutable references** do not implement `Copy`.
- **Non-fixed-size data structures** do not implement `Copy`."


**Additionally**, [the official documentation](https://doc.rust-lang.org/std/marker/trait.Copy.html) on the page introducing the `Copy` trait includes all the data structures in the Rust standard library that implement the `Copy` trait. When you visit the documentation for any particular data structure, you can also check the *Trait Implementation* section to see if it implements the `Copy` trait.

![Trait Implementation](images/rust-07-06.webp)

## Summary

Today, we learned about Rust's single ownership model, Move semantics, and Copy semantics. Here’s a recap of the key points for you to review:

- **Ownership**: A value can only be owned by one variable, and at any given moment, there can only be one owner. When the owner goes out of scope, the value it owns is dropped, and the memory is freed.
- **Move semantics**: Assigning or passing a value will cause it to Move, transferring ownership. Once ownership is transferred, the previous variable can no longer be accessed.
- **Copy semantics**: If a value implements the `Copy` trait, then assigning or passing it will use Copy semantics, and the value will be copied bit by bit (shallow copy), creating a new instance of the value.


By using the single ownership model, Rust addresses the problem of heap memory being too flexible and difficult to release safely and efficiently. However, the ownership model also introduces many new concepts, such as the Move/Copy semantics we discussed today.

Since these are new concepts, they can be somewhat challenging to learn. But if you focus on the core idea—**that Rust restricts arbitrary referencing behavior through single ownership**—understanding the design logic behind these concepts becomes much easier.

In the next article, we will continue learning about Rust's ownership and lifetimes, specifically how to “borrow” data when you don’t want to transfer ownership and can’t use Copy semantics...

---