---
title: "Rust Standard Library Traits: Complete Guide to Memory & Type"

description: "Master Rust standard library traits: Clone/Copy/Drop for memory, Send/Sync for concurrency, From/Into for conversions, and Deref patterns."

summary: "Comprehensive guide to essential Rust standard library traits covering memory management (Clone, Copy, Drop), marker traits for thread safety (Send, Sync), type conversions (From, Into, AsRef), operator overloading (Deref, DerefMut), and display traits (Debug, Display, Default). Learn how traits enable flexible APIs and system extensibility."

date: 2025-09-23
series: ["Rust"]
weight: 1
tags: ["rust", "traits", "memory-management", "type-conversion", "standard-library"]
author: ["Garry Chen"]
cover:
  image: images/rust-14-00.webp
  hiddenInList: true
  caption: "Rust Standard Library Traits Overview"

---

When building software systems, after clarifying requirements we need to perform architectural analysis and design. In that process, properly defining and using traits will make the code structure highly extensible and make the system very flexible.

In real problem solving, **good use of these traits will make your code structure clearer, and reading/using the code will better match Rust ecosystem conventions**. For example, if a data structure implements the Debug trait, then when you want to print the data structure you can use `{:?}`; if your data structure implements `From<T>`, you can directly use `into()` to convert data.

## trait

The Rust standard library defines a large number of standard traits. Let’s count the ones we’ve already learned and see what we’ve accumulated:
* `Clone` / `Copy` trait — define deep-copy and shallow-copy behavior for data;
* `Read` / `Write` trait — define I/O read/write behavior;
* `Iterator` — defines iterator behavior;
* `Debug` — defines how data is displayed in debug form;
* `Default` — defines how a default value for a type is produced;
* `From<T>` / `TryFrom<T>` — define how data is converted between types.

We will learn several other important trait categories, including traits related to memory allocation/release, marker traits used to distinguish types and help the compiler with type-safety checks, traits for type conversion, operator-related traits, and `Debug`/`Display`/`Default`.

## Memory-related: `Clone` / `Copy` / `Drop`

First let’s look at memory-related traits: `Clone` / `Copy` / `Drop`. These three traits were already encountered when [we]({{< ref "rust-quickly-start-07.md" >}}) [introduced]({{< ref "rust-quickly-start-08.md" >}}) [ownership]({{< ref "rust-quickly-start-09.md" >}}); here we’ll take a deeper look at their definitions and use cases.

### `Clone` trait

First, Clone:

```rust
pub trait Clone {
  fn clone(&self) -> Self;

  fn clone_from(&mut self, source: &Self) {
    *self = source.clone()
  }
}
```

The Clone trait has two methods, `clone()` and `clone_from()`; the latter has a default implementation, so usually we only need to implement `clone()`. You may wonder what `clone_from()` is for, since `a.clone_from(&b)` seems equivalent to `a = b.clone()`.

Actually they are not identical. If `a` already exists and `clone` would allocate during the operation, **using `a.clone_from(&b)` can avoid allocations and improve efficiency**.

`Clone` can be implemented via the derive macro to simplify code. If every field in your data structure already implements `Clone`, you can use `#[derive(Clone)]`. See the code below defining `Developer` struct and `Language` enum:

```rust
#[derive(Clone, Debug)]
struct Developer {
  name: String,
  age: u8,
  lang: Language
}

#[allow(dead_code)]
#[derive(Clone, Debug)]
enum Language {
  Rust,
  TypeScript,
  Elixir,
  Haskell
}

fn main() {
    let dev = Developer {
        name: "Tyr".to_string(),
        age: 18,
        lang: Language::Rust
    };
    let dev1 = dev.clone();
    println!("dev: {:?}, addr of dev name: {:p}", dev, dev.name.as_str());
    println!("dev1: {:?}, addr of dev1 name: {:p}", dev1, dev1.name.as_str())
}
```

If `Language` did not implement `Clone`, deriving `Clone` for `Developer` would fail to compile. Running this code shows that for name (a String), the heap memory is also cloned, so `Clone` is a deep copy — both stack and heap contents are copied.

Note that `clone`’s signature is `&self`, which in most cases is sufficient: to clone data we only need a read-only reference to the existing data. But for types like `Rc<T>`, whose `clone()` updates internal reference counts, the `clone()` call mutates internal state; such types use interior mutability (e.g., Cell<T>/RefCell<T>) to perform the mutation. If you have similar needs, refer to those patterns.

### `Copy` trait

Unlike `Clone`, `Copy` has no methods — it is a marker trait. Its definition is:

```rust
pub trait Copy: Clone {}
```

From this definition, to implement `Copy` a type must also implement `Clone`, then you implement an empty `Copy` trait. You might ask: what’s the point of a trait with no behavior?

**Although it has no methods, it can be used as a trait bound for type-safety checks** — hence it’s called a marker trait.

Like `Clone`, if all fields of a data structure implement `Copy`, you can derive `Copy` via `#[derive(Copy)]`. Try adding `Copy` to `Developer` and `Language`:

```rust
#[derive(Clone, Copy, Debug)]
struct Developer {
  name: String,
  age: u8,
  lang: Language
}

#[derive(Clone, Copy, Debug)]
enum Language {
  Rust,
  TypeScript,
  Elixir,
  Haskell
}
```

This code will fail: `String` does not implement `Copy`. Therefore `Developer` can only clone, not copy. Recall: if a type implements `Copy`, assignments and function calls will copy the value; otherwise ownership is moved.

So in the above, `Developer` will use move semantics when passed as a parameter, whereas `Language` will use copy semantics.

Earlier, when discussing mutable/immutable references, we mentioned that immutable references implement `Copy` while mutable references `&mut T` do not. Why is that?

If mutable references implemented `Copy`, creating a mutable reference and assigning it to another variable would violate ownership rules: there must be at most one mutable reference in the same scope. The Rust standard library carefully chooses which types can implement `Copy` and which cannot.

### `Drop` trait

We already covered `Drop` in the [memory-management]({{< ref "rust-quickly-start-11.md" >}}) section. Here is its definition again:

```rust
pub trait Drop {
    fn drop(&mut self);
}
```

In most cases you don’t need to implement `Drop` for your types: the system will automatically call `drop` for each field in a data structure in order. But there are two situations where you might implement `Drop` manually.

First, when you want to perform some action when data’s lifetime ends, e.g., logging.

Second, when you need to release external resources. The compiler doesn’t know about extra resources you may hold, so it can’t drop them for you. For example, lock release is implemented via `Drop` in `MutexGuard<T>`:

```rust
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

Note that the `Copy` trait and the `Drop` trait are mutually exclusive; the two cannot coexist. When you try to implement both `Copy` and `Drop` for the same data type, the compiler will give an error. This is easy to understand: **`Copy` performs a shallow bitwise copy, assuming that the copied data has no resources that need to be released; whereas `Drop` exists precisely to release extra resources**.

Let’s write some code to help understand this. In the code, we forcibly use `Box::into_raw` to get the pointer to heap memory and put it into the `RawBuffer` struct, so that we take over the responsibility of releasing this heap memory.

Although `RawBuffer` can implement the `Copy` trait, it then cannot implement the `Drop` trait. If the program insists on writing it this way, it will lead to a memory leak because the heap memory that should be released is not released.

However, this operation will not break Rust’s correctness guarantees: even if you Copy N copies of `RawBuffer`, since Drop cannot be implemented, the heap memory pointed to by `RawBuffer` will not be released, so there will be no use-after-free memory safety issue.

```rust
use std::{fmt, slice};

// Note here, we implemented Copy because *mut u8 / usize both support Copy
#[derive(Clone, Copy)]
struct RawBuffer {
    // Raw pointers use *const / *mut, which is different from & references
    ptr: *mut u8,
    len: usize,
}

impl From<Vec<u8>> for RawBuffer {
    fn from(vec: Vec<u8>) -> Self {
        let slice = vec.into_boxed_slice();
        Self {
            len: slice.len(),
            // After into_raw, Box no longer manages this memory; RawBuffer must handle the release
            ptr: Box::into_raw(slice) as *mut u8,
        }
    }
}

// If RawBuffer implements Drop trait, heap memory can be released when the owner goes out of scope
// However, Drop trait conflicts with Copy trait; either don’t implement Copy or don’t implement Drop
// If Drop is not implemented, it causes a memory leak, but it won’t break correctness
// For example, there will be no use-after-free problem.
// You can try uncommenting below to see what happens
// impl Drop for RawBuffer {
//     #[inline]
//     fn drop(&mut self) {
//         let data = unsafe { Box::from_raw(slice::from_raw_parts_mut(self.ptr, self.len)) };
//         drop(data)
//     }
// }

impl fmt::Debug for RawBuffer {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let data = self.as_ref();
        write!(f, "{:p}: {:?}", self.ptr, data)
    }
}

impl AsRef<[u8]> for RawBuffer {
    fn as_ref(&self) -> &[u8] {
        unsafe { slice::from_raw_parts(self.ptr, self.len) }
    }
}

fn main() {
    let data = vec![1, 2, 3, 4];

    let buf: RawBuffer = data.into();

    // Because buf allows Copy, we copy it here
    use_buffer(buf);

    // buf can still be used
    println!("buf: {:?}", buf);
}

fn use_buffer(buf: RawBuffer) {
    println!("buf to die: {:?}", buf);

    // No need to explicitly drop; written here just to show that the copied buf was dropped
    drop(buf)
}

```

When it comes to code safety, which is more harmful: memory leaks or use-after-free? It is definitely the latter. Rust’s bottom line is memory safety, so it chooses the lesser evil.

In fact, no programming language can guarantee no memory leaks caused by human oversight. For example, if a developer forgets to delete entries in a hash map during runtime, it leads to memory leaks. But Rust ensures that even if the developer makes such mistakes, no memory safety issues like use-after-free occur.


## Marker traits: `Sized` / `Send` / `Sync` / `Unpin`

Okay, after discussing memory-related traits, let’s look at marker traits.

We’ve already seen one marker trait: `Copy`. Rust also supports other marker traits: `Sized` / `Send` / `Sync` / `Unpin`.

Sized trait marks types with a concrete size. When using generic parameters, Rust automatically adds a Sized constraint to generic parameters, for example with `Data<T>` and the function `process_data` handling `Data`:

```rust
struct Data<T> {
    inner: T,
}

fn process_data<T>(data: Data<T>) {
    todo!();
}
```

This is equivalent to:

```rust
struct Data<T: Sized> {
    inner: T,
}

fn process_data<T: Sized>(data: Data<T>) {
    todo!();
}
```

Most of the time, we want this constraint added automatically, because with it, the generic structure’s size is fixed at compile time, allowing it to be passed as a function parameter. Without it, if `T` is a dynamically sized type, `process_data` won’t compile.

However, this automatically added constraint is not always suitable. **In rare cases, we want `T` to be dynamically sized. What then? Rust provides `?Sized` to lift this constraint**.

If the developer explicitly defines `T: ?Sized`, then `T` can be any size. If you remember the `Cow` type from [before]({{< ref "rust-quickly-start-12.md" >}}), `Cow`’s generic parameter `B` uses `?Sized`:

```rust
pub enum Cow<'a, B: ?Sized + 'a> where B: ToOwned,
{
    // Borrowed data
    Borrowed(&'a B),
    // Owned data
    Owned(<B as ToOwned>::Owned),
}
```

This way `B` can be `[T]` or `str` types, both dynamically sized. Note that `Borrowed(&'a B)` itself has a fixed size because internally it’s a reference, and references have a fixed size.


### `Send` / `Sync`

Next, let’s look at `Send` / `Sync`. Their definitions are:

```rust
pub unsafe auto trait Send {}
pub unsafe auto trait Sync {}
```

Both traits are `unsafe` `auto` traits, `auto` means the compiler automatically adds them where appropriate. `Unsafe` means manually implementing them requires the developer to ensure memory safety.

`Send`/`Sync` are the foundation of concurrency safety in Rust:
* If a type `T` implements the `Send` trait, it means `T` can be safely moved from one thread to another, i.e., its ownership can be transferred between threads.

* If a type `T` implements the `Sync` trait, it means `&T` can be safely shared among multiple threads. A type `T` satisfies the `Sync` trait if and only if `&T` satisfies the `Send` trait.

For the role of `Send`/`Sync` in thread safety, we can look at it this way: **if a type `T: Send`, then exclusive access to `T` within a thread is thread-safe; if a type `T: Sync`, then read-only sharing of `T` across threads is safe**.

For user-defined data structures, if all of their internal fields implement `Send` / `Sync`, then the data structure itself will automatically implement `Send` / `Sync`. Basically, native data structures all support `Send` / `Sync`, so the vast majority of custom data structures also satisfy `Send` / `Sync`. In the standard library, data structures that do not support `Send` / `Sync` mainly include:

* Raw pointers `*const T` / `*mut T`. They are unsafe, so they are neither `Send` nor `Sync`.
* `UnsafeCell<T>` does not support `Sync`. That is, any data structure using `Cell` or `RefCell` does not support `Sync`.
* Reference counting `Rc` does not support `Send` nor `Sync`, so `Rc` cannot be used across threads.


[Previously]({{< ref "rust-quickly-start-09.md" >}}) introduced `Rc` / `RefCell`, let’s see what happens if we try to use `Rc` / `RefCell` across threads.
In Rust, if you want to create a new thread, you need to use `std::thread::spawn`:

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T> 
where
    F: FnOnce() -> T,
    F: Send + 'static,
    T: Send + 'static,
```

Its parameter is a closure, and this closure needs `Send + 'static`:
* `'static` means the free variables captured by the closure must be either a type owning its value, or a reference with a static lifetime;
* `Send` means the ownership of those captured free variables can be moved from one thread to another.

From this interface, we can conclude: if passing `Rc` across threads, it will not compile, because `Rc` does not implement `Send` and `Sync`. Let’s write code to verify:
``` rust
// Rc is neither Send nor Sync
fn rc_is_not_send_and_sync() {
    let a = Rc::new(1);
    let b = a.clone();
    let c = a.clone();
    thread::spawn(move || {
        println!("c= {:?}", c);
    });
}
```

Sure enough, this code does not compile.

![Rc is neither Send nor Sync](images/rust-14-01.webp)

So, can `RefCell<T>` transfer ownership across threads? `RefCell` implements `Send`, but not `Sync`, so it looks like it should work:

```rust
fn refcell_is_send() {
    let a = RefCell::new(1);
    thread::spawn(move || {
        println!("a= {:?}", a);
    });
}
```

Verification shows this works.

Since `Rc` cannot `Send`, we cannot use `Rc<RefCell<T>>` across threads. Then what about using `Arc`, which supports `Send`/`Sync`, to get a type that can be shared and modified across threads: `Arc<RefCell<T>>`, will that work?

```rust
// RefCell now has multiple Arc owners; although Arc is Send/Sync, RefCell is not Sync
fn refcell_is_not_sync() {
    let a = Arc::new(RefCell::new(1));
    let b = a.clone();
    let c = a.clone();
    thread::spawn(move || {
        println!("c= {:?}", c);
    });
}
```

It does not work.

Because the data inside `Arc` is shared, it requires a `Sync` data structure, but `RefCell` is not `Sync`, so compilation fails. Therefore, in multithreaded scenarios, we can only use `Arc` with `Send`/`Sync` support, together with `Mutex`, to construct a type that can be shared and modified across threads:

``` rust
use std::{
    sync::{Arc, Mutex},
    thread,
};

// Arc<Mutex<T>> allows sharing and mutating data across threads
fn arc_mutext_is_send_sync() {
    let a = Arc::new(Mutex::new(1));
    let b = a.clone();
    let c = a.clone();
    let handle = thread::spawn(move || {
        let mut g = c.lock().unwrap();
        *g += 1;
    });

    {
        let mut g = b.lock().unwrap();
        *g += 1;
    }

    handle.join().unwrap();
    println!("a= {:?}", a);
}

fn main() {
    arc_mutext_is_send_sync();
}
```

It is recommended you read and run all these code examples. For compilation errors, carefully read the compiler error messages; they will help you understand the Send/Sync traits and how they guarantee thread safety.


## Type conversion related: `From<T>` / `Into<T>` / `AsRef<T>` / `AsMut<T>`

OK, after learning about marker traits, let’s look at traits related to type conversion. In software development, we often need to convert one data structure into another in some context.

But there are many ways to do conversions. Look at the code below: which way do you think is better?
```rust
// First method: provide a method for each conversion
// Convert string s to Path
let v = s.to_path();
// Convert string s to u64
let v = s.to_u64();

// Second method: implement an Into<T> trait between s and the target type
// v's type is inferred from context
let v = s.into();
// Or explicitly annotate v's type
let v: u64 = s.into();
```

In the first method, inside the implementation of type `T`, we need to provide a method for each possible conversion; In the second, we implement a data conversion trait between type `T` and type `U`, so the same method can be used for different conversions.

Obviously, the second method is better, because it follows the Open-Close Principle: “**Objects in software (classes, modules, functions, etc.) should be open for extension, but closed for modification**.”

In the first method, every time we add a new conversion target, we need to modify type T’s implementation. But in the second, we only need to add a new implementation for the data conversion trait.

Based on this idea, for conversions between value types and reference types, Rust provides two different sets of traits:
* Value-to-value conversions: `From<T>` / `Into<T>` / `TryFrom<T>` / `TryInto<T>`
* Reference-to-reference conversions: `AsRef<T>` / `AsMut<T>`


### `From<T>` / `Into<T>`

Let’s first look at `From<T>` and `Into<T>`. Their definitions are as follows:

```rust
pub trait From<T> {
    fn from(T) -> Self;
}

pub trait Into<T> {
    fn into(self) -> T;
}
```

When you implement `From<T>`, `Into<T>` is automatically implemented. This is because:

```rust
// Implementing From automatically implements Into
impl<T, U> Into<U> for T where U: From<T> {
    fn into(self) -> U {
        U::from(self)
    }
}
```

So in most cases, just implementing `From<T>` is enough, and both of these conversion methods will then work. For example:

```rust
let s = String::from("Hello world!");
let s: String = "Hello world!".into();
```

These two ways are equivalent. Which to choose? `From<T>` can do type inference based on context and has more use cases; Also, since implementing `From<T>` automatically implements `Into<T>` but not vice versa, you should only implement `From<T>` when needed, not `Into<T>`.

In addition, `From<T>` and `Into<T>` are reflexive: Converting a value of type `T` into type `T` directly returns it. This is because the standard library provides the following implementation:

```rust

impl<T> From<T> for T {
    fn from(t: T) -> T {
        t
    }
}
```


With `From<T>` and `Into<T>`, many function interfaces can become more flexible. For example, if a function takes an IpAddr as a parameter, we can use `Into<IpAddr>` to allow more types to be used with this function. See the code below:

```rust
use std::net::{IpAddr, Ipv4Addr, Ipv6Addr};

fn print(v: impl Into<IpAddr>) {
    println!("{:?}", v.into());
}

fn main() {
    let v4: Ipv4Addr = "2.2.2.2".parse().unwrap();
    let v6: Ipv6Addr = "::1".parse().unwrap();
    
    // IPAddr implements From<[u8; 4] to convert IPv4 address
    print([1, 1, 1, 1]);
    // IPAddr implements From<[u16; 8] to convert IPv6 address
    print([0xfe80, 0, 0, 0, 0xaede, 0x48ff, 0xfe00, 0x1122]);
    // IPAddr implements From<Ipv4Addr>
    print(v4);
    // IPAddr implements From<Ipv6Addr>
    print(v6);
}
```

So, using `From<T>` / `Into<T>` reasonably can make code concise, match Rust’s style of strong readability, and better follow the Open-Close Principle.

Note, if your data type may have errors during conversion, you can use `TryFrom<T>` and `TryInto<T>`. They work the same as `From<T>` / `Into<T>`, except the trait has an associated type `Error`, and the returned result is `Result<T, Self::Error>`.


### `AsRef` / `AsMut`

After understanding `From<T>` / `Into<T>`, `AsRef<T>` and `AsMut<T>` are easy to understand. They are used for conversions from reference to reference. Let’s first look at their definitions:

```rust
pub trait AsRef<T> where T: ?Sized {
    fn as_ref(&self) -> &T;
}

pub trait AsMut<T> where T: ?Sized {
    fn as_mut(&mut self) -> &mut T;
}
```

In the trait definitions, T is allowed to be dynamically sized types, such as `str`, `[u8]`, etc. `AsMut<T>` is the same as `AsRef<T>` except it generates a mutable reference from a mutable reference, so we mainly focus on `AsRef<T>`.

Look at the standard library interface for opening files: `std::fs::File::open`:

``` rust
pub fn open<P: AsRef<Path>>(path: P) -> Result<File>
```


Its parameter `path` is a type that conforms to `AsRef<Path>`, so you can pass `String`, `&str`, `PathBuf`, `Path`, and other types to this parameter. Moreover, when you use `path.as_ref()`, you get a `&Path`.

Let’s write a piece of code to experience the usage and implementation of `AsRef<T>`:

```rust
#[allow(dead_code)]
enum Language {
    Rust,
    TypeScript,
    Elixir,
    Haskell,
}

impl AsRef<str> for Language {
    fn as_ref(&self) -> &str {
        match self {
            Language::Rust => "Rust",
            Language::TypeScript => "TypeScript",
            Language::Elixir => "Elixir",
            Language::Haskell => "Haskell",
        }
    }
}

fn print_ref(v: impl AsRef<str>) {
    println!("{}", v.as_ref());
}

fn main() {
    let lang = Language::Rust;
    // &str implements AsRef<str>
    print_ref("Hello world!");
    // String implements AsRef<str>
    print_ref("Hello world!".to_string());
    // The enum we defined ourselves also implements AsRef<str>
    print_ref(lang);
}
```

Now, with a deeper understanding of how to use `From` / `Into` / `AsRef` / `AsMut` for type conversions in Rust, we will encounter these traits again in real-world practice in the future.

One additional point about the small example above: if your code has a statement like `v.as_ref().clone()`, meaning you first convert `v` by reference and then obtain an owned value, you should implement `From<T>` and then use `v.into()`.


## Operator-related: `Deref` / `DerefMut`

Operator-related traits — in the previous lecture we saw the `Add<Rhs>` trait, which allows you to overload the addition operator. Rust provides traits for all operators, and you can overload certain operators for your own types.

Here we use a diagram to briefly summarize them. For more detailed information, you can read the official documentation.

![Operator traits](images/rust-14-02.webp)

The key operator traits to introduce are `Deref` and `DerefMut`. Let’s look at their definitions:

```rust
pub trait Deref {
    // The type of the dereferenced result
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}

pub trait DerefMut: Deref {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

As you can see, `DerefMut` “inherits” from `Deref`, but additionally provides a `deref_mut` method to get a mutable dereference. So here we mainly learn `Deref`.

For ordinary references, dereferencing is intuitive because it only has one address pointing to the value, and from this address you can get the desired value, as in the following example:

```rust
let mut x = 42;
let y = &mut x;
// Dereferencing internally calls DerefMut (its implementation is *self)
*y += 1;
```

But for smart pointers, which field to dereference is not so intuitive. Let’s look at how the `Rc` we learned earlier implements `Deref`:

```rust
impl<T: ?Sized> Deref for Rc<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.inner().value
    }
}
```

You can see it ultimately points to the value inside the heap’s RcBox, and when dereferenced, you get the value corresponding to `value`. As shown in the diagram, the final print result is v = 1.

![Dereferencing Rc](images/rust-14-03.webp)

From the diagram, you can also see that `Deref` and `DerefMut` are called automatically. `*b` is expanded to `*(b.deref())`.

In Rust, most smart pointers implement `Deref`, and we can also implement `Deref` for our own data structures. Here is an example:
```rust
use std::ops::{Deref, DerefMut};

#[derive(Debug)]
struct Buffer<T>(Vec<T>);

impl<T> Buffer<T> {
    pub fn new(v: impl Into<Vec<T>>) -> Self {
        Self(v.into())
    }
}

impl<T> Deref for Buffer<T> {
    type Target = [T];

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

impl<T> DerefMut for Buffer<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.0
    }
}

fn main() {
    let mut buf = Buffer::new([1, 3, 2, 4]);
    // Because we implemented Deref and DerefMut, buf can directly access Vec<T> methods
    // This line is equivalent to: (&mut buf).deref_mut().sort(), i.e., (&mut buf.0).sort()
    buf.sort();
    println!("buf: {:?}", buf);
}
```

But in this example, the data structure `Buffer<T>` wraps `Vec<T>`, so many methods originally implemented on `Vec<T>` become inconvenient to use since we have to access them with `buf.0`. What to do?

We can implement `Deref` and `DerefMut`, so that during dereferencing, we directly access `buf.0`, eliminating verbosity and hiding of internal fields.

Another point to note in this code: when calling `buf.sort()`, no explicit dereferencing is performed. Why is it equivalent to calling `buf.0.sort()`? That’s because the `sort()` method’s first parameter is `&mut self`, and at this point the Rust compiler forces `Deref`/`DerefMut` dereferencing. So it’s equivalent to `(*(&mut buf)).sort()`.


## Others: `Debug` / `Display` / `Default`

Now that we have a good understanding of operator-related traits, let’s look at some other common traits: `Debug` / `Display` / `Default`.

First, `Debug` / `Display` — their definitions are as follows:

```rust
pub trait Debug {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result<(), Error>;
}

pub trait Display {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result<(), Error>;
}
```

You can see `Debug` and `Display` have the same signature: they both take a `&self` and a `&mut Formatter`. So why do we need two identical traits?

Because `Debug` is designed for developers to debug-print data structures, while `Display` is designed to show data structures to users. That’s why the `Debug` trait implementation can be derived automatically, while `Display` must be implemented manually. When using them, `Debug` prints with `{:?}`, and `Display` prints with `{}`.

Finally, let’s look at the `Default` trait. Its definition is:

```rust
pub trait Default {
    fn default() -> Self;
}
```

The `Default` trait provides default values for types. It can also be derived via `#[derive(Default)]` if all fields in the type implement Default. When initializing a data structure, we can partially initialize it and use `Default::default()` for the rest.

How to use `Debug`/`Display`/`Default`? Let’s see an example:

```rust
use std::fmt;
// struct can derive Default if all fields implement Default
#[derive(Clone, Debug, Default)]
struct Developer {
    name: String,
    age: u8,
    lang: Language,
}

// enum cannot derive Default
#[allow(dead_code)]
#[derive(Clone, Debug)]
enum Language {
    Rust,
    TypeScript,
    Elixir,
    Haskell,
}

// Implement Default manually
impl Default for Language {
    fn default() -> Self {
        Language::Rust
    }
}

impl Developer {
    pub fn new(name: &str) -> Self {
        // Use ..Default::default() for remaining fields
        Self {
            name: name.to_owned(),
            ..Default::default()
        }
    }
}

impl fmt::Display for Developer {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(
            f,
            "{}({} years old): {:?} developer",
            self.name, self.age, self.lang
        )
    }
}

fn main() {
    // Use T::default()
    let dev1 = Developer::default();
    // Use Default::default(), but type cannot be inferred, must be specified
    let dev2: Developer = Default::default();
    // Use T::new
    let dev3 = Developer::new("Tyr");
    println!("dev1: {}\ndev2: {}\ndev3: {:?}", dev1, dev2, dev3);
}
```

## Summary

Today we introduced basic traits related to memory management, type conversion, operators, and data display, as well as marker traits — a special kind of trait mainly used to help the compiler check type safety.

![trait summary](images/rust-14-04.webp)

When developing with Rust, traits play a very central role. **A well-designed trait can greatly improve the usability and extensibility of the entire system**.

Many excellent third-party libraries build their capabilities around traits, such as the Service trait in tower-service mentioned earlier, and the Parser trait in the parser combinator library nom that you may frequently use in the future.

Because traits implement late binding. As mentioned in the discussion of programming fundamentals before, late binding brings great flexibility in software development.

From the perspective of data, data structures are late binding of concrete data, generic structures are late binding of concrete data structures; from the perspective of code, functions are late binding of a set of expressions implementing some functionality, generic functions are late binding of functions. So what is trait the late binding of?

**Trait is the late binding of behavior**. We can, without knowing the specific data structures to handle, first agree on many system behaviors through traits. This is why when explaining standard traits at the beginning, we often used the phrase “specify behavior.”


