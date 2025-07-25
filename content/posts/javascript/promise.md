---
title: "JavaScript Async Programming: From Callbacks to Promises"

description: "Deep dive into async programming fundamentals - hardware latency, event loops, callback hell, and how JavaScript Promises solve concurrency challenges."

summary: "Comprehensive guide to JavaScript async programming - from hardware limitations and CPU latency to event loops, callback hell, and the evolution of Promises for better concurrency."

date: 2025-06-13
series: ["JavaScript"]
weight: 1
tags: ["JavaScript", "Async Programming", "Promises", "Event Loop", "Callbacks"]
author: ["Garry Chen"]
cover:
  image: images/shock wave.gif
  hiddenInList: true
  caption: "Callback Hell"

---
You may have heard that Moore’s Law is no longer holding true. Technological progress doesn’t always grow exponentially. As chip integration reaches extreme levels—where a single square millimeter can hold hundreds of millions of transistors, what we often refer to as the nanometer process—the semiconductor industry is hitting a ceiling. Shrinking things further brings us close to the physical limit, where even a single atom can barely fit inside a transistor. At that point, we inevitably face quantum effects—what people jokingly call magic or quantum voodoo, as in: “When in doubt, blame quantum mechanics.”

In short, computer hardware development has reached a bottleneck. The speed of CPUs is unlikely to improve much further. Meanwhile, with the widespread adoption of mobile internet and the rise of the Internet of Things, applications are facing increasing pressure from high volumes of requests. When hardware can no longer scale easily, we have no choice but to optimize more than ever at the software and system level.

One very noticeable trait in computing is that different types of hardware have vastly different access speeds. **This gap becomes the core focus of nearly all optimization efforts**. Jeff Dean once proposed a famous set of numbers to illustrate just how dramatic these differences in hardware access times are. There’s a webpage that gives a very intuitive sense of it.

![Latency Numbers Every Programmer Should Know](/images/latency_numbers_every_programmer_should_know.webp)

What we need to focus on here are the orders of magnitude of time it takes to **access memory**, make **network requests**, and **access disk storage**.

Thanks to many internal optimizations in CPUs (like pipelining), we can roughly assume that executing a single instruction takes about 1 nanosecond (a few clock cycles). In contrast, a memory access takes around 100 nanoseconds, which is 100 times slower than an instruction. Random access from an SSD takes about 16,000 nanoseconds, while access from a mechanical hard drive takes 2,000,000 nanoseconds, or 2 milliseconds. And for something we use constantly—the internet—a round trip across the Atlantic takes about 150,000,000 nanoseconds, or 150 milliseconds.

You can clearly see that compared to these other hardware components, the CPU is blazingly fast. If nanoseconds feel abstract, let’s convert them into seconds for a more intuitive sense. Suppose executing one CPU instruction takes 1 second:

* Memory access would take 1 minute and 40 seconds.

* Random SSD read: 4 hours and 24 minutes.

* Hard disk read: over 23 days.

* One transatlantic network round trip: 4.8 years.

Now you really get a sense of how fast CPUs are.

But speed brings its own problems. As the saying goes, “It’s lonely at the top.” From the CPU’s point of view, all other hardware is unbearably slow (relativity, anyone?). And the key issue is that our programs depend heavily on these “slow” components. For example, reading a file from disk into memory before performing calculations on it. That means if a certain instruction depends on data still loading from disk, the CPU has to wait before executing it. Say we want to execute `answer = a + 1`, but the value of a is still on disk. Just to perform a 1ns addition, the CPU might end up waiting 2,000,000ns—a wait that feels like an eternity.

To solve this and similar problems, early computer scientists designed time-sharing operating systems. Let’s skip the history lesson and fast forward to now. In modern systems, time-sharing roughly corresponds to threads. The OS treats threads as the smallest unit of scheduled execution. When a thread runs into a time-consuming operation, the OS suspends it and switches the CPU to another thread. This way, the CPU doesn't have to sit idle for millions of nanoseconds just to execute a single instruction that takes one nanosecond. With good scheduling, CPU utilization is significantly improved, allowing more tasks to be completed in the same amount of time.

About 20 years ago, using multithreading was the most common approach to concurrency. The popular Apache server used a thread-per-request model: every request would create a new thread to handle it, and the OS would handle scheduling and cleanup. This worked just fine back then—think about it: the internet was much less developed, far fewer people were online, and websites were much simpler. So, servers didn’t have to handle high concurrency.

But over time—especially with the rise of mobile internet and IoT—things changed. People are now constantly connected, and major websites are facing explosive growth in user traffic. The limitations of multithreading soon became obvious, especially in the web domain.

Why is the web such a typical example? Because most web services are I/O-bound. The flow usually looks like this:

``` text
Receive a request → Query the database → Make a few RPC calls → Combine the data → Return the response
```
In this process, the CPU does very little—most of the time is spent waiting for responses from the DB or downstream services. If you use a multithreaded model to run a web server, you’ll quickly notice that although there are many threads, most of them are just waiting for network or disk I/O. CPU usage remains low, and most of the CPU is wasted on thread management and switching.

What’s worse, as the number of concurrent requests increases, the overhead of threads becomes significant. Each thread requires its own stack space—typically 8MB. So just 1,000 threads would use 8GB of memory. On top of that, thread switching is expensive. Each switch requires the OS to save and restore all the thread’s registers. With many threads, this becomes a major cost.

So, even though multithreading is **intuitive and natively** supported by the OS, as concurrent workloads grow, we’re forced to find better solutions.

Now we finally get to the point: asynchronous programming.

One thing to clarify up front:

> The goal of asynchronous programming is not to make individual tasks faster, but to let computers complete more tasks in the same amount of time.

Asynchronous systems are actually very complex and span three layers:

* Hardware
* Operating System
* Async programming paradigm

Clearly, hardware is the foundation of all asynchronous capability. But since this is about asynchronous programming, we’ll focus more on the software side—especially OS-level support and programming paradigms.

When people talk about async, they often confuse OS-level mechanisms with programming-level concepts. In discussions and blogs, you’ll often see terms like:

> I/O multiplexing, epoll, libev, callback hell, async/await…

In the next sections, I’ll walk you through these concepts systematically, so you can truly understand what asynchronous programming is all about—from the ground up.


## A Preliminary Understanding of the Benefits of Asynchronous Programming

Let’s start with the simplest piece of code:

``` javascript
let number = read("number.txt");
print(number + 1);

```
This pseudocode reads a number from the file number.txt, adds 1 to it, and prints the result. Very straightforward.

Because the `number + 1` operation needs a definite value of number, the program must wait for `read` to finish before executing `print` (since number won’t have a value otherwise). And since read involves accessing the disk, which physically takes time, the total time needed to complete read + print won’t change whether the code is synchronous or asynchronous. So:

> An individual asynchronous task won't magically outpace its synchronous counterpart.

The main purpose of using async is to make full use of CPU resources.

Let’s go back to our example. Suppose the operating system provides a `read_async` function. When called, it immediately returns an object, which we can later use to check whether the read operation has completed. Here’s how the code might change:


``` javascript
let operation = read_async("number.txt");
let number = 0;
while true {
    if operation.is_finished() {
        number = operation.get_result();
        break;
    }
}
print(number + 1);

```

It actually seems worse now!

Since we still need a definite value for `number` before calling `print`, we’re stuck in a loop, constantly checking whether the operation is finished. So even though we quickly got the operation object, we have no choice but to keep polling it. Yes, the CPU usage went up—but all that CPU time is wasted just checking the status. Compared to the earlier synchronous code, this version takes the same amount of time, but uses far more CPU resources and achieves nothing extra. This kind of async code does nothing but waste energy and complicate the logic.

So, what’s the real value of async?

Let’s slightly modify the example:

```javascript
let a = read("facebook.com");
let b = read("amazon.com");

print(a + b);
```

Assume each network request takes 50ms, and we ignore the time taken by print. Then the total time for this code to run is 100ms.

Now let’s try the asynchronous version:

```javascript
let op_a = read_async("facebook.com");
let op_b = read_async("amazon.com");

let a = "";
let b = "";

while true {
    if op_a.is_finished() {
        a = op_a.get_content();
        break;
    }
}

while true {
    if op_b.is_finished() {
        b = op_b.get_content();
        break;
    }
}

print(a + b);
```

Again, even though these are asynchronous reads and return immediately, you still have to wait at least 50ms before any result comes back.

But here’s the key difference:

The two async requests—read_async("facebook.com") and read_async("amazon.com")—are sent out one after another, almost simultaneously. So, ideally, they can complete at the same time. That means, by the time the CPU finishes looping to check `op_a`, it might already be complete. Then the program checks `op_b`, and maybe it has also finished. So instead of waiting 50ms twice, the total wait time is around 50ms total.


> Async doesn't make logically sequential tasks faster; it only speeds up tasks that are logically parallel.

While the async version is faster in this case, it also comes with a cost. The synchronous version took 100ms, but during that time, the CPU was idle—basically "sleeping." In the async version, the CPU kept running, polling the status of the operations. So yes, it's faster—but it consumes more energy!


## Combining the Advantages of Synchronous Programming

In the synchronous code example above, when a synchronous call is made, the operating system suspends the current thread, waits for the operation to complete, and then wakes the thread up to continue execution. During this wait, the CPU can run other threads’ tasks, so it’s not sitting idle. If there’s nothing else to run, it can even go into a low-power, semi-sleep state.

This highlights a key point:

> The operating system knows exactly when a disk read or a network call completes.

That’s the only way it can wake up the correct thread at the right time. (The OS uses a mechanism called interrupts here, which is beyond the scope of this article.)

Since the OS has this ability, imagine if it provided a function like this:

``` javascript
fn wait_until_get_ready(Operation) -> Response {
    // A blocking task that suspends the thread until the operation is ready
}
```
With this function, we can rewrite our async code like this:

``` javascript
let op_a = read_async("facebook.com");
let op_b = read_async("amazon.com");
let a = wait_until_get_ready(op_a);
let b = wait_until_get_ready(op_b);
print(a + b);
```

When `wait_until_get_ready(op_a)` is called and `op_a` isn’t ready yet, the OS suspends the thread and waits until about 50ms later when `op_a` is ready. This process, like synchronous blocking code, doesn’t consume any CPU resources. Then it continues to `wait_until_get_ready(op_b)`, which might already be ready.

In this way, we use asynchronous code to finish the task in just 50ms without any extra CPU cost — a perfect result!

To make asynchronous code work like this, we rely on two key factors:

* `read_async` must return immediately after handing off the task to the OS, instead of blocking until the task finishes. This allows concurrent execution of tasks that don’t depend on each other.

* `wait_until_get_ready` depends on the OS’s notification mechanism to know when the operation is ready. This avoids the need for polling, which greatly saves CPU resources.

Both `read_async` and `wait_until_get_ready` are pseudocode functions, but implementing them relies heavily on low-level support from the operating system.

This involves many well-known technologies — for example:

* select, poll, and epoll in Linux

* kqueue in macOS

* IOCP (I/O Completion Ports) in Windows

Each OS implements these mechanisms differently and with varying levels of complexity.

By the way, Linux’s `epoll` is not as well-designed or performant as Windows’ IOCP. But with the release of the Linux 5.1 kernel, a new mechanism called io_uring was introduced — essentially a nod to IOCP — and it may bring new opportunities and improvements to async programming on Linux.


## Evolving from Scratch to JavaScript

Next, still based on the two “async primitives” mentioned earlier, I’d like to talk about what I personally think is the most important and most frequently encountered part of asynchronous programming for developers — the asynchronous programming paradigm.

The examples above were very simple, but in real-world scenarios, it’s quite rare to have several independent tasks running in parallel that can just be awaited at the end. Most tasks have logical dependencies, and when that happens, our code becomes much harder to write and read.

Let’s continue the earlier example (with a small change):

```javascript
let op_a = read_async("facebook.com");
let op_b = read_async("amazon.com");
let a = wait_until_get_ready(op_a);
write("facebook.html", a);
let b = wait_until_get_ready(op_b);
write("amazon.html", b);
```

Previously we assumed each async request took 50ms, but in real cases, we can’t make such assumptions — especially with network operations. Different requests often take different response times, which is easy to understand.

Suppose facebook.com takes 50ms and amazon.com takes 10ms. What’s the problem with the above code?

If we call `let a = wait_until_get_ready(op_a)` first, the thread is blocked until `op_a` is ready — so the code won’t continue until 50ms later. But `op_b` was already ready by the 10ms mark. Our program didn’t handle it in time.

The root cause is: we don’t know when each async operation will finish, so we can only `wait_until_get_ready` in a fixed order, which leads to poor performance. So what’s the solution?

The problem lies in the fact that `wait_until_get_ready` can only `wait` for one operation. That’s inconvenient.

We might as well submit a feature request to Linux Torvalds and ask the operating system to support two functions:

```javascript
fn add_to_wait_list(operations: Vec<Operation>)
fn wait_until_some_of_them_get_ready() ->Vec<Operation>
 ```

With `add_to_wait_list`, we register operations with a global listener. Then we call `wait_until_some_of_them_get_ready`, which blocks until one or more registered operations are ready, and returns a list of those ready operations. If nothing is ready, it returns an empty list immediately.

Imagine Linux Torvalds has implemented these functions and given us API access. Our async code could now look like this:

 ```javascript
let op_a = read_async("facebook.com");
let op_b = read_async("amazon.com");
add_to_wait_list([op_a, op_b]);
while true {
    let list = wait_until_some_of_them_get_ready();
    if list.is_empty() {
        break;
    }
    for op in list {
        if op == op_a {
            write("facebook.html", op.get_content());
        } else if op == op_b {
            write("amazon.html", op.get_content());
        }
    }
}
```

Through this way, our program can react to the event immediately, without waiting foolishly, once receive a request, output a file immediately.

This approach ensures our program isn't just sitting idly by, waiting for tasks in the order we wrote them. Instead, it's agile, responding to whichever task completes first and handling it immediately. It's a more adaptive approach to asynchronous programming, ensuing we're always making the most of out time and resources.

This way, our program can react to completed async operations immediately, avoiding unnecessary waiting. As soon as a response is ready, we can process it.

But if you think more deeply, you’ll see two remaining issues:

* Because the operations take different times, `wait_until_some_of_them_get_ready` might return one or many operations. So we need to keep looping until all are done.

* The returned operations vary, so we need to figure out what each ready operation is and then execute the corresponding logic. This leads to complicated logic matching.

Imagine if we had 10,000 async operations, and each required different handling — that’s potentially 10,000 switch cases!

So, what can we do?

There’s a simple solution: since each `operation` is tied to specific follow-up logic, we can bind a callback function directly to the `operation`. For example:

```javascript
function read_async_v1(targetURL: String, callback: Function) {
    let operation = read_async("facebook.com");
    operation.callback = callback;
    return operation;
}
```

Now when we create an async task, we also attach its callback. Then our loop becomes much cleaner — no more O(n) matching.

This is basically dynamic dispatch, like in C++.

```javascript
let op_a = read_async_v1("facebook.com", function(content) {
    write("facebook.html", content);
});

let op_b = read_async_v1("amazon.com", function(content) {
    write("amazon.html", content);
});

add_to_wait_list([op_a, op_b]);

while true {
    let list = wait_until_some_of_them_get_ready();
    if list.is_empty() {
        break;
    }
    for op in list {
        op.callback(op.get_content());
    }
}
```

The key step here is that the object returned by `read_async` must support attaching a callback function.

Once we reach this stage, we can improve it further by letting `read_async_v2` handle registration too:

```javascript
function read_async_v2(target, callback) {
    let operation = read_async(target);
    operation.callback = callback;
    add_to_wait_list([operation]);
}
```

Then our code can be more simple:

```javascript
read_async_v2("facebook.com", function(content) {
    write("facebook.html", content);
});

read_async_v2("amazon.com", function(content) {
    write("amazon.html", content);
});

while true {
    let list = wait_until_some_of_them_get_ready();
    if list.is_empty() {
        break;
    }
    for op in list {
        op.callback(op.get_content());
    }
}
```
Since we now bind logic directly to each `operation`, every async program ends with a loop that waits for events and dispatches callbacks. So if we had a chance to design a programming language, we could embed this loop into the language runtime, so users don’t have to write it manually every time.

Then, our code would just be:

```javascript
read_async("facebook.com", function(content) {
    write("facebook.html", content);
});

read_async("amazon.com", function(content) {
    write("amazon.html", content);
});

// The compiler or async framework would insert the event loop here

```
See? This is exactly how JavaScript works!
The V8 engine runs the event loop for us — what we usually call the EventLoop in JS.

We didn’t use any magic. Just by relying on a few basic OS primitives, we’ve naturally transitioned into JavaScript’s async programming model.

Quick Recap:

* To handle more requests at once, we started with multithreading.But threads are heavy on memory and CPU, so we moved to using the OS’s async interfaces.

* Since we don’t know when an async operation will finish, we can’t just poll all the time.Instead, we use OS-provided event queues and `wait_until_some_of_them_get_ready`, i.e., the EventLoop.

* After events are ready, we didn’t know what to do with them, so we bound callbacks to each event. This made our code cleaner and more efficient.

* Every async app needs an event loop to wait and dispatch callbacks, so we might as well put the event loop inside the language runtime.

The function `wait_until_some_of_them_get_ready` maps to: epoll in Linux, kqueue in macOS, IOCP in Windows.Not just JavaScript — many C/C++ async frameworks follow similar ideas. For example: Redis uses libevent, Node.js uses libuv

These frameworks all offer async I/O interfaces, support event registration, and rely on callbacks.
However, languages like C, which don’t support closures, make writing and debugging async code much harder than in JS.

So, from this evolution, you can see that JavaScript’s async callbacks are already a very advanced way to write async code.

But what comes next... is truly where the devil shows up!

## The Callback Hell

When writing asynchronous code using callbacks, you’ll soon run into the infamous callback hell. For example:

```javascript
login(user => {
    getStatus(status => {
        getOrder(order => {
            getPayment(payment => {
                getRecommendAdvertisements(ads => {
                    setTimeout(() => {
                        alter(ads)
                    }, 1000)
                })
            })
        })
    })
})
```
This is a classic example of callback hell. Every asynchronous operation needs a callback to handle what happens next after it finishes. When these operations depend on one another, the callbacks get nested deeper and deeper. As your business logic grows more complex, this nested structure makes your code hard to read, write, and debug.

JavaScript developers have put a lot of thought into how to solve this problem. One of the most popular solutions is Promise. Now, a promise **isn’t some kind of magic** that can turn synchronous code into asynchronous code. It’s just a programming pattern—a kind of wrapper. It doesn’t rely on extra help from the OS, which is why we call this a programming paradigm. A promise is exactly that: a paradigm.

Let’s look again at the example above and ask: what’s really causing this deep nesting? At the core, it’s because we use callbacks to express “what to do next once this async operation finishes.” When multiple async operations need to happen in order, the nesting grows and grows—hence callback hell.

But if these operations are sequential, can we describe them in a more linear, straightforward way? For example:

```javascript
login(username, password)
    .then(user => getStatus(user.id))
    .then(status => getOrder(status.id))
    .then(order => getPayment(order.id))
    .then(payment => getRecommendAdvertisements(payment.id))
    .then(ads => {/* ... */});
```

This version is much easier to follow. Do A, then B, then C… The order is obvious, and it matches how we naturally think: one step at a time. But how does this work behind the scenes?

Think about we implement async callback, async function will return an `operation` object, this object keeps the function pointer of async function, so when `EventLoop` find the `operation` is ready, then it directly jump to the corresponding callback function to execute. But at above chain code of calling `.then`, when we call `login(username, pwd).then(...)`, take a look after the `login` function is done, then call `then`. This is the equivalent of a callback that I have already committed the asynchronous function to execute before I bind it. Is that possible?

Let's review our previous `read_async_v2`:

```javascript
function read_async_v2(target, callback) {
  let operation = read_async(target);
  operation.callback = callback;
  add_to_wait_list([operation]);
}
```
We set the `operation` 's callback inside of the function directly, and add `operation` to the listen queue. But we don't have to set the callback function at this moment, just before the `EventLoop` execute it. According to this idea, we can save `operation` to an object, and add callback function to the object through `operation`. such as:

```javascript
function read_async_v3(target) {
    let operation = read_async(target);
    add_to_wait_list([operation]);
    return {
        then: function(callback) {
            operation.callback = callback;
        }
    }
}

// we can do this
read_async_v3("facebook.com").then(logic)
```

But this only supports a single callback. To support chained `.then()` calls, we can do something like this:

```javascript
function read_async_v4(target) {
  let operation = read_async(target);
  add_to_wait_list([operation]);
  let chainableObject = {
    callbacks: [],
    then: function(callback) {
      this.callbacks.push(callback);
      return this;
    },
    run: function(data) {
      let nextData = data;
      for cb in this.callbacks {
        nextData = cb(nextData);
      }
    }
  };
  operation.callback = chainableObject.run;
  return chainableObject;
}

// So we can do this
read_async_v4("facebook.com").then(logic1).then(logic2).then(/*...*/)

```
Above code, the main idea is, we return an object, which has a `callback` queue, every time we call `then` function of the object to add a new callback. Due to `then` function return the object itself, so we can chain call `then` function. Then, we set the `operation` 's callback to the `run` function of the object, when the `operation` is ready, which is the `EventLoop` called is the wrapped `run` function of the object, then it will execute the callback queue one by one which we added through `then` function.

It seems all done ? No, there is a serious problem, the chain call actually binding to a async function, so when the async function is ready, `run` function will finish all the callback functions which are binding to `then`. If the callbacks include another async call, like we first request facebook.com then return the response of facebook.com, then request amazon.com, then return the response of amazon.com:

```javascript
read_async_v4("facebook.com")
    .then(data => console.log("facebook.com: ${data}"))
    .then((_) => read_async_v4("amazon.com"))
    .then(data => console.log("amazon.com: ${data}"))
```
But the code above has a problem, the three `then` functions are the callback of 'facebook.com', so when the request of 'facebook.com' is done, `EventLoop` execute the `run` function of `operation`, then `run` function will call the three callbacks one by one. When the two callback called, it just send an async request to 'amazon.com', then return a `chainable` object to 'amazon.com'. So the param of third `then` is not the content response of 'amazon.com' we expected, instead of a `chainable` object. So the final result may be like this:

```javascript
"facebook.com: <html> ... </html>"
"amazon.com: [object Object]"
```
After a while, thr request of 'amazon.com' is done, but there is no callback setted, so the result of 'amazon.com' is abandoned. This is not what we expected. The correct result should be like this:

```javascript
read_async_v4("facebook.com")
    .then(data => console.log("facebook.com: ${data}"))
    .then((_) => {
        read_async_v4("amazon.com")
            .then(data => console.log("amazon.com: {$data}"))
            .then((_) => {
                read_async_v4("google.com")
                    .then(data => console.log("google.com: ${data}"));
            });
    });
```
This is what we really want, first output the response of 'facebook.com', then output the response of 'amazon.com'. If there is async request in `then`, then must bind it to `then` internally...... but does it return back to the callback hell?

To solve this problem is quite easy, just change the `run` function, when a callback returns a `chainableObject`, then we bind the remaining callbacks to that `object`, then return

```javascript
function read_async_v5(target) {
  let operation = read_async(target);
  add_to_wait_list([operation]);
  let chainableObject = {
    callbacks: [],
    then: function(callback) {
      this.callbacks.push(callback);
      return this;
    },
    run: function(data) {
      let nextData = data;
       let self = this;
      while self.callbacks.length > 0 {
          // pop the first callback
          let cb = self.callbacks.pop_front();
          nextData = cb(nextData);
          // if the callback return a chainable object, then bind the remaining callbacks to the object
          // then return
          if isChainableType(nextData) {
              nextData.callbacks = self.callbacks;
              return;
          }
      }
    }
  };
  operation.callback = chainableObject.run;
  return chainableObject;
}
```
After this change, we can implement the really async chain call:

```javascript
read_async_v5("facebook.com")
    .then(data => console.log("facebook.com: ${data}"))
    .then((_) => read_async_v5("amazon.com"))
    .then(data => console.log("amazon.com: ${data}"))
```

Though all the `then` at first, bind callback to the `operation` which request `facebook.com` asynchronously, but is temporary. When finish the second callback, it returns a `chainableObject`, then bind the remain callbacks to the new object, don't execute. After the `amazon.com` request is ready, `EventLoop` execute the `run` function of the `operation`, then continue to execute the remaining callbacks.

If we provide a library, which include all the async function, the common is will return a `ChainableObject`, after this, we can use `then` to combine to finish our development. This is so call async EcoSystem.

Suppose we provide an async library, it includes: `http_get`, `http_post`, `read_fs`, `write_fs` functions which return a `ChainableObject`. The we can combine them to implement complex logic, the previous callback hell can rewrite like this:

```javascript
http_post("/login", body)
  .then(user => http_get("/order?user_id=${user.id}"))
  .then(order => http_post("/payment", {oid: order.id}))
  .then(/*...*/)

```
Does it looks like the same as the chain calls base the `Promise` in js ?

Actually the above `chainableObject` is the `Promise` in js. The different is , due to the js base on callback programming paradigm at first, every internal async function in the standard lib can only accept callback, in order to backward compatible, can't change those internal function to return a `Promise`. So the js's solution is to provide a `promise` construct function, which 'wrap' all async function to `promise`. If base the `chainableObject` we implemented, we can do this:

```javascript
function ChainableObject() {
  return chainableObject = {
    callbacks: [],
    then: function(callback) { /* like before */},
    run: function(data) {
      let nextData = data;
      if self.resolveData != null {
        nextData = self.resolveData;
      }
      while self.callbacks.length > 0 {
        // like before
      }
    },
    resolveData: null,
  };
}

function Convert2Chainable(targetFunction) {
  let obj = new ChainableObject();
  function resolve(data) {
    obj.resolveData = data;
  }
  targetFunction(resolve);
  return obj;
}
```
Then, we can use the old callback in standard library to write the code like this:

```javascript
Convert2Chainable(resolve => {
  let sleep = 100;
  setTimeout(() => {
    resolve(sleep);
  }, sleep);
}).then(data => console.log("sleep: ${data}ms"))
  .then((_) => Convert2Chainable(resolve => {
    let sleep = 200;
    setTimeout(() => {
      resolve(sleep);
    }, sleep);
})
.then(data => console.log("sleep: ${data}ms"))
.then((_) => Convert2Chainable(resolve => {
  $.ajax("facebook.com", function(res) {
    resolve(res);
  });
})
.then(data => console.log("facebook.com: ${data}"));

```
The key idea here is that `Convert2Chainable` lets us wrap any callback-style function into a chainable object. Our `ChainableObject` is a simplified version of a JS `Promise`. In real-world usage, a full Promise implementation handles errors and other edge cases too—but the core concept is the same.

In this article, I spent some time walking you through the core principles of asynchronous programming step by step using examples. You can clearly see that all asynchronous tasks are initiated by a single thread, and all subsequent logic is also handled by this same thread. When a large number of concurrent requests come in, we can handle them all with just one thread—that’s the true value of asynchronous programming. This is also a key reason why **Node.js** can be used for backend development and still achieve very good performance! Well-known tools like **Redis** and **Nginx** are built on this kind of asynchronous programming model and benefit greatly from the advantages it brings.


---