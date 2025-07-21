---
title: "Go Pipeline Pattern Guide: Channels & Concurrent Processing"

description: "Master Go pipeline patterns with channels, goroutines, and fan-in/fan-out. Build concurrent data processing pipelines for scalable apps."

summary: "Learn Go's pipeline pattern for building modular, concurrent applications. Explore channel-based pipelines, HTTP middleware chains, and fan-in/fan-out patterns using goroutines for efficient data processing and stream operations."

date: 2025-07-10
series: ["Golang"]
weight: 1
tags: ["golang", "pipeline-pattern", "channels", "concurrency", "goroutines"]
author: ["Garry Chen"]
cover:
  image: images/go-09-00.webp
  hiddenInList: true
  caption: "Pipeline"

---


In this article, we focus on the Pipeline pattern in Go programming. Those familiar with Unix/Linux command-line will not find pipelines strange—they’re a way to chain commands to build more powerful functionality. Today, stream processing, functional programming, and simple API orchestration in API gateways for microservices are all influenced by the pipeline approach. Pipeline technique makes it easy to break code into multiple small modules with single responsibility and high cohesion/low coupling, then stitch them to achieve complex functionality.

# HTTP handling

We already saw a pipeline example in \[Go programming patterns: Decorators], and we'll revisit it here. In that article, we had small handler functions like `WithServerHeader()`, `WithBasicAuth()`, `WithDebugLog()`. When implementing HTTP APIs, we could compose them simply:

```go
http.HandleFunc("/v1/hello", WithServerHeader(WithAuthCookie(hello)))
http.HandleFunc("/v2/hello", WithServerHeader(WithBasicAuth(hello)))
http.HandleFunc("/v3/hello", WithServerHeader(WithBasicAuth(WithDebugLog(hello))))
```

Using a helper function:

```go
type HttpHandlerDecorator func(http.HandlerFunc) http.HandlerFunc

func Handler(h http.HandlerFunc, decors ...HttpHandlerDecorator) http.HandlerFunc {
    for i := range decors {
        d := decors[len(decors)-1-i] // iterate in reverse
        h = d(h)
    }
    return h
}
```

We can simplify nested decorators to:

```go
http.HandleFunc("/v4/hello", Handler(hello,
                WithServerHeader, WithBasicAuth, WithDebugLog))
```

# Channel-based pipelines

Writing a generic pipeline framework in Go isn't easy, but don't forget Go’s standout features: goroutines and channels. We can use them to build pipelines. 

Rob Pike introduced this pattern in “[Go Concurrency Patterns: Pipelines and cancellation](https://go.dev/blog/pipelines).”

## Channel forwarding functions

First, an `echo()` function sends integers into a channel:

```go
func echo(nums []int) <-chan int {
  out := make(chan int)
  go func() {
    for _, n := range nums {
      out <- n
    }
    close(out)
  }()
  return out
}
```

Then, typical pipeline stages:

## Square function

```go
func sq(in <-chan int) <-chan int {
  out := make(chan int)
  go func() {
    for n := range in {
      out <- n * n
    }
    close(out)
  }()
  return out
}
```

## Filter odds

```go
func odd(in <-chan int) <-chan int {
  out := make(chan int)
  go func() {
    for n := range in {
      if n%2 != 0 {
        out <- n
      }
    }
    close(out)
  }()
  return out
}
```

## Sum function

```go
func sum(in <-chan int) <-chan int {
  out := make(chan int)
  go func() {
    var sum = 0
    for n := range in {
      sum += n
    }
    out <- sum
    close(out)
  }()
  return out
}
```

Client code:

```go
var nums = []int{1,2,3,4,5,6,7,8,9,10}
for n := range sum(sq(odd(echo(nums)))) {
  fmt.Println(n)
}
```

This chains echo → odd → sq → sum, like `echo $nums | odd | sq | sum` in Unix.

To avoid nested calls, use a helper:

```go
type EchoFunc func([]int) <-chan int
type PipeFunc func(<-chan int) <-chan int

func pipeline(nums []int, echo EchoFunc, pipeFns ...PipeFunc) <-chan int {
  ch := echo(nums)
  for i := range pipeFns {
    ch = pipeFns[i](ch)
  }
  return ch
}
```

Usage:

```go
for n := range pipeline(nums, echo, odd, sq, sum) {
    fmt.Println(n)
}
```

# Fan‑in / Fan‑out

Go’s goroutines and channels also support one‑to‑many and many‑to‑one pipelines. Here’s a fan‑in example:

We want to compute the sum of primes in a large array by partitioning and then re‑aggregating.

**Main function:**

```go
func makeRange(min, max int) []int {
  a := make([]int, max-min+1)
  for i := range a {
    a[i] = min + i
  }
  return a
}

func main() {
  nums := makeRange(1, 10000)
  in := echo(nums)

  const nProcess = 5
  var chans [nProcess]<-chan int
  for i := range chans {
    chans[i] = sum(prime(in))
  }

  for n := range sum(merge(chans[:])) {
    fmt.Println(n)
  }
}
```

**Prime filter:**

```go
func is_prime(value int) bool {
  for i := 2; i <= value/2; i++ {
    if value%i == 0 {
      return false
    }
  }
  return value > 1
}

func prime(in <-chan int) <-chan int {
  out := make(chan int)
  go func() {
    for n := range in {
      if is_prime(n) {
        out <- n
      }
    }
    close(out)
  }()
  return out
}
```

* Create numbers 1…10000.
* Echo them into channel `in`.
* Spawn `nProcess` goroutines each running `sum(prime(in))`.
* Merge their results together.

**Merge function:**

```go
func merge(cs []<-chan int) <-chan int {
  var wg sync.WaitGroup
  out := make(chan int)

  wg.Add(len(cs))
  for _, c := range cs {
    go func(c <-chan int) {
      for n := range c {
        out <- n
      }
      wg.Done()
    }(c)
  }
  go func() {
    wg.Wait()
    close(out)
  }()
  return out
}
```

Diagrammatically, the program structure is:

![fan-in fan-out](images/go-09-01.png)

---

