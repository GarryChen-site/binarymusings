---
title: "Go Functional Options Pattern Guide: Clean API Design"

description: "Learn Go's Functional Options pattern for flexible API design. Compare with Builder pattern and config structs. Build clean, maintainable APIs."

summary: "Learn the popular Functional Options pattern in Go for building clean, configurable APIs that are easy to use, maintain, and extend without complex constructors or configuration structs."

date: 2025-06-30
series: ["Golang"]
weight: 1
tags: ["Golang", "Functional Options", "API Design", "Design Patterns", "Configuration"]
author: ["Garry Chen"]
cover:
  image: images/go-03-00.jpg
  hiddenInList: true
  caption: "Function"

---

In this article, we’ll discuss the **Functional Options** programming pattern. This is a functional programming use case and a great coding technique—arguably the most popular such pattern in Go today. But before diving into the pattern, let’s first see what problem it solves.



## Configuration Option Problem

In programming, we often need to configure an object (or business entity). For example, here’s a sample business entity:

```go
type Server struct {
    Addr     string
    Port     int
    Protocol string
    Timeout  time.Duration
    MaxConns int
    TLS      *tls.Config
}
```

In this `Server`, we see:

* `Addr` (IP address) and `Port` are **required** (no defaults).
* `Protocol`, `Timeout`, and `MaxConns` are optional but have defaults (e.g., `"tcp"`, `30*time.Second`, `1024`).
* `TLS` is optional and can be `nil`.

To create a `Server` with varying configurations, you end up writing many constructors:

```go
func NewDefaultServer(addr string, port int) (*Server, error) {
    return &Server{addr, port, "tcp", 30*time.Second, 100, nil}, nil
}

func NewTLSServer(addr string, port int, tls *tls.Config) (*Server, error) {
    return &Server{addr, port, "tcp", 30*time.Second, 100, tls}, nil
}

func NewServerWithTimeout(addr string, port int, timeout time.Duration) (*Server, error) {
    return &Server{addr, port, "tcp", timeout, 100, nil}, nil
}

func NewTLSServerWithMaxConnAndTimeout(addr string, port int, maxconns int, timeout time.Duration, tls *tls.Config) (*Server, error) {
    return &Server{addr, port, "tcp", 30*time.Second, maxconns, tls}, nil
}
```

Since Go doesn’t support function overloading, each variant needs a different name.


## Configuration Object Approach

A common fix is using a config struct:

```go
type Config struct {
    Protocol string
    Timeout  time.Duration
    MaxConns int
    TLS      *tls.Config
}

type Server struct {
    Addr string
    Port int
    Conf *Config
}
```

Now you use a single constructor:

```go
func NewServer(addr string, port int, conf *Config) (*Server, error) { ... }

// usage:
srv1, _ := NewServer("localhost", 9000, nil)

conf := Config{Protocol:"tcp", Timeout:60*time.Second}
srv2, _ := NewServer("localhost", 9000, &conf)
```

This works, but you still have `conf` being optional—so checks for `nil` or `Config{}` make the code feel messy.



## Builder Pattern

Java folks might default to a Builder pattern:

```java
User user = new User.Builder()
  .name("Jeff")
  .email("jeff@example.com")
  .nickname("J")
  .build();
```

In Go:

```go
type ServerBuilder struct {
    Server
}

func (sb *ServerBuilder) Create(addr string, port int) *ServerBuilder {
    sb.Server.Addr = addr
    sb.Server.Port = port
    // set defaults...
    return sb
}

func (sb *ServerBuilder) WithProtocol(p string) *ServerBuilder { ... }
// other WithXxx methods...
func (sb *ServerBuilder) Build() Server {
    return sb.Server
}
```

Usage:

```go
sb := ServerBuilder{}
server := sb.Create("127.0.0.1", 8080).
  WithProtocol("udp").
  WithMaxConn(1024).
  WithTimeout(30*time.Second).
  Build()
```

This is clear, but the builder struct wraps `Server`. You *could* use methods directly on `Server`, but then where to store interim state or errors? A builder wrapper keeps `Server` clean.


## Functional Options

Enter **Functional Options**, a functional style in Go.

First, define the option function type:

```go
type Option func(*Server)
```

Next, build several option functions:

```go
func Protocol(p string) Option {
    return func(s *Server) {
        s.Protocol = p
    }
}

func Timeout(t time.Duration) Option {
    return func(s *Server) {
        s.Timeout = t
    }
}

func MaxConns(m int) Option {
    return func(s *Server) {
        s.MaxConns = m
    }
}

func TLS(cfg *tls.Config) Option {
    return func(s *Server) {
        s.TLS = cfg
    }
}
```

Each accepts a parameter and returns a function that sets that field on `*Server`. For example, `MaxConns(30)` returns `func(s *Server){ s.MaxConns = 30 }`. This is a higher-order function. In mathematics, it's like this kind of definition: the formula for calculating the area of a rectangle is `rect(width, height) = width * height`; this function requires two parameters. If we wrap it, we can turn it into the formula for calculating the area of a square: `square(width) = rect(width, width)`. In other words, `square(width)` returns another function, which is `rect(w, h)`, except that both of its parameters are the same. That is: `f(x) = g(x, x)`.


Finally, define `NewServer` to take variadic options:

```go
func NewServer(addr string, port int, options ...func(*Server)) (*Server, error) {
    srv := Server{
        Addr:     addr,
        Port:     port,
        Protocol: "tcp",
        Timeout:  30 * time.Second,
        MaxConns: 1000,
        TLS:      nil,
    }
    for _, option := range options {
        option(&srv)
    }
    return &srv, nil
}
```

Use it like:

```go
s1, _ := NewServer("localhost", 1024)
s2, _ := NewServer("localhost", 2048, Protocol("udp"))
s3, _ := NewServer("0.0.0.0", 8080, Timeout(300*time.Second), MaxConns(1000))
```

This is clean, elegant—no need for config structs, nil checks, or builders. It’s intuitive, configurable, extendable, self-documenting, newcomer-friendly, and unambiguous.


**Benefits**:

* Intuitive code
* High configurability
* Easy to maintain/extend
* Self-documenting
* Clear for new developers
* No confusion over nil vs. empty values
