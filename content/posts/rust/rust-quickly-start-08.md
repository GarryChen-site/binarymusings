---
title: "Rust Borrowing & References: Lifetimes and Borrow Checker"

description: "Learn Rust borrowing and references with clear examples: pass-by-value vs references, read-only vs mutable borrows, lifetimes, borrow checker fixes."

summary: "Hands-on guide to Rust borrowing. See how pass-by-value differs from explicit references, when to use read-only vs mutable borrows, and why borrows can’t outlive owners. Includes address demos, lifetimes, and practical fixes for borrow checker errors."

date: 2025-08-09
series: ["Rust"]
weight: 1
tags: ["rust", "borrowing", "references", "lifetimes", "borrow-checker"]
author: ["Garry Chen"]
cover:
  image: images/rust-08-00.webp
  hiddenInList: true
  caption: "Rust Borrowing & References"

---


When we assign variables, pass parameters, and return values from functions, if the data structure involved hasn't implemented the `Copy` trait, the ownership of the value is transferred by default using *Move semantics*. Once the ownership is transferred, the original variable can no longer access the data. However, if the data structure has implemented the `Copy` trait, *Copy semantics* will automatically be used to make a copy of the value, and the original variable will still be able to access the data.

Although single ownership solves the problem of arbitrary sharing of values that occurs in other languages, it also introduces some inconveniences. In the previous article, we mentioned: **what if you don't want to transfer the ownership of a value, but also can't use *Copy semantics* because the data structure doesn't implement the `Copy` trait?** In that case, you can *borrow* the data, which is the topic of this lesson: Borrow semantics.

## Borrow Semantics

As the name implies, *Borrow semantics* allow a value’s ownership to be used in other contexts without being transferred. It's like staying at a hotel or renting an apartment—guests or tenants have temporary usage rights but no ownership. Additionally, borrow semantics are implemented through reference syntax (`&` or `&mut`).

At this point, you might be confused. How did we introduce a new concept called "borrowing," but then write about "reference" syntax?

Actually, in Rust, *borrowing* and *reference* are the same concept. However, in other languages, the meaning of *reference* is different from Rust’s, so Rust introduced the concept of *borrowing* to distinguish between them.

In other languages, a reference is an alias, which you can simply understand as something like “Lu Xun” being another name for “Zhou Shuren.” Multiple references have identical access rights to the value, essentially sharing ownership. But in Rust, all references only "borrow" *temporary usage rights* and do not break the rule of single ownership.

Thus, **by default, Rust's borrow is read-only**, just like when staying in a hotel, you need to leave the room as it was when you checked out. However, in some cases, we also need *mutable borrowing*, similar to renting an apartment where you can make necessary modifications. We'll explain this in more detail later.

So, if you want to avoid *Copy* or *Move*, you can use borrowing, or rather, references.

## Read-only Borrowing/Referencing

Essentially, a reference is a controlled pointer to a specific type. When learning other languages, you’ll notice there are two ways to pass parameters: *pass-by-value* and *pass-by-reference*.

![pass-by-reference vs pass-by-value](images/rust-08-01.webp)

Take Java as an example: when passing an integer to a function, this is *pass-by-value*, similar to Rust’s *Copy semantics*. However, when passing an object or any heap-based data structure, Java implicitly passes it by reference. As mentioned earlier, Java's reference is an alias for the object, which leads to references to the same memory being spread all over the program, making it reliant on the garbage collector (GC) to manage memory.

But Rust does not have the concept of passing by reference. In Rust, **all parameter passing is by value**, whether through *Copy* or *Move*. So, in Rust, you must explicitly pass a reference of some data to another function.

Rust’s references implement the `Copy` trait, so according to *Copy semantics*, a copy of the reference will be passed to the function being called. For that function, it does not own the data itself; it’s only borrowing the data temporarily, while ownership remains with the original owner.

In Rust, references are first-class citizens, on par with other data types.

Let’s demonstrate this using the code from the previous article that had two errors:

```rust
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

Let’s slightly modify the code by adding references to make it compile and check the addresses of the value and reference.

```rust
fn main() {
    let data = vec![1, 2, 3, 4];
    let data1 = &data;
    // What is the address of the value? What is the address of the reference?
    println!(
        "addr of value: {:p}({:p}), addr of data {:p}, data1: {:p}",
        &data, data1, &&data, &data1
    );
    println!("sum of data1: {}", sum(data1));

    // What is the address of the data on the heap?
    println!(
        "addr of items: [{:p}, {:p}, {:p}, {:p}]",
        &data[0], &data[1], &data[2], &data[3]
    );
}

fn sum(data: &Vec<u32>) -> u32 {
    // Does the value's address change? Does the reference's address change?
    println!("addr of value: {:p}, addr of ref: {:p}", data, &data);
    data.iter().fold(0, |acc, x| acc + x)
}
```

Before running this code, you can first think about whether the address corresponding to `data` remains the same and whether the address of the `data1` reference still points to the same place after being passed to the `sum()` function.

Once you have your thoughts, you can run the code to verify if you're correct. Now, let’s look at the analysis in the following diagram:

![kernel address view](images/rust-08-02.webp)

Both `data1`, `&data`, and `data1'` passed into the `sum()` function point to the `data` itself, and the address of this value is fixed. However, the addresses of their references are all different, which confirms what we discussed earlier when introducing the `Copy` trait: **read-only references implement the `Copy` trait, which means assigning or passing references will produce new shallow copies**.

Even though many read-only references point to `data`, the data on the heap still has only one owner, so having multiple references to the value doesn’t affect the uniqueness of ownership.

But now we encounter a new problem: what happens if `data` goes out of scope and is dropped while there are still references pointing to it? Wouldn't this lead to the very memory safety issue we are trying to avoid, such as using freed memory (use-after-free)? What should we do?

## Borrow Lifetimes and Constraints

Therefore, references to a value must also follow constraints, namely: the borrow cannot outlive the value’s lifetime.

This constraint is intuitive and easy to understand. In the above code, the `sum()` function is one level down the call stack from the `main()` function. After it ends, the `main()` function continues to execute, so the lifetime of `data` defined in `main()` is longer than the reference to `data` in `sum()`, ensuring there are no issues.

But what about code like this?

```rust
fn main() {
    let r = local_ref();
    println!("r: {:p}", r);
}

fn local_ref<'a>() -> &'a i32 {
    let a = 42;
    &a
}
```

Clearly, the variable `r` in the longer-living `main()` function holds a reference to a local variable in the shorter-living `local_ref()` function, violating the reference constraint. Rust won’t allow this code to compile.

So, what if we try to use a stack reference in heap memory?

Based on previous development experience, you might instinctively say: "No!" because the lifetime of heap memory is generally longer and more flexible than stack memory, making this unsafe.

Let's write some code and see: we’ll store a reference to a local variable in a mutable array. From our basic knowledge, we know that a mutable array is stored on the heap, and only a fat pointer to it exists on the stack, so this is a typical case of storing a stack variable reference in the heap.

```rust
fn main() {
    let mut data: Vec<&u32> = Vec::new();
    let v = 42;
    data.push(&v);
    println!("data: {:?}", data);
}
```

Surprisingly, this compiles. What’s going on? Let’s modify the code a bit and see if it still compiles. Now it fails!

```rust
fn main() {
    let mut data: Vec<&u32> = Vec::new();
    push_local_ref(&mut data);
    println!("data: {:?}", data);
}

fn push_local_ref(data: &mut Vec<&u32>) {
    let v = 42;
    data.push(&v);
}
```

At this point, you might be a little confused. Why does the same reference to stack memory compile in some cases but not in others?

These three pieces of code may seem complex, but if you focus on the core idea—"in a given scope, at any time, a value can only have one owner"—you’ll find it simple.

The lifetime of heap variables is not arbitrarily flexible because heap memory’s lifecycle is tightly bound to the owner on the stack. And the lifecycle of stack memory is related to the stack's lifecycle. So, the key is to understand **the lifecycle of the call stack**.

Now, you can easily deduce why the code in situations 1 and 3 fails to compile: they reference values with shorter lifetimes. However, the code in situation 2 works because the heap memory references the stack memory, but their lifetimes are the same.

![call stack](images/rust-08-03.webp)

So far, we've covered Rust's default behavior for read-only borrowing. Borrowers cannot modify the borrowed value, which can be compared to staying in a hotel—you only have the right to use it, not alter it.

But as mentioned earlier, there are cases where we need *mutable borrowing*, where we want to modify the value during the borrowing period, similar to renting a house and making necessary modifications.


## Mutable Borrowing / References

Before introducing mutable borrowing, since a value can only have one owner at a time, the only way to modify the value is through its sole owner. However, allowing borrowing to change the value itself introduces new issues.

Let’s first look at the case where multiple mutable references coexist:

```rust
fn main() {
    let mut data = vec![1, 2, 3];

    for item in data.iter_mut() {
        data.push(*item + 1);
    }
}
```

In this code, while iterating over the mutable array `data`, new elements are being added to `data`. This is a dangerous operation because it breaks the loop invariant, which can easily lead to infinite loops or even system crashes. Therefore, having multiple mutable references in the same scope is unsafe.

Because the Rust compiler prevents this situation, the above code will fail to compile. Let’s simulate the potential deadlock caused by multiple mutable references using Python:

```python
if __name__ == "__main__":
    data = [1, 2]
    for item in data:
        data.append(item + 1)
        print(item)
    # unreachable code
    print(data)
```

If having multiple mutable references in the same context is unsafe, what about having **one mutable reference alongside several read-only references**? Let’s look at another example:

```rust
fn main() {
    let mut data = vec![1, 2, 3];
    let data1 = vec![&data[0]];
    println!("data[0]: {:p}", &data[0]);

    for i in 0..100 {
        data.push(i);
    }

    println!("data[0]: {:p}", &data[0]);
    println!("boxed: {:p}", &data1);
}
```

In this code, the immutable array `data1` holds a reference to an element in the mutable array `data`. This is a read-only reference. We then add 100 new elements to `data`, accessing a mutable reference to `data` by calling `data.push()`.

At first glance, the coexistence of a read-only reference and a mutable reference seems harmless because the element `data1` points to is not being changed.

However, upon closer inspection, you’ll find that there is a potential memory safety issue here. If we continue adding elements and the space reserved for data on the heap is insufficient, Rust will allocate a larger memory block, copy the existing values over, and then free the old memory. This would cause the `&data[0]` reference held by `data1` to become invalid, leading to a memory safety issue.

## Rust’s Constraints

While some automatic memory management systems like GC can avoid the second issue (coexisting mutable and read-only references), they cannot address the first issue (multiple mutable references).

Therefore, to ensure memory safety, Rust enforces strict rules on the use of mutable references:

- **Only one active mutable reference is allowed in a given scope.** "Active" means a mutable reference that is actually used to modify data. If it is defined but not used or only used as a read-only reference, it is not considered active.
  
- **Active mutable references (write) and read-only references (read) are mutually exclusive** and cannot coexist in the same scope.

These constraints might seem familiar to you. Indeed, they resemble the rules for read-write access in concurrency (like `RwLock`). You can draw parallels when learning them.

From the constraints on mutable references, we can see that Rust not only solves memory safety issues that GC can handle, but also those that GC cannot. When writing code, the Rust compiler serves as a mentor, constantly urging you to adopt best practices for writing safe code.

Once we peel away the many layers of ownership rules, and dig deeper into the fundamental concepts, we uncover how values are stored in the heap or stack, and how values are accessed in memory. From these concepts, we can either extend their applications or limit their usage. This is how we find fundamental solutions to complex problems, which is the design philosophy behind Rust.

## Summary

Today, we learned about **Borrowing Semantics**, understanding the principles of read-only references and mutable references. Combined with the Move/Copy semantics from the previous article, Rust’s compiler ensures that the code does not violate a series of rules:

- A value can have only one owner at a time. When the owner goes out of scope, the value it owns is discarded. Assignment or passing by value will cause the value to be **moved**, transferring ownership. Once ownership is transferred, the previous variable can no longer access the value.

- If a value implements the `Copy` trait, assignment or passing by value will use **Copy semantics**, meaning the value will be copied bit by bit to create a new value.

- A value can have multiple **read-only references**.

- A value can have only **one active mutable reference**. **Mutable references (write)** and **read-only references (read)** are mutually exclusive, much like how data access works in concurrency.

- The lifetime of a reference cannot exceed the lifetime of the value it refers to.

![a value within a certain scope](images/rust-08-04.webp)

But there are always special cases. For example, in Directed Acyclic Graphs (DAGs), we want to bypass the “one owner per value” rule. How do we handle that? We’ll explore that in the next article...

