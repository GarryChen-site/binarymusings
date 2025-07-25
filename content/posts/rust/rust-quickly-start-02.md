---
title: "Rust Programming Concepts: Types, Pointers, Async & Generics"

description: "Master core programming concepts in Rust: data types, pointers/references, functions/closures, interfaces, concurrency, async programming, and generics."

summary: "Explore fundamental programming concepts essential for Rust development. Learn about primitive and composite data types, pointers vs references, functions and closures, interfaces and virtual tables, concurrency vs parallelism, async/await patterns, and generic programming paradigms. Build a solid foundation for understanding Rust's ownership model, dynamic dispatch, and concurrent processing."

date: 2025-07-25
series: ["Rust"]
weight: 1
tags: ["rust", "programming-concepts", "data-types", "async-programming", "generics"]
author: ["Garry Chen"]
cover:
  image: images/rust-02-00.webp
  hiddenInList: true
  caption: "Concepts"

---

In the previous lecture, we explored the basic workings of memory. To recap briefly: data stored on the stack is static, with a fixed size and a fixed lifetime, whereas data stored on the heap is dynamic, with variable size and variable lifetime.

Today, we will continue to outline other basic concepts frequently encountered in programming development. There are many small concepts to grasp, so I’ve categorized them into four main sections for easier understanding: Data (values and types, pointers and references), Code (functions, methods, closures, interfaces, and virtual tables), Execution (concurrency and parallelism, synchronous and asynchronous operations, and Promise/async/await), and Programming Paradigms (generic programming).

By revisiting these concepts, I hope you can solidify your foundational knowledge in software development. This is crucial for understanding many challenging aspects in Rust, such as ownership, dynamic dispatch, and concurrent processing.

# Data
Data is the object that programs manipulate. A program that does not process data is meaningless. Let’s first review concepts related to data, including values and types, pointers, and references.

# Values and Types
Strictly speaking, a type distinguishes between values. It includes information about the length of the value in memory, alignment, and the operations that can be performed on the value. A value is an instance of data that conforms to a specific type. For example, 64u8 is of the u8 type, which corresponds to an integer entity of one byte in size, with a value range of 0 to 255. The specific entity here is 64.

Values are stored as a sequence of bytes according to the representation defined by their type. For instance, the value 64 is stored in memory as 0x40 or 0b01000000.

It is important to note that values cannot be discussed independently of their specific types. The same byte 0x40 in memory represents different things depending on its type. If its type is ASCII char, it represents the `@` symbol rather than the number 64.

Regardless of whether a language is strongly typed or weakly typed, it has specific internal representations of types. Generally, types in programming languages can be divided into primitive types and composite types.

Primitive Types: These are the most basic data types provided by a programming language. Examples include characters, integers, floating-point numbers, boolean values, arrays, tuples, pointers, references, functions, and closures. All primitive types have a fixed size, allowing them to be allocated on the stack.

Composite Types: Also known as complex types, these are composed of a combination of primitive and other types. Composite types can be further divided into:

* Structure Types: Complex data structures that combine multiple types to express a single value. For instance, a Person structure might contain name, age, and email fields. In terms of algebraic data types, a structure is a product type.

* Tagged Unions: Also called disjoint unions, these can store an object of one of several fixed types, determined by its tag. Examples include the Maybe type in Haskell or Optional in Swift. In algebraic data type terms, a tagged union is a sum type.

Many languages do not support tagged unions and instead offer enumerations (enums), which are a subset of tagged unions but less powerful and unable to express complex structures.

Understanding these definitions might be challenging, so refer to the following diagram for better clarity.

![Data Types](images/rust-02-01.webp)

# Pointers and References

In memory, a value is stored at a specific location, which corresponds to a memory address. A pointer is a value that holds this memory address, and it can be dereferenced to access the data at that address. Theoretically, a pointer can be dereferenced to any data type.

References are similar to pointers, but with restricted dereferencing capabilities. They can only dereference to the data type they refer to, and cannot be used otherwise. For instance, a reference pointing to the value 42u8 can only be dereferenced as the u8 data type.

Therefore, pointers have fewer usage restrictions but can lead to more risks. If a pointer is not dereferenced with the correct type, it can cause various memory issues, leading to system crashes or potential security vulnerabilities.

As mentioned, pointers and references are primitive types and can be allocated on the stack.

Depending on the type of data they point to, some references require not only a pointer to the memory address but also additional information like the memory address’s length.

For example, a pointer to the string “hello world” also includes the string’s length and capacity, using a total of three words, which occupy 24 bytes on a 64-bit CPU. Such pointers that carry more information than regular pointers are called fat pointers. Many data structure references are implemented using fat pointers.

# Code

Data is the object that programs manipulate, while code is the main body of program execution. It is also the medium through which developers convert physical world requirements into digital logic. We will discuss functions, closures, interfaces, and virtual tables.

# Functions, Methods, and Closures

A function is a fundamental element of programming languages, encapsulating a set of related statements and expressions that accomplish a specific task. Functions abstract repetitive behavior in code. In modern programming languages, functions are often first-class citizens, meaning they can be passed as parameters, returned as values, and included as part of composite types.

In object-oriented programming languages, functions defined within a class or object are called methods. Methods often relate to object pointers, such as the self reference in Python objects or the this reference in Java objects.

A closure is a data structure that stores a function along with its environment. Free variables referenced in the closure’s context are captured and become part of the closure’s type.

Generally, if a programming language treats functions as first-class citizens, it will support closures, as functions returned as values often need to return a closure.

Refer to this diagram to help understand how a closure captures its contextual environment.

![Closure](images/rust-02-02.webp)

# Interfaces and Virtual Tables

Interfaces are a core part of software system development, reflecting the system designer’s abstract understanding of the system. As an abstraction layer, interfaces isolate the user from the implementation, significantly enhancing reusability and extensibility.

Many programming languages have the concept of interfaces, allowing developers to design based on interfaces, such as Java’s interface, Elixir’s behavior, Swift’s protocol, and Rust’s trait.

For example, in HTTP, the Request/Response service handling model is a typical interface. By defining how to map from a Request to a Response under different inputs, the system can dispatch compliant Requests to our service in appropriate scenarios.

Designing based on interfaces is a crucial capability in software development, and Rust particularly emphasizes this. In later chapters on Traits, we will detail how to use Traits for interface design.

When using interfaces to reference specific types at runtime, the code gains runtime polymorphism. However, at runtime, once an interface reference is used, the original type of the variable is erased, and we cannot determine the capabilities of the reference just by analyzing a pointer.

Therefore, when generating this reference, we need to construct a fat pointer that, besides pointing to the data itself, also points to a list of methods supported by this interface. This list is what we know as the virtual table (vtable).

The diagram below shows the process of a Vec data type being type-erased at runtime, generating a reference to the Write interface.

![Virtual Table](images/rust-02-03.webp)

Because the virtual table records the interfaces that the data can execute, we can dynamically dispatch different implementations for an interface at runtime based on context.

For example, if we want to implement different formatting tools for various languages for a Formatter interface in an editor, we can load all supported languages and their formatting tools into a hash table at editor load time. The hash table’s key is the language type, and the value is a reference to each formatting tool that implements the Formatter interface. This way, when a user opens a file in the editor, we can find the corresponding Formatter reference based on the file type and perform formatting operations.

# Execution Modes

The way a program runs after it is loaded often determines its execution efficiency. Therefore, we will discuss concurrency, parallelism, synchronous and asynchronous execution, and several important concepts in asynchronous execution: Promise, async, and await.

# Concurrency and Parallelism

Concurrency and parallelism are concepts frequently encountered in software development.

Concurrency is the ability to handle multiple tasks at the same time. For instance, a system can work on Task 1 to a certain extent, save its context, suspend it, and switch to Task 2, then switch back to Task 1 after some time.

Parallelism is the technique of processing multiple tasks simultaneously. This means Task 1 and Task 2 can work in the same time slice without context switching. The following diagram illustrates the difference between the two.

![Concurrency vs Parallelism](images/rust-02-04.webp)

Concurrency is a capability, while parallelism is a technique. Once our system has the ability to handle concurrency, the code can run in parallel on multiple CPU cores. Therefore, we often talk about high-concurrency handling rather than high-parallel handling.

Many programming languages with high-concurrency capabilities embed an M:N scheduler in user programs, distributing M concurrent tasks reasonably across N CPU cores for parallel execution, maximizing program throughput.

# Synchronous and Asynchronous

Synchronous execution means that after a task starts, subsequent operations are blocked until the task is completed. In software, most of our code is synchronous. For example, in a CPU, the next instruction in the pipeline is executed only after the previous one is completed. Similarly, if Function A calls Function B and then Function C sequentially, it will execute Function C only after Function B is completed.

Synchronous execution ensures the causality of the code, which is essential for program correctness.

However, when dealing with I/O operations, the significant gap between efficient CPU instructions and inefficient I/O becomes a performance bottleneck. The following diagram compares the latency of CPU, memory, I/O devices, and networks.

![I/O Latency](images/rust-02-05.webp)

We can see that compared to memory access, I/O operations are two orders of magnitude slower. When encountering an I/O operation, the CPU can only idle while waiting for the I/O device to complete. Therefore, operating systems provide asynchronous I/O to applications, allowing them to utilize CPU time for other tasks while the current I/O operation is being processed.

Asynchronous execution means that after a task starts, other tasks that are not causally related can continue to execute without waiting for the previous task to complete.

In asynchronous operations, the results of the asynchronous processing are usually stored in a Promise. A Promise is an object that represents a value that will be available at some point in the future. Promises generally exist in three states:

1. Initial state: The Promise has not yet run.
2. Pending state: The Promise has run but not yet completed.
3. Fulfilled or Rejected state: The Promise either successfully resolves to a value or fails.

If you are not familiar with the term Promise, in many languages that support asynchronous operations, it is also called Future, Delay, or Deferred. Besides this term, you may often see the async/await pair of keywords.

Generally, async defines a task that can be executed concurrently, while await triggers the concurrent execution of the task. In most languages, async/await is syntactic sugar that uses a state machine to wrap Promises, making asynchronous calls feel similar to synchronous calls and making the code easier to read.

# Programming Paradigms

To better maintain code during continuous iteration, we introduce various programming paradigms to enhance code quality. Finally, let’s talk about generic programming.

If you come from weakly-typed languages like C, Python, or JavaScript, generic programming is a concept and skill you need to master. Generic programming involves two aspects: the generics of data structures and the generic usage of those data structures in code.

# Generics of Data Structures

First, let’s discuss generics in data structures, often referred to as parameterized types or parametric polymorphism. Consider the following data structure:

```rust
struct Connection<S> {
 io: S,
 state: State,
}
```

Here, it has a parameter S, and the type of the field io is S. The specific type of S is only determined in the context where Connection is used.

You can think of parameterized data structures as type-generating functions. When “called” with specific types, they return a type that carries those types. For example, if we provide TcpStream as the type for S, we get the Connection type, where the type of io is TcpStream.

You might wonder, if S can be any type, how do we know what behaviors S has? How does the compiler know that S contains the send method when we call `io.send()`?

This is a good question. We need to constrain S using interfaces. Therefore, languages that support generic programming often provide strong interface programming capabilities.

Generics in data structures provide a high level of abstraction. Just as humans use numbers to abstract the quantity of specific things and invent algebra to further abstract specific numbers, generics allow us to delay binding, making data structures more general and widely applicable. This also greatly reduces code duplication and improves maintainability.


# Generic Code

The other aspect of generic programming is writing generic code using these data structures. When we use generic structures, the related code also requires additional abstraction.

A familiar example of this is binary search

![Binary Search](images/rust-02-06.webp)

In the C implementation on the left, several operations are implicitly tied to `int[]`. Therefore, to perform binary search on different data types, the implementation must change accordingly. The C++ implementation on the right abstracts these operations, allowing us to use the same code for binary search on data types that support iterators.

Similarly, this code can be used in broader contexts, making it simpler and easier to maintain.


# Summary

Today, we discussed four categories of fundamental concepts: data, code, runtime methods, and programming paradigms.

Values cannot be discussed separately from types. Types are generally divided into primitive types and composite types. Pointers and references both point to the memory address of a value, but they behave differently when dereferencing. References can only dereference to the original data type, while pointers have no such restrictions. However, unconstrained pointer dereferencing can lead to memory safety issues.

Functions abstract repetitive behaviors in code, methods are functions defined within objects, and closures are special functions that capture free variables in their context as part of the closure.

Interfaces isolate callers from implementers, greatly promoting code reuse and extensibility. Interface-oriented programming makes systems flexible. When using interfaces to reference specific types, we need virtual tables to assist runtime code execution. Virtual tables enable dynamic dispatch, forming the basis of runtime polymorphism.

Concurrency is the foundation of parallelism, allowing multiple tasks to be handled simultaneously. Parallelism is the manifestation of concurrency, processing multiple tasks at the same time. Synchronous execution blocks subsequent operations, while asynchronous execution allows them. Promises, widely used in asynchronous operations, represent results available at a future time. async/await wraps promises, often implemented using state machines.

Generic programming uses parameterization to delay binding in data structures, enhancing generality. Type parameters can be constrained by interfaces to ensure certain behaviors. When using generic structures, our code also requires higher levels of abstraction.

---