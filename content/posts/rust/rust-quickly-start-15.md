---
title: "Rust Smart Pointers Guide: Box, Cow & MutexGuard Explained"

description: "Master Rust smart pointers: Box<T> for heap allocation, Cow for copy-on-write, MutexGuard for thread safety. Learn Deref/Drop traits implementation."

summary: "Deep dive into Rust smart pointers covering Box<T> for heap memory allocation, Cow<'a, B> for copy-on-write optimization, and MutexGuard<T> for thread-safe resource management. Learn how smart pointers implement Deref, DerefMut, and Drop traits to provide automatic memory management and resource cleanup. Includes practical examples of custom allocators, URL parsing optimization, and building your own MyString smart pointer for efficient small string storage."

date: 2025-11-01
series: ["Rust"]
weight: 1
tags: ["rust", "smart-pointers", "memory-management", "box", "cow"]
author: ["Garry Chen"]
cover:
  image: images/rust-quick/rust-15-00.webp
  hiddenInList: true
  caption: "Data Structures with Smart Pointers in Rust"

---


Up to now, we have learned Rust’s ownership and lifetimes, memory management, and type system. There’s still one major foundational area we haven’t covered yet: data structures. Among them, the most confusing topic is smart pointers — so today we’re going to tackle this challenge.

We briefly introduced pointers before, but let’s review first: a pointer is a value that holds a memory address, and through dereferencing, you can access the memory address it points to — theoretically, you can dereference to any data type. A reference is a special kind of pointer whose dereferencing access is restricted — it can only dereference to the type of data it references, and cannot be used for other purposes.

So, what exactly is a smart pointer?


## Smart Pointers

Based on pointers and references, Rust takes inspiration from C++ and provides smart pointers. A smart pointer is a data structure that behaves much like a pointer, but besides holding a pointer to data, it also contains metadata that provides additional functionality.

This definition might sound vague, so let’s clarify it by comparing it to other data structures. 

Doesn’t it remind you of the fat pointer concept we discussed earlier?
A smart pointer is always a fat pointer, but a fat pointer is not necessarily a smart pointer. For example, `&str` is just a fat pointer — it has a pointer to the string data on the heap, along with metadata about the string length.

Let’s look at the difference between the smart pointer `String` and `&str`:

![String and &str memory layout](images/rust-quick/rust-15-01.webp)

From the diagram, you can see that `String`, aside from having an extra capacity field, doesn’t look that special. **However, `String` has ownership of the heap value, while `&str` does not. This is the key difference between a smart pointer and a regular fat pointer in Rust**.

Now another question arises — what’s the difference between a smart pointer and a struct? Because as we know, `String` is defined using a struct:

``` rust
pub struct String {
    vec: Vec<u8>,
}
```

Unlike ordinary structs, `String` implements `Deref` and `DerefMut`. This allows it to return a `&str` when dereferenced. See the standard library implementation below:

``` rust
impl ops::Deref for String {
    type Target = str;

    fn deref(&self) -> &str {
        unsafe { str::from_utf8_unchecked(&self.vec) }
    }
}

impl ops::DerefMut for String {
    fn deref_mut(&mut self) -> &mut str {
        unsafe { str::from_utf8_unchecked_mut(&mut *self.vec) }
    }
}
```

In addition, since it allocates data on the heap, `String` also needs to clean up the resources it allocates. Inside, `String` uses `Vec<u8>`, so it can rely on `Vec<T>`’s ability to release heap memory. Here’s the implementation of the `Drop` trait for `Vec<T>` in the standard library:

``` rust
unsafe impl<#[may_dangle] T, A: Allocator> Drop for Vec<T, A> {
    fn drop(&mut self) {
        unsafe {
            // use drop for [T]
            // use a raw slice to refer to the elements of the vector as the weakest necessary type;
            // could avoid questions of validity in certain cases
            ptr::drop_in_place(ptr::slice_from_raw_parts_mut(self.as_mut_ptr(), self.len))
        }
        // RawVec handles deallocation
    }
}
```

So, to refine our definition — **in Rust, any data structure that needs to perform resource cleanup and implements `Deref` / `DerefMut` / `Drop` is a smart pointer**.

According to this definition, besides `String`, we have already encountered many smart pointers in previous articles — for example,
`Box<T>` and `Vec<T>` for heap memory allocation, and `Rc<T>` and `Arc<T>` for reference counting. Many other data structures, such as `PathBuf`, `Cow<'a, B>`, `MutexGuard<T>`, `RwLockReadGuard<T>`, and `RwLockWriteGuard<T>` are also smart pointers.

Today, we’ll analyze three data structures that use smart pointers in depth: `Box<T>` (for allocating memory on the heap), `Cow<'a, B>` (for copy-on-write), and `MutexGuard<T>` (for data locking).

Finally, we’ll try implementing our own smart pointer. By the end of this article, you should not only understand smart pointers more deeply but also be able to build your own when needed to solve specific problems.

## `Box<T>`

Let’s first look at `Box<T>`. It is Rust’s most fundamental way of allocating memory on the heap. Most other data types that involve heap memory allocation use `Box<T>` internally — for example, `Vec<T>`.

To understand why `Box<T>` exists, we should recall how heap memory allocation works in C.

In C, you need to use `malloc` / `calloc` / `realloc` / `free` to handle memory allocation. Many times, the allocated memory is used back and forth across function calls, making it hard to determine who should be responsible for freeing it — which causes a significant mental burden for developers.

C++ improves upon this by providing a smart pointer `unique_ptr`, which releases heap memory automatically when the pointer goes out of scope, ensuring single ownership of heap memory. This `unique_ptr` is the predecessor of Rust’s `Box<T>`.

If you look at the definition of `Box<T>`, you’ll see that internally it contains a `Unique<T>` — a tribute to C++. `Unique<T>` is a private data structure that we cannot use directly; it wraps a `*const T` pointer and uniquely owns it.

``` rust
pub struct Unique<T: ?Sized> {
    pointer: *const T,
    // NOTE: this marker has no consequences for variance, but is necessary
    // for dropck to understand that we logically own a `T`.
    //
    // For details, see:
    // https://github.com/rust-lang/rfcs/blob/master/text/0769-sound-generic-drop.md#phantom-data
    _marker: PhantomData<T>,
}
```

We know that allocating memory on the heap requires a memory allocator. If you’ve taken an operating systems course, you might remember how a simple [buddy system](https://en.wikipedia.org/wiki/Buddy_memory_allocation) allocates and manages heap memory.

The purpose of designing a memory allocator — apart from correctness — is to make efficient use of remaining memory and control the amount of fragmentation created during allocation and deallocation. In a multicore environment, it must also efficiently handle concurrent allocation requests.

Heap allocation in `Box<T>` actually involves a default generic parameter `A`, which must satisfy the `Allocator` trait, and defaults to `Global`:

``` rust
pub struct Box<T: ?Sized, A: Allocator = Global>(Unique<T>, A)
```

The `Allocator` trait provides many methods:
* `allocate`: the main method used to allocate memory — corresponding to C’s `malloc` / `calloc`;
* `deallocate`: used to free memory — corresponding to C’s `free`;
* `grow / shrink`: used to expand or shrink already allocated heap memory — corresponding to C’s `realloc`.

We won’t go into detail about the `Allocator` trait here.
If you want to replace the default memory allocator, you can use the `#[global_allocator]` attribute macro to define your own global allocator. The following code shows how to use jemalloc in Rust:

``` rust
use jemallocator::Jemalloc;

#[global_allocator]
static GLOBAL: Jemalloc = Jemalloc;

fn main() {}
```

After this setup, any memory you allocate using `Box::new()` will be allocated by jemalloc. Additionally, if you want to implement your own global allocator, you can implement the `GlobalAlloc` trait. The difference between `GlobalAlloc` and `Allocator` mainly lies in whether zero-length allocations are permitted.


## Use Cases

Now let’s implement our own memory allocator. Don’t worry — we’re just going to debug and see how memory gets allocated and freed; we won’t actually implement a real allocation algorithm.

First, let’s look at memory allocation. Here, `MyAllocator` will simply use the system allocator, but we’ll add an `eprintln!()` statement. Unlike `println!()`, `eprintln!()` prints data to stderr.

``` rust
use std::alloc::{GlobalAlloc, Layout, System};

struct MyAllocator;

// Track recursion depth to prevent infinite loops
use std::cell::Cell;
thread_local! {
    static IN_ALLOC: Cell<bool> = Cell::new(false);
}

unsafe impl GlobalAlloc for MyAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let data: *mut u8 = unsafe { System.alloc(layout) };

        // Only log if we're not already inside an allocation
        IN_ALLOC.with(|in_alloc| {
            if !in_alloc.get() {
                in_alloc.set(true);
                eprintln!("ALLOC: {:p}, size {}", data, layout.size());
                in_alloc.set(false);
            }
        });

        data
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        IN_ALLOC.with(|in_alloc| {
            if !in_alloc.get() {
                in_alloc.set(true);
                eprintln!("FREE: {:p}, size {}", ptr, layout.size());
                in_alloc.set(false);
            }
        });

        unsafe { System.dealloc(ptr, layout) };
    }
}

#[global_allocator]
static GLOBAL: MyAllocator = MyAllocator;

#[allow(dead_code)]
struct Matrix {
    // Using an irregular number like 505 makes dbg! output easier to identify
    data: [u8; 505],
}

impl Default for Matrix {
    fn default() -> Self {
        Self { data: [0; 505] }
    }
}

fn main() {
    // Many allocations happen before this line executes
    let data = Box::new(Matrix::default());

    // One of the 1024-byte allocations in the output is caused by println!
    println!(
        "!!! allocated memory: {:p}, len: {}",
        &*data,
        std::mem::size_of::<Matrix>()
    );

    // data is dropped here — you’ll see FREE in the output
    // Many other pieces of memory will also be freed afterward
}

```

Note that you cannot use `println!()` here, because `stdout` prints to a shared global buffer protected by a Mutex lock. This process involves memory allocation, which would again trigger `println!()`, eventually causing the program to crash. `eprintln!()`, on the other hand, prints directly to `stderr` without buffering.

When you run this code, you’ll see output like the following — the 505-byte allocation is from our `Box::new()`:

``` shell
❯ cargo run --bin allocator --quiet
ALLOC: 0x105311db0, size 4
ALLOC: 0x105311ba0, size 64
ALLOC: 0x105311e10, size 456
ALLOC: 0x105312010, size 505
ALLOC: 0x105312210, size 1024
ALLOC: 0x105311be0, size 64
!!! allocated memory: 0x105312010, len: 505
FREE: 0x105312010, size 505
FREE: 0x105312210, size 1024
FREE: 0x105311db0, size 4
```

When allocating heap memory with `Box`, note that `Box::new()` is a function, so the data passed into it first appears on the stack before being moved to the heap. If our `Matrix` structure isn’t just 505 bytes but instead is very large, that could cause issues.

For example, the following code attempts to allocate 16 MB on the heap — if you run it in the playground, it will immediately cause a stack overflow:

``` rust
fn main() {
    // Allocate 16 MB on the heap, but it first appears on the stack before moving
    let boxed = Box::new([0u8; 1 << 24]);
    println!("len: {}", boxed.len());
}
```

However, if you compile it locally with `cargo run --release`, it runs successfully!

That’s because running with `cargo run` or in the Playground uses the debug build by default, which performs no inlining optimization.
The implementation of `Box::new()` is just one line, and it’s explicitly marked as inline — so in release mode, the function call gets optimized away:

``` rust
#[cfg(not(no_global_oom_handling))]
#[inline(always)]
#[doc(alias = "alloc")]
#[doc(alias = "malloc")]
#[stable(feature = "rust1", since = "1.0.0")]
pub fn new(x: T) -> Self {
    box x
}
```

If inlining doesn’t happen, the entire 16 MB array would be passed through stack memory into `Box::new`, causing a stack overflow. Here we also discover a new keyword — `box`. However, `box` is an internal Rust keyword that user code cannot invoke. It only appears in Rust’s own code for heap allocation. During compilation, the `box` keyword uses the allocator to allocate memory.

Now that we understand how `Box<T>` allocates memory, we also care about how it frees memory — let’s look at its `Drop` implementation:

``` rust
#[stable(feature = "rust1", since = "1.0.0")]
unsafe impl<#[may_dangle] T: ?Sized, A: Allocator> Drop for Box<T, A> {
    fn drop(&mut self) {
        // FIXME: Do nothing, drop is currently performed by compiler.
    }
}
```

Ah — currently, the `Drop` trait doesn’t do anything, because the compiler automatically inserts the deallocate code. This is part of Rust’s design philosophy: **before a specific implementation is fully stabilized, the interface itself can be stabilized first, while the implementation is gradually refined over time**.

This strategy greatly helps avoid breaking changes for developers as the language evolves. Breaking changes force developers to significantly modify their existing code when upgrading language versions.

Python serves as a cautionary tale — due to many breaking changes,
it took more than ten years for the migration from Python 2 to Python 3 to be completed. Therefore, Rust is extremely cautious when designing its interfaces. Many important ones live as library components for a long time before finally entering the standard library — for example, the Future trait. Once an interface becomes stable, its internal implementation can then evolve gradually.


## `Cow<'a, B>`

After understanding how Box works, let’s look at the principle and use cases of `Cow<'a, B>`. In this [article]({{< ref "rust-quickly-start-12.md" >}}) (on generic data structures), we briefly talked about the three constraints on parameter `B`.

`Cow` is a smart pointer in Rust that provides Clone-on-Write behavior — similar to Copy-on-Write in virtual memory management: **it wraps a read-only borrow, but if the caller needs ownership or wants to modify the content, it will clone the borrowed data**.

Let’s look at the definition of `Cow`:

``` rust
pub enum Cow<'a, B> where B: 'a + ToOwned + ?Sized {
  Borrowed(&'a B),
  Owned(<B as ToOwned>::Owned),
}
```

It’s an `enum` that can contain either a read-only reference to type `B`, or an owned value of type `B`.

Here two traits are introduced. First is `ToOwned`, which, when defined, also introduces the `Borrow` trait. Both are under `std::borrow`:

``` rust
pub trait ToOwned {
    type Owned: Borrow<Self>;
    #[must_use = "cloning is often expensive and is not expected to have side effects"]
    fn to_owned(&self) -> Self::Owned;

    fn clone_into(&self, target: &mut Self::Owned) { ... }
}

pub trait Borrow<Borrowed> where Borrowed: ?Sized {
    fn borrow(&self) -> &Borrowed;
}
```

If you don’t fully understand this code yet, don’t worry — to understand the `Cow` trait, `ToOwned` is the main hurdle, because type `Owned: Borrow<Self>` is tricky to grasp. Let’s break it down step by step.

First, type `Owned: Borrow<Self>` means it’s a trait with an associated type. If you’ve forgotten about this concept, review this [article]({{< ref "rust-quickly-start-13.md" >}}). Here `Owned` is the associated type that must be defined by the implementor — and unlike a normal associated type, this one must satisfy `Borrow<T>`. For example, let’s see the `ToOwned` implementation for `str`:

``` rust
impl ToOwned for str {
    type Owned = String;
    #[inline]
    fn to_owned(&self) -> String {
        unsafe { String::from_utf8_unchecked(self.as_bytes().to_owned()) }
    }

    fn clone_into(&self, target: &mut String) {
        let mut b = mem::take(target).into_bytes();
        self.as_bytes().clone_into(&mut b);
        *target = unsafe { String::from_utf8_unchecked(b) }
    }
}
```

You can see that the associated type `Owned` is defined as `String`. And per the requirement, `String` must implement `Borrow<T>`. Now, what is the generic parameter `T` in `Borrow<T>` here?

`ToOwned` requires `Borrow<Self>`. At this point, the implementor of `ToOwned` is `str`, so `Borrow<Self>` becomes `Borrow<str>`.
In other words, `String` must implement `Borrow<str>`. And indeed, the [docs](https://doc.rust-lang.org/std/string/struct.String.html#impl-Borrow%3Cstr%3E) show it does:

``` rust
impl Borrow<str> for String {
    #[inline]
    fn borrow(&self) -> &str {
        &self[..]
    }
}
```

Feeling a little dizzy? Here’s a diagram that summarizes the relationships among these traits:

![Cow, ToOwned, Borrow relationships](images/rust-quick/rust-15-02.webp)

With this diagram, you can better understand how `Cow`, `ToOwned`, and `Borrow<T>` relate to each other.

You might wonder: why define `Borrow` as a generic trait? Is it really necessary? Can a type be borrowed as different reference types?

Yes, it can. Here’s an example:

``` rust
use std::borrow::Borrow;

fn main() {
    let s = "hello world!".to_owned();

    // The type must be specified here because String implements multiple Borrow<T>
    // Borrow as &String
    let r1: &String = s.borrow();
    // Borrow as &str
    let r2: &str = s.borrow();

    println!("r1: {:p}, r2: {:p}", r1, r2);
}
```

In this example, `String` can be borrowed as both `&String` and `&str`.

Now, back to `Cow`. Since it’s a smart pointer, it naturally implements the `Deref` trait:

``` rust
impl<B: ?Sized + ToOwned> Deref for Cow<'_, B> {
    type Target = B;

    fn deref(&self) -> &B {
        match *self {
            Borrowed(borrowed) => borrowed,
            Owned(ref owned) => owned.borrow(),
        }
    }
}
```

The logic is simple: depending on whether self is Borrowed or Owned, we extract the reference accordingly:
* For Borrowed, it’s already a reference.
* For Owned, we call its `borrow()` method to get a reference.

That’s pretty powerful — even though `Cow` is an enum, through its `Deref` implementation, we get a unified experience. For example, with `Cow<str>`, it feels nearly identical to using `&str` or `String`. Note: **this technique of dispatching based on enum variants is a third kind of dispatch**. We previously talked about static dispatch (via generics) and dynamic dispatch (via trait objects).


## Use Cases

So what’s `Cow` useful for? Clearly, it helps defer memory allocation and copying until necessary — in many scenarios, this can significantly improve performance. If the owned data type inside `Cow<'a, B>` is something heap-allocated (like `String` or `Vec<T>`), it can also reduce the number of heap allocations.

As we’ve discussed before, heap allocation and deallocation are much more expensive than stack operations — they often involve system calls and locks. **Reducing unnecessary heap allocations is key to improving performance**. And `Cow<'a, B>` helps you achieve that, while keeping your code ergonomic and simple.

Let’s look at a real-world example to back this up.

When parsing URLs, we often need to extract query string parameters into key–value pairs. In most languages, the extracted KVs are new strings — and in a system processing tens or hundreds of thousands of requests per second, this means massive amounts of heap allocation.

In Rust, we can handle this efficiently with `Cow`. While reading the URL:
* For each parsed key or value, we can use a `&str` that points into the URL buffer, then wrap it in `Cow`.
* If the parsed content needs to be decoded (e.g., "hello%20world"), we can generate a decoded `String` and wrap that in `Cow` as well.

See the following example:

``` rust
use std::borrow::Cow;

use url::Url;
fn main() {
    let url = Url::parse("https://example.com/rust?page=1024&sort=desc&extra=hello%20world").unwrap();
    let mut pairs = url.query_pairs();

    assert_eq!(pairs.count(), 3);

    let (mut k, v) = pairs.next().unwrap();
    // Because both k and v are Cow<str>, they feel just like &str or String when used
    // At this moment, they are both Borrowed
    println!("key: {}, v: {}", k, v);
    // When modification occurs, k becomes Owned
    k.to_mut().push_str("_lala");

    print_pairs((k, v));

    print_pairs(pairs.next().unwrap());
    // When processing extra=hello%20world, the value is processed into "hello world"
    // So here the value is Owned
    print_pairs(pairs.next().unwrap());
}

fn print_pairs(pair: (Cow<str>, Cow<str>)) {
    println!("key: {}, value: {}", show_cow(pair.0), show_cow(pair.1));
}

fn show_cow(cow: Cow<str>) -> String {
    match cow {
        Cow::Borrowed(v) => format!("Borrowed {}", v),
        Cow::Owned(v) => format!("Owned {}", v),
    }
}
```

Isn’t it concise.

This kind of processing, like URL parsing, is very common in Rust’s standard library and third-party libraries. For example, Rust’s famous [serde](https://serde.rs/) library can very efficiently serialize/deserialize Rust data structures, and it has very good support for `Cow`.

We can use the following code to deserialize a JSON data into the `User` type, while allowing the `name` field inside `User` to use `Cow` to reference the content from the JSON text:

``` rust
use serde::Deserialize;
use std::borrow::Cow;

#[derive(Debug, Deserialize)]
struct User<'input> {
    #[serde(borrow)]
    name: Cow<'input, str>,
    age: u8,
}

fn main() {
    let input = r#"{ "name": "Tyr", "age": 18 }"#;
    let user: User = serde_json::from_str(input).unwrap();

    match user.name {
        Cow::Borrowed(x) => println!("borrowed {}", x),
        Cow::Owned(x) => println!("owned {}", x),
    }
}
```

In the future, when you build systems in Rust, you can also fully consider using `Cow` in your data types.


## MutexGuard

If the smart pointers introduced above such as `String`, `Box<T>`, and `Cow<'a, B>` all provide a good user experience through `Deref`, then `MutexGuard<T>` is another very interesting type of smart pointer: it not only provides a good user experience via `Deref`, **but also ensures that resources other than memory used are released when exiting, through the `Drop` trait**.

The `MutexGuard` structure is generated when calling `Mutex::lock`:

``` rust
pub fn lock(&self) -> LockResult<MutexGuard<'_, T>> {
    unsafe {
        self.inner.raw_lock();
        MutexGuard::new(self)
    }
}
```

First, it acquires the lock resource; if it cannot get it, it will wait here; if it gets it, it will pass the reference of the Mutex structure to `MutexGuard`.

Let’s look at the definition of `MutexGuard` and its implementations of `Deref` and `Drop`; it’s quite simple:

``` rust
// Here must_use is used; when you get a MutexGuard but don’t use it, a warning will appear
#[must_use = "if unused the Mutex will immediately unlock"]
pub struct MutexGuard<'a, T: ?Sized + 'a> {
    lock: &'a Mutex<T>,
    poison: poison::Guard,
}

impl<T: ?Sized> Deref for MutexGuard<'_, T> {
    type Target = T;

    fn deref(&self) -> &T {
        unsafe { &*self.lock.data.get() }
    }
}

impl<T: ?Sized> DerefMut for MutexGuard<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        unsafe { &mut *self.lock.data.get() }
    }
}

impl<T: ?Sized> Drop for MutexGuard<'_, T> {
    #[inline]
    fn drop(&mut self) {
        unsafe {
            self.lock.poison.done(&self.poison);
            self.lock.inner.raw_unlock();
        }
    }
}
```

From the code you can see that when `MutexGuard` ends, the `Mutex` will perform unlock, so when users use `Mutex`, they don’t need to worry about when to release the mutex. Because no matter how you pass `MutexGuard` along the call stack, even if you exit early during error handling, Rust’s ownership mechanism ensures that as soon as `MutexGuard` leaves the scope, the lock will be released.

## Use Cases

Let’s look at an example of using `Mutex` and `MutexGuard` . The code is simple.

``` rust
use lazy_static::lazy_static;
use std::borrow::Cow;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

// The lazy_static macro can generate complex static objects
lazy_static! {
    // Generally, Mutex and Arc are used together to provide access to shared memory in multithreaded environments
    // If you declare Mutex as static, its lifetime is static, and Arc is not needed
    static ref METRICS: Mutex<HashMap<Cow<'static, str>, usize>> =
        Mutex::new(HashMap::new());
}

fn main() {
    // Use Arc to provide shared ownership in concurrent environments (using reference counting)
    let metrics: Arc<Mutex<HashMap<Cow<'static, str>, usize>>> =
        Arc::new(Mutex::new(HashMap::new()));
    for _ in 0..32 {
        let m = metrics.clone();
        thread::spawn(move || {
            let mut g = m.lock().unwrap();
            // At this time, only the thread that obtained MutexGuard can access the HashMap
            let data = &mut *g;
            // Cow implements the From trait for many data structures,
            // so we can use "hello".into() to generate a Cow
            let entry = data.entry("hello".into()).or_insert(0);
            *entry += 1;
            // MutexGuard is dropped, and the lock is released
        });
    }

    thread::sleep(Duration::from_millis(100));

    println!("metrics: {:?}", metrics.lock().unwrap());
}
```

If you’re wondering how thread safety of the lock is ensured — for example, if thread 1 acquires the lock and then moves `MutexGuard` to thread 2 for use, then locking and unlocking would happen in completely different threads, creating a high risk of deadlock. What to do?

Don’t worry — `MutexGuard` does not allow `Send`, only `Sync`, which means you can pass a reference of `MutexGuard` to another thread for use, but you cannot move the entire `MutexGuard` to another thread:

``` rust
impl<T: ?Sized> !Send for MutexGuard<'_, T> {}
unsafe impl<T: ?Sized + Sync> Sync for MutexGuard<'_, T> {}
```

Smart pointers similar to `MutexGuard` have many uses. For example, if you want to create a connection pool, you can use the Drop trait to recycle checked-out connections and put them back into the pool. If you’re interested in this, you can look at the implementation of [r2d2](https://github.com/sfackler/r2d2/blob/master/src/lib.rs#L611), which is a database connection pool implementation in Rust.



## Implementing Your Own Smart Pointer

So far, we’ve covered three classic smart pointers — `Box<T>` for creating memory on the heap, `Cow<'a, B>` for providing copy-on-write, and `MutexGuard<T>` for data locking — along with how they’re implemented and used.

Now, what if we want to implement our own smart pointer? Or let’s ask the question differently: what kind of data structures are suitable to be implemented as smart pointers?

Many times, **we need to implement automatically optimized data structures** — ones that use specialized structures and algorithms in certain situations, and general ones in others.

For example, when the content of a `HashSet` is small, we could implement it with an array; as the content grows, we could switch to a hash table. If we want users not to worry about these implementation details and still get good performance through the same interface, we can consider using a smart pointer to unify this behavior.

## Practice Exercise

Let’s look at a practical example. As mentioned earlier, in Rust, a `String` occupies 24 bytes on the stack and stores its actual content on the heap. For short strings, this wastes memory. Is there a way to use the standard string type only when the string length exceeds a certain threshold?

Referring to `Cow`, we can handle this with an `enum`: when the string is smaller than N bytes, we store it directly in an array on the stack; otherwise, we use a `String`. But this N shouldn’t be too large — otherwise, when we use `String`, it would waste more memory than the current version.

How should we design it? As we discussed in the memory management section earlier, when using an enum, the extra tag plus padding for alignment takes up some memory. Since the `String` structure is aligned to 8 bytes, our enum would be at least 8 + 24 = 32 bytes.

Therefore, we can design a data structure that **uses one byte to represent the string’s length, 30 bytes to represent the string content, and one byte for the tag — exactly 32 bytes, the same as `String`. This allows both to fit neatly into one enum**. Let’s call this enum `MyString`. Its structure looks like the following diagram:

![MyString memory layout](images/rust-quick/rust-15-03.webp)

To make `MyString` behave the same as `&str`, we can implement the `Deref` trait so that `MyString` can be dereferenced into `&str`. In addition, we can implement the `Debug`/`Display` and `From<T>` traits to make `MyString` easier to use.

The full implementation code is as follows (code). It’s not hard to understand — you can try implementing it yourself or type it out line by line to see how it works.

``` rust
use std::{fmt, ops::Deref, str};

const MINI_STRING_MAX_LEN: usize = 30;

// In MyString, String has 3 words, totaling 24 bytes, so it’s aligned to 8 bytes.
// Therefore, the enum’s tag + padding take at least 8 bytes, and the whole structure occupies 32 bytes.
// MiniString can hold up to 30 bytes (plus 1 byte for length and 1 byte for tag), also totaling 32 bytes.
struct MiniString {
    len: u8,
    data: [u8; MINI_STRING_MAX_LEN],
}

impl MiniString {
    // The new interface is not exposed externally, ensuring the byte length of v ≤ 30
    fn new(v: impl AsRef<str>) -> Self {
        let bytes = v.as_ref().as_bytes();
        // When copying content, we must use the string’s byte length
        let len = bytes.len();
        let mut data = [0u8; MINI_STRING_MAX_LEN];
        data[..len].copy_from_slice(bytes);
        Self {
            len: len as u8,
            data,
        }
    }
}

impl Deref for MiniString {
    type Target = str;

    fn deref(&self) -> &Self::Target {
        // Since the interface for generating MiniString is hidden and only comes from strings,
        // the following line is safe.
        str::from_utf8(&self.data[..self.len as usize]).unwrap()
        // Alternatively, you can directly use the unsafe version:
        // unsafe { str::from_utf8_unchecked(&self.data[..self.len as usize]) }
    }
}

impl fmt::Debug for MiniString {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // Since we implemented the Deref trait, we can directly get a &str for output
        write!(f, "{}", self.deref())
    }
}

#[derive(Debug)]
enum MyString {
    Inline(MiniString),
    Standard(String),
}

// Implement the Deref trait to uniformly get &str in both cases
impl Deref for MyString {
    type Target = str;

    fn deref(&self) -> &Self::Target {
        match *self {
            MyString::Inline(ref v) => v.deref(),
            MyString::Standard(ref v) => v.deref(),
        }
    }
}

impl From<&str> for MyString {
    fn from(s: &str) -> Self {
        match s.len() > MINI_STRING_MAX_LEN {
            true => Self::Standard(s.to_owned()),
            _ => Self::Inline(MiniString::new(s)),
        }
    }
}

impl fmt::Display for MyString {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.deref())
    }
}

fn main() {
    let len1 = std::mem::size_of::<MyString>();
    let len2 = std::mem::size_of::<MiniString>();
    println!("Len: MyString {}, MiniString: {}", len1, len2);

    let s1: MyString = "hello world".into();
    let s2: MyString = "This is a very long string that exceeds the mini string limit".into();

    println!("s1: {:?}, s2: {:?}", s1, s2);

    println!(
        "s1: {}({} bytes, {} chars), s2: {}({} bytes, {} chars)",
        s1,
        s1.len(),
        s1.chars().count(),
        s2,
        s2.len(),
        s2.chars().count()
    );

    // MyString can use all &str interfaces, thanks to Rust’s automatic Deref
    assert!(s1.ends_with("world"));
    assert!(s2.starts_with("This"));
}

```

This simple implementation of `MyString` behaves identically to `&str` whether its internal data is the purely stack-based `MiniString` version or the heap-based `String` version. It sacrifices just a little efficiency and memory to allow small strings to be stored efficiently on the stack and used seamlessly.

In fact, Rust has a third-party library called smartstring that implements this functionality. Our version isn’t the most memory-efficient — for `String`, it uses 8 extra bytes — but smartstring, through optimization, achieves the same goal using only 24 bytes (the same size as `String`). If you’re interested, you’re welcome to check out its [source code](https://github.com/bodil/smartstring).


## Summary

Today we introduced three important smart pointers, each with unique implementations and use cases. 

`Box<T>` creates memory on the heap and serves as the foundation for many other data structures.

`Cow` implements a clone-on-write structure that lets you obtain ownership of data only when necessary. The `Cow` structure is a classic example of using an enum to dispatch based on current state — you can even use a similar approach instead of trait objects for dynamic dispatch, achieving tens of times better performance.

If you want to handle resource management properly, `MutexGuard` is an excellent reference — it wraps the lock obtained from a `Mutex` and ensures that as soon as `MutexGuard` goes out of scope, the lock is released. If you’re building a resource pool, you can use an approach similar to `MutexGuard`.



