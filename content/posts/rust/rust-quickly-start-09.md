---
title: "Rust Rc, Arc, RefCell: Shared Ownership & Concurrency"

description: "Learn Rust multiple ownership with Rc/Arc, interior mutability via RefCell, Box::leak and DAG examples. Use Mutex/RwLock to safely share data across threads."

summary: "Master Rust’s shared ownership and runtime checks. Use Rc/Arc for multiple owners, RefCell for interior mutability, and Mutex/RwLock for thread-safe state. Includes a DAG example and a clear look at Box::leak, plus when to prefer static checks vs dynamic checks."

date: 2025-08-11
series: ["Rust"]
weight: 1
tags: ["rust", "smart-pointers", "concurrency", "multithreading", "interior-mutability"]
author: ["Garry Chen"]
cover:
  image: images/rust-09-00.webp
  hiddenInList: true
  caption: "Rust reference counting"

---

Previously, we learned about the single ownership rule in Rust, which satisfies most of our needs for memory allocation and usage. This rule allows Rust’s borrow checker to perform static checks during compilation, ensuring safety and efficiency without runtime overhead.

However, there are exceptions to this rule. What should we do in the following scenarios?
* In a Directed Acyclic Graph (DAG), a single node might be pointed to by two or more other nodes. How can we represent this in the ownership model?
* When multiple threads need to access the same shared memory, how do we handle that?

These issues arise during the program’s runtime and cannot be resolved by ownership’s static checks at compile time. To address such special cases, Rust offers runtime **dynamic checks** for greater flexibility.

This approach illustrates Rust’s philosophy: handle most common use cases at compile time to ensure safety and performance; for situations that cannot be handled during compilation, rely on runtime checks, trading some performance for flexibility. 

How Does Rust Handle Runtime Dynamic Checks?

Rust uses reference-counted smart pointers, specifically **Rc (Reference Counter) and Arc (Atomic Reference Counter)**, to enable multiple ownership. Note that Arc here is not the same as ARC (Automatic Reference Counting) in Objective-C/Swift, though they solve similar problems using reference counting.

## Rc

Let’s first discuss Rc. For a data structure T, you can create a reference-counted smart pointer Rc that allows multiple ownership. When you create an Rc, the underlying data is stored on the heap. As we mentioned earlier, the heap is the only place where dynamically allocated memory can be shared safely.

```rust
use std::rc::Rc;

fn main() {    
    let a = Rc::new(1);
}
```

If you want to create additional owners of this data, you can use the clone() method.

**When you call `clone()` on an Rc, it doesn’t copy the underlying data. Instead, it increases the reference count**. When an Rc goes out of scope and is dropped, the reference count decreases. The memory is only released when the reference count reaches zero.

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(1);
    let b = a.clone();
    let c = a.clone();
}
```

In the code above, we create three Rc instances: `a`, `b`, and `c`. All of them point to the same data on the heap, meaning the heap data now has three owners. When the program ends, `c` is dropped first, reducing the reference count to 2. Then `b` is dropped, reducing it to 1, and finally `a` is dropped. Once the count reaches zero, the heap memory is released.

![Rc visualization](images/rust-09-01.webp)

Why Doesn’t the Compiler Complain About Ownership Conflicts?

Let’s analyze the example: `a` is the original owner of the data created by `Rc::new(1)`. Both `b` and `c` are created by calling `a.clone()`, which returns new Rc instances. From the compiler’s perspective, each of `a`, `b`, and `c` owns its respective Rc smart pointer, and there are no ownership conflicts. If this explanation is confusing, we can dive into the implementation of clone():

```rust
fn clone(&self) -> Rc<T> {
    // Increase the reference count
    self.inner().inc_strong();
    // Create a new `Rc` instance from `self.ptr`
    Self::from_inner(self.ptr)
}
```

As we can see, calling `clone()` does not copy the actual data. It simply increments the reference count and creates a new Rc instance pointing to the same memory.

You might wonder: How is the data stored on the heap, and why isn’t this heap memory controlled by the stack’s lifecycle?


## Box::leak() Mechanism

In the previous discussion, we noted that under Rust’s ownership model, the lifetime of heap memory is tied to the lifetime of the stack memory that created it. The implementation of Rc seems to contradict this principle. Indeed, if Rust strictly adhered to the single ownership model, it would not be able to support reference counting (Rc).

Rust provides a way to bypass the compile-time ownership rules, **allowing code to allocate heap memory that is not governed by stack memory lifetimes**—similar to how memory is handled in C or C++. This mechanism is `Box::leak()`.

`Box` is Rust’s smart pointer, which allocates data structures on the heap while maintaining a pointer to them on the stack. Normally, the heap memory’s lifetime is still tied to the stack pointer’s lifetime.

`Box::leak()`, as its name suggests, leaks the heap-allocated object, making it independent of stack memory control. The leaked object essentially becomes a “free” object whose lifetime can extend to match the entire process’s lifetime.

![Box::leak() visualization](images/rust-09-02.webp)

In essence, `Box::leak()` intentionally creates a memory leak. Note that in C/C++, every block of heap memory allocated with malloc is effectively similar to Rust’s `Box::leak()`.

**With `Box::leak()`, we can bypass Rust’s static checks and ensure that the heap memory pointed to by `Rc` has the maximum possible lifetime**. At runtime, reference counting ensures that the memory is eventually released at the appropriate time.

Having understood Rc, we can better grasp Rust’s dual approach to ownership checks:

* Static checks: Performed by the compiler to enforce ownership rules.
* Dynamic checks: Using `Box::leak()` to create heap memory with unrestricted lifetimes, and reference counting to manage that memory’s lifecycle during runtime.

## Implementing a DAG

With `Rc`, we can now implement a Directed Acyclic Graph (DAG), which would have been impossible under the strict single ownership model.

For simplicity, assume each Node contains an id and a pointer to its downstream nodes. In a DAG, a node may have multiple incoming edges, so we use `Rc<Node>` to represent it. Additionally, a node may have no downstream nodes, so we use `Option<Rc<Node>>` for the downstream pointer.

![DAG visualization](images/rust-09-03.webp)

Here’s how we define and implement the Node structure:

* `new()`: Creates a new Node.  
*`update_downstream()`: Sets the downstream of the Node.  
* `get_downstream()`: Clones the downstream in the Node.

``` rust
use std::rc::Rc;

#[derive(Debug)]
struct Node {
    id: usize,
    downstream: Option<Rc<Node>>,
}

impl Node {
    pub fn new(id: usize) -> Self {
        Self {
            id,
            downstream: None,
        }
    }

    pub fn update_downstream(&mut self, downstream: Rc<Node>) {
        self.downstream = Some(downstream);
    }

    pub fn get_downstream(&self) -> Option<Rc<Node>> {
        self.downstream.as_ref().map(|v| v.clone())
    }
}

Using the above methods, we can build a DAG as shown:

fn main() {
    let mut node1 = Node::new(1);
    let mut node2 = Node::new(2);
    let mut node3 = Node::new(3);
    let node4 = Node::new(4);
    node3.update_downstream(Rc::new(node4));

    node1.update_downstream(Rc::new(node3));
    node2.update_downstream(node1.get_downstream().unwrap());
    println!("node1: {:?}, node2: {:?}", node1, node2);
}
```

## RefCell

After creating the DAG, you might wonder: Can the DAG be modified after it’s built?


Let’s attempt to modify Node3 to point to a new node (Node5) by adding the following code to the main() function:

``` rust
let node5 = Node::new(5);
let node3 = node1.get_downstream().unwrap();
node3.update_downstream(Rc::new(node5));

println!("node1: {:?}, node2: {:?}", node1, node2);

```

However, this will fail to compile with the error: “cannot borrow as mutable”. 

This happens because Rc is a **read-only reference counter**, and you cannot obtain a mutable reference to modify the data inside Rc. What can we do?

This is where `RefCell` comes in.

Like `Rc`, `RefCell` bypasses Rust’s compile-time checks and allows mutable borrowing of otherwise immutable data at runtime. This introduces the concept of interior mutability.

## Interior Mutability

With interior mutability, it naturally leads to the concept of exterior mutability, so `let's first look at this simpler definition and learn by comparison. 

When we explicitly declare a mutable value with `let mut`, or declare a mutable reference with `&mut`, the compiler can perform strict checks at compile time, ensuring that only mutable values or mutable references can modify the internal data of a value. This is called *exterior mutability*, and it is declared using the `mut` keyword. 

However, this is not flexible enough. Sometimes, we want to bypass this compile-time check and modify values or references that were not declared as `mut`. In other words, **in the eyes of the compiler, the value is read-only, but at runtime, this value can be mutably borrowed to modify its internal data**. This is where `RefCell` comes into play.

Here’s a simple example:

``` rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(1);
    {
        // Borrow a mutable reference to the RefCell's data
        let mut v = data.borrow_mut();
        *v += 1;
    }
    println!("data: {:?}", data.borrow());
}
```

In this example, `data` is a `RefCell` with an initial value of 1. As you can see, we haven't declared `data` as a mutable variable. Later, we can obtain a mutable internal reference by using the `borrow_mut()` method of `RefCell` and perform an increment operation on it. Finally, we can get an immutable internal reference using the `borrow()` method of `RefCell`, and since we've incremented it, its value becomes 2. 

You might wonder why we wrap the two lines of code that obtain and operate on the mutable borrow inside a pair of curly braces. 

This is because, according to ownership rules, **we cannot have active mutable and immutable borrows at the same time within the same scope**. By using these curly braces, we explicitly limit the lifetime of the mutable borrow to prevent conflicts with the subsequent immutable borrow. 

Now, think about it further—if these curly braces were omitted, would the code fail to compile, or would it cause a runtime error?

``` rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(1);
    
    let mut v = data.borrow_mut();
    *v += 1;
    
    println!("data: {:?}", data.borrow());
}

```

If you run Code , there will be no compilation issues, but when you reach line 9, you will encounter an error like: “already mutably borrowed: BorrowError.” As you can see, the ownership borrowing rules still apply here, but they are checked at runtime. This is the key difference between exterior mutability and interior mutability. 

Let’s summarize it in the following table:

![exterior_mutability_vs_interior_mutability](images/rust-09-04.webp)


## Implementing a Mutable DAG

Now that we have an intuitive understanding of `RefCell`, let’s see how to use it together with `Rc` to make the previously discussed DAG mutable.

First, the `downstream` data structure needs to have `Rc` internally wrapped with a `RefCell`. This way, we can take advantage of `RefCell`’s interior mutability to obtain mutable references to the data, while `Rc` allows multiple owners of the value.

![Mutable DAG visualization](images/rust-09-05.webp)

```rust
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Debug)]
struct Node {
    id: usize,
    // Use Rc<RefCell<T>> to make the node mutable
    downstream: Option<Rc<RefCell<Node>>>,
}

impl Node {
    pub fn new(id: usize) -> Self {
        Self {
            id,
            downstream: None,
        }
    }

    pub fn update_downstream(&mut self, downstream: Rc<RefCell<Node>>) {
        self.downstream = Some(downstream);
    }

    pub fn get_downstream(&self) -> Option<Rc<RefCell<Node>>> {
        self.downstream.as_ref().map(|v| v.clone())
    }
}

fn main() {
    let mut node1 = Node::new(1);
    let mut node2 = Node::new(2);
    let mut node3 = Node::new(3);
    let node4 = Node::new(4);

    node3.update_downstream(Rc::new(RefCell::new(node4)));
    node1.update_downstream(Rc::new(RefCell::new(node3)));
    node2.update_downstream(node1.get_downstream().unwrap());
    println!("node1: {:?}, node2: {:?}", node1, node2);

    let node5 = Node::new(5);
    let node3 = node1.get_downstream().unwrap();
    // Get a mutable reference to modify downstream
    node3.borrow_mut().downstream = Some(Rc::new(RefCell::new(node5)));

    println!("node1: {:?}, node2: {:?}", node1, node2);
}
```

As you can see, by using the nested structure `Rc<RefCell<T>>`, our DAG can now be modified normally.


## Arc and Mutex/RwLock

We solved the DAG problem using `Rc` and `RefCell`, but can we also use `Rc` to handle the issue of multiple threads accessing the same memory, which was mentioned at the beginning?

No. This is because `Rc` uses a non-thread-safe reference counter for performance reasons. Therefore, we need another smart pointer with reference counting that is thread-safe: `Arc`, which implements a thread-safe reference counter.

`Arc` internally uses an atomic `Usize` for reference counting, instead of a regular `usize`. From the name itself, you can tell that `Atomic Usize` is an atomic type, which uses special CPU instructions to ensure safety across threads.

Rust implements two different reference-counted data structures purely for performance reasons. Here we can also feel Rust’s extreme focus on performance. **If you don't need cross-thread access, you can use the highly efficient `Rc`; if you need cross-thread access, then you must use `Arc`**.

Similarly, `RefCell` is not thread-safe. If we want to use interior mutability in a multithreaded context, Rust provides `Mutex` and `RwLock`.

These two data structures should be familiar to you. `Mutex` is a mutual exclusion lock, where the thread that acquires the lock has exclusive access to the data. `RwLock` is a read-write lock, where the thread that acquires the write lock has exclusive access to the data, but multiple read locks are allowed when there is no write lock. The rules for read-write locks are very similar to Rust's borrowing rules, so we can learn by analogy.

Both `Mutex` and `RwLock` are used in multithreaded environments to protect access to shared data. If we want to use the DAG we constructed earlier in a multithreaded environment, we need to replace `Rc<RefCell<T>>` with either `Arc<Mutex<T>>` or `Arc<RwLock<T>>`.

## Summary

We have gained a deeper understanding of ownership and learned how to use data structures such as `Rc / Arc`, `RefCell / Mutex / RwLock`.

If we want to bypass the limitation of “one value having one owner,” we can use reference-counting smart pointers like `Rc / Arc`. `Rc` is very efficient but can only be used in a single-threaded environment, while `Arc` uses atomic structures, making it slightly less efficient but safe for multithreaded environments.

However, `Rc / Arc` are immutable, so to modify the internal data, we need to introduce interior mutability. In a single-threaded environment, we can use `RefCell` inside `Rc`; in a multithreaded environment, we can use `Arc` nested with `Mutex` or `RwLock`.

![Summary ](images/rust-09-06.webp)