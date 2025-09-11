---
title: "Rust Memory Management: Stack vs Heap & Memory Safety"

description: "Learn Rust memory fundamentals: stack vs heap allocation, memory safety, and how Rust prevents common memory issues without GC overhead."

summary: "Master Rust memory management fundamentals: understand stack vs heap allocation, memory safety principles, and how Rust eliminates memory leaks and use-after-free errors without garbage collection overhead."

date: 2025-07-24
series: ["Rust"]
weight: 1
tags: ["rust", "memory-management", "stack-heap", "memory-safety", "systems-programming"]
author: ["Garry Chen"]
cover:
  image: images/rust-01-00.webp
  hiddenInList: true
  caption: "Memory Management"

---

The most basic concept in code is variables and values, and the place where they are stored is memory, so we start with memory.

## Memory

Our programs constantly interact with memory. Even in a simple statement like assigning “hello world!” to s, there are deep interactions with the Read-Only Data segment (RODATA), the heap, and the stack:

``` rust
let s = "hello world".to_string();
```

Here’s a breakdown of the memory usage with a string example, which you can explore using a Rust playground snippet.

The string literal “hello world” is stored in the .RODATA segment (in GCC) or the .RDATA segment (in VC++) at compile time. Upon program load, it is assigned a fixed memory address.

When executing “hello world”.to_string(), new memory is allocated on the heap. The string “hello world” is copied byte by byte to this heap memory.

When the heap data is assigned to s, s is a variable on the stack. It needs to know the heap memory’s address. Because the heap data size is indeterminate and can grow, it also needs to know the current length and capacity of the data.

To represent this string, three words are used:

* First Word: Pointer to the heap memory.
* Second Word: Current length of the string (11 bytes).
* Third Word: Total capacity of the allocated memory (11 bytes).

In a 64-bit system, these three words occupy 24 bytes.

![string](/images/rust-01-01.webp)

Earlier, we mentioned that the content of a string is stored on the heap, while the pointer to the string and related information are stored on the stack. Now, it’s time to test your understanding of memory fundamentals: when should data be placed on the stack, and when should it go on the heap?

Many developers using languages with automatic memory management, like Java or Python, might have some vague impressions or rules:

* primitive types are stored on the stack, objects on the heap;
* small amounts of data on the stack, large amounts on the heap.

While these are generally correct, they don’t get to the heart of the matter. If you only memorize rules and formulas in your work, you may get confused when encountering special cases. However, if you understand the logic behind these formulas, you can quickly find the answer through simple reasoning, even if you forget them. So, let’s delve into the design principles of the heap and the stack and see how they actually work.

## Stack

The stack is the foundation of a program’s execution. Whenever a function is called, a contiguous block of memory is allocated at the top of the stack, known as a frame.

We know that the stack grows from the top down. At the very bottom of a program’s call stack, excluding the entry frame, is the frame corresponding to the main() function. As the main() function makes successive calls, the stack expands layer by layer. When the calls return, the stack unwinds layer by layer, releasing the memory back.

During the call, a new frame allocates enough space to store the register context. A copy of the general-purpose registers used in the function is saved on the stack. When the function call ends, the original register context can be restored from the copy, as if nothing happened. Additionally, the local variables needed by the function are also reserved when the frame is allocated.

![frame](images/rust-01-02.webp)

So, how is the size of a frame determined when a function runs?

This is thanks to the compiler. When compiling and optimizing code, a function is the smallest unit of compilation.

Within this function, the compiler needs to know which registers will be used and which local variables need to be stored on the stack. All of this has to be determined at compile time. Therefore, the compiler must know the size of each local variable in order to reserve space.

Now we understand: at compile time, any data whose size cannot be determined or can change cannot be safely placed on the stack; it is better to place it on the heap. For instance, consider a function that takes a string as a parameter:

``` rust
fn say_name(name: String) {}

say_name("Lindsey".to_string());
say_name("Rosie".to_string());
```

The size of the string’s data structure is indeterminate at compile time, and its size is only known when the code is executed at runtime. For example, in the code above, “Lindsey” and “Rosie” have different lengths, and the say_name() function only knows the specific length of the parameter at runtime.

Thus, we cannot place the string itself on the stack. We must first place it on the heap, then allocate a corresponding pointer on the stack to reference the memory on the heap.

## Problems with Stack Allocation

From the diagram, you can also see that memory allocation on the stack is very efficient. By simply adjusting the stack pointer, the necessary space can be reserved; moving the stack pointer back releases the reserved space. Reserving and releasing memory only involves modifying registers without additional calculations or system calls, which makes it very efficient.

So theoretically, whenever possible, we should allocate variables on the stack to achieve better performance.

Then why, in actual practice, do we avoid allocating large amounts of data on the stack?

This is mainly to avoid stack overflow by considering the size of the call stack. Once the current program’s call stack exceeds the maximum stack space allowed by the system, it cannot create new frames to run the next function to be executed, resulting in a stack overflow. At this point, the program will be terminated by the system, and crash information will be generated.

Over-allocating memory on the stack is one cause of stack overflow. Another more widely known cause is improperly terminated recursive functions. A recursive function continuously calls itself, and each call forms a new frame. If the recursive function does not terminate, it will eventually lead to stack overflow.

## Heap
While using the stack is very efficient, its limitations are also obvious. When we need dynamically sized memory, we can only use the heap. For example, variable-length arrays, lists, hash tables, and dictionaries are all allocated on the heap.

When allocating memory on the heap, it is generally a best practice to reserve some space.

For instance, if you create a list and add two values to it:

``` rust
let mut arr = Vec::new();
arr.push(1);
arr.push(2);
```

the actual reserved size of this list is 4, not equal to its length of 2. This is because heap memory allocation uses the malloc() function provided by libc, which internally makes system calls to the operating system to allocate memory. System calls are expensive, so we want to avoid frequent malloc() calls.

In the above code, if we allocate as much as we need each time, the list would have to allocate a large chunk of memory every time a new value is added, copy the existing data, add the new value, and then release the old memory, which is very inefficient. So, when allocating heap memory, the reserved space size of 4 is greater than the actual required size of 2.

In addition to dynamically sized memory needing to be allocated on the heap, dynamically lifecycled memory also needs to be allocated on the heap.

As mentioned earlier, stack memory is reclaimed after the function call ends, and the frame used is recycled, along with the memory corresponding to the related variables. Therefore, the lifecycle of stack memory is not controlled by the developer and is limited to the current call stack.

On the other hand, each block of heap memory needs to be explicitly released, giving heap memory a more flexible lifecycle that allows data sharing between different call stacks.

![heap](/images/rust-01-03.webp)

## Problems with Heap Allocation

However, the flexibility of heap memory also brings many challenges to memory management.

If heap memory is managed manually and the allocated memory is forgotten to be released, it will cause memory leaks. Once there is a memory leak, the longer the program runs, the more memory it consumes, eventually being terminated by the operating system for using up all available memory.

If heap memory is referenced by the call stacks of multiple threads, changes to this memory need to be handled very carefully. It must be locked for exclusive access to avoid potential issues. For example, if one thread is iterating through a list while another thread is releasing an item from the list, this could lead to accessing a wild pointer and cause heap out of bounds errors. Heap out of bounds is the number one memory safety issue.

If heap memory is freed, but the corresponding pointer on the stack pointing to this heap memory is not cleared, a use-after-free situation can occur. This can cause the program to crash at best or pose hidden security risks at worst. According to research by the Microsoft Security Response Center (MSRC), this is the second biggest memory safety issue.

![heap problems](/images/rust-01-04.webp)

## How GC and ARC Solve the Problem

To avoid the issues caused by manual heap memory management, a series of programming languages led by Java adopt a method of tracing garbage collection (Tracing GC) to automatically manage heap memory. This method periodically marks objects that are no longer referenced and then sweeps them away to manage memory automatically, reducing the burden on developers.

On the other hand, ObjC and Swift take a different approach: Automatic Reference Counting (ARC). At compile time, retain/release statements are inserted into each function to automatically maintain the reference count of heap objects. When the reference count reaches zero, the release statement frees the object.

Let’s compare these two solutions.

In terms of efficiency, GC does not require additional operations for memory allocation and release, whereas ARC adds a lot of extra code to handle reference counting. Therefore, GC is more efficient and has higher throughput.

However, the timing of GC releasing memory is uncertain, and the Stop-The-World (STW) events triggered during release can cause indeterminate latency in code execution. As a result, programming languages with GC are generally not suitable for embedded systems or real-time systems.

When using Android phones, occasional lag is often due to this issue, whereas iOS phones run smoothly. This difference is mostly because of the underlying memory management strategies. Additionally, when developing backend services, the p99 (99th percentile) of API or service response times can be adversely affected by GC STW, leading to suboptimal performance.

A side note: the GC performance discussed above differs from the general perception of performance. The common notion of performance encompasses both throughput and latency, which can be different from actual performance metrics. GC and ARC are typical examples. Although GC has higher efficiency and throughput for memory allocation and release compared to ARC, the occasional high latency gives the impression that GC is less performant than ARC.

## Summary

Today, we revisited fundamental concepts and analyzed the characteristics of the stack and the heap.

For values stored on the stack, their size must be determined at compile time. The variables stored on the stack have a lifetime confined to the scope of the current call stack and cannot be referenced across different call stacks.

The heap can store data types with unknown or dynamically changing sizes. Variables stored on the heap have a lifetime that begins when they are allocated and ends when they are released, allowing heap variables to be referenced across multiple call stacks. However, this also makes heap variable management very complex. Manual management can lead to many memory safety issues, while automatic management, whether GC or ARC, involves performance overhead and other problems.

In a nutshell: Data stored on the stack is static, has a fixed size, and a fixed lifetime; data stored on the heap is dynamic, has a variable size, and a variable lifetime.

---