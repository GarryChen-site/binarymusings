---
title: "Go Network Programming: How Epoll Powers Performance"

description: "Deep dive into Go's net package - how goroutines and epoll deliver high-performance async I/O with simple synchronous code patterns."

summary: "Exploring how Go combines the simplicity of synchronous programming with the efficiency of epoll-based asynchronous I/O for high-performance network programming."

date: 2025-06-13
series: ["Golang"]
weight: 1
tags: ["Golang", "Network Programming", "Epoll", "Goroutines", "Performance"]
author: ["Garry Chen"]
cover:
  image: images/go_epoll.webp
  hiddenInList: true

---


In traditional network programming, synchronous blocking was synonymous with poor performance. A single context switch could cost around 3 microseconds of CPU time. Although asynchronous, non-blocking models based on `epoll` improved performance, the callback-based programming style felt unnatural and hard to follow. The resulting code was often difficult to understand.

The rise of Golang brought coroutine-based programming to a new level. It combines the simplicity and clarity of synchronous programming with the efficiency of coroutines working alongside `epoll` under the hood—eliminating the high cost of thread switching. In short, it’s both easy to use and performs well.

Today, we're going to dive deep into the net package provided by Golang itself, to understand how it manages to deliver on its promises.

## I. Golang net usage

Given that not everyone reading this may have experience with Golang, let's start with a simple example of a Golang service using the official net package. To keep things clear, I'll only be showing the core code.
``` go
package main

import "net"


func main() {
    // Create a listener
    listener, _ := net.Listen("tcp", "127.0.0.1:9008")
    for {
        // Accept incoming connections
        conn, err := listener.Accept()
        // Handle each connection in a new goroutine
        go process(conn)
    }
}

func process(conn net.Conn) {
    defer conn.Close() // Close connection when done
    var buf [1024]byte
    len, err := conn.Read(buf[:]) // Read data from connection
    _, err = conn.Write([]byte("I am server!")) // Send response
    ...
}

```
In this sample service program, we first use `net.Listen` to listen to a local port, 9008 in this case. Then we call `Accept` to receive and handle the connection. If a connection request is received, we use `go` process to launch a coroutine for handling it. During the connection handling, I demonstrate both read and write operations (Read and Write).

At first glance, this service program looks like a traditional synchronous model. Operations like `Accept`, `Read`, and `Write` all seem to "block" the current coroutine. For instance, with the `Read` function, if the server calls it before the client data arrives, `Read` will not return, effectively parking the current coroutine. Only once data arrives does `Read` return, allowing the processing coroutine to resume.

If you were to write similar server code in other languages, such as C or Java, you'd be in a world of hurt. That's because each synchronous `Accept`, `Read`, and `Write` operation would block your current thread, leading to a lot of wasted CPU cycles due to thread context switching.

Yet, in Golang, this kind of code performs impressively well. Why is that? Well, let's keep reading to find out.

## II. The Underlying Process of Listen

In more traditional languages like C or Java, the `listen` function is typically implemented as a direct call to the kernel's listen system call. If you're thinking that's what the Listen function does in Golang's net package, think again.

Unlike its counterparts in other language, Golang's net Listen does several things:
* Creates a `socket` and sets it to non-blocking
* `bind` and listens to a local port
* Calls `listen` to start listening
* Uses `epoll_create` to create an `epoll` object
* Uses `epoll_etl` to add the listening sockets to `epoll`, waiting for a connection

One call to Golang's Listen is equivalent to multiple calls to functions like `socket`, `bind`, `listen`, `epoll_create`, `epoll_etl`, etc. in C. It's a highly abstracted operation, hiding a lot of the underlying implementation details from the programmer.

But don't just take my word for it-let's take a peek under the hood.

The Listen function's entry point can be found in the `net/dial.go` file in Golang's source code. Let's delve deeper into the details.

### 2.1 Entry Point for `Listen`

Unlike the detailed dissections often associated with source code reviews, our journey here requires an understanding of the broader picture rather than a line-by-line analysis.

```go
// src/net/dial.go
func Listen(network, address string) (Listener, error) {
	var lc ListenConfig
	return lc.Listen(context.Background(), network, address)
}
```

This is an entry point, the next will go through the `Listen` function of `ListenConfig`. The `Listen` function in `ListenConfig` will determine if it is a tcp type, and go through the `listenTCP` function under `sysListener`. Then, after two or three function jump, will go to socket function in `net/sock_posix.go`

  
```go
// net/sock_posix.go
func socket(ctx context.Context, net string, family, ...) (fd *netFD, err error) {
	s, err := sysSocket(family, sotype, proto)

	...

	if laddr != nil && raddr == nil {
		switch sotype {
		case syscall.SOCK_STREAM, syscall.SOCK_SEQPACKET:
			if err := fd.listenStream(ctx, laddr, listenerBacklog(), ctrlCtxFn); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		}
		...
	}
}
	
```
### 2.2 Creating the `socket`

`sysSocket` differs from other languages by doing socket creation, binding, and listening all in one go:

```go
// src/net/sys_cloexec.go
func sysSocket(family, sotype, proto int) (int, error) {
	// See ../syscall/exec_unix.go for description of ForkLock.
	syscall.ForkLock.RLock()
	s, err := socketFunc(family, sotype, proto)
	if err == nil {
		syscall.CloseOnExec(s)
	}
	syscall.ForkLock.RUnlock()
	if err != nil {
		return -1, os.NewSyscallError("socket", err)
	}
	if err = syscall.SetNonblock(s, true); err != nil {
		poll.CloseFunc(s)
		return -1, os.NewSyscallError("setnonblock", err)
	}
	return s, nil
}
```

Within `sysSocket`, the invoked `socketFunc` is essentially a `socket` system call. 

```go
// net/hook_unix.go
var (
	// Placeholders for socket system calls.
	socketFunc        func(int, int, int) (int, error)  = syscall.Socket
	connectFunc       func(int, syscall.Sockaddr) error = syscall.Connect
	listenFunc        func(int, int) error              = syscall.Listen
	getsockoptIntFunc func(int, int, int) (int, error)  = syscall.GetsockoptInt
)
```

Once the `socket` is created, `syscall.SetNoblcok` is employed to switch the socket into non-blocking mode.

```go
// src/syscall/exec_unix.go
func SetNonblock(fd int, nonblocking bool) (err error) {
	flag, err := fcntl(fd, F_GETFL, 0)
	if err != nil {
		return err
	}
	if nonblocking {
		flag |= O_NONBLOCK
	} else {
		flag &^= O_NONBLOCK
	}
	_, err = fcntl(fd, F_SETFL, flag)
	return err
}
```

### 2.3 Binding and Listening
Now, inside `listenStream`, we `bind` and `listen` using syscalls:

```go
// net/sock_posix.go
func (fd *netFD) listenStream(ctx context.Context, laddr sockaddr, ...) error {
	...

	if err = syscall.Bind(fd.pfd.Sysfd, lsa); err != nil {
		return os.NewSyscallError("bind", err)
	}
	if err = listenFunc(fd.pfd.Sysfd, backlog); err != nil {
		return os.NewSyscallError("listen", err)
	}
	if err = fd.init(); err != nil {
		return err
	}
	lsa, _ = syscall.Getsockname(fd.pfd.Sysfd)
	fd.setAddr(fd.addrFunc()(lsa), nil)
	return nil
}
```

`listenFunc` operates as a macro, pointing directly to the `syscall.Listen` system call.

```go
// net/hook_unix.go
var (
	socketFunc        func(int, int, int) (int, error)  = syscall.Socket
	connectFunc       func(int, syscall.Sockaddr) error = syscall.Connect
	listenFunc        func(int, int) error              = syscall.Listen
	getsockoptIntFunc func(int, int, int) (int, error)  = syscall.GetsockoptInt
)
```

### 2.4 Creating and Initializing `epoll`

At the `fd.init` line of code. After several function call expansions, the creation of the `epoll` object is executed. This step also adds the `socket` handle-which is currently in a listening state-to the `epoll` object for network event management.


```go
// src/internal/poll/fd_poll_runtime.go
func (pd *pollDesc) init(fd *FD) error {
	serverInit.Do(runtime_pollServerInit)
	ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
	if errno != 0 {
		return errnoErr(syscall.Errno(errno))
	}
	pd.runtimeCtx = ctx
	return nil
}
```

The `serverInit.Do` function is deployed to ensure that the function passed as a parameter only executes once. We won't go into too much detail about this. The parameter, `runtime_pollServerInit`, is a call to the `poll_runtime_pollServerInit` function belonging to the runtime package. You can find its source code within runtime/netpoll.go.

```go
// src/runtime/netpoll.go
//go:linkname poll_runtime_pollServerInit internal/poll.runtime_pollServerInit
func poll_runtime_pollServerInit() {
	netpollGenericInit()
}
```

This function eventually triggers the execution of `netpollGenericInit`, where the `epoll` object is created.

```go
// src/runtime/netpoll_epoll.go
func netpollinit() {
	var errno uintptr
	epfd, errno = syscall.EpollCreate1(syscall.EPOLL_CLOEXEC)
	...
}
```

`runtime_pollOpen` accepts the file descriptor of the pre-listened `socket` as its parameter. Within this function, the descriptor is added to the `epoll` object.

```go
// runtime/netpoll.go
//go:linkname poll_runtime_pollOpen internal/poll.runtime_pollOpen
func poll_runtime_pollOpen(fd uintptr) (*pollDesc, int) {
	...
	errno := netpollopen(fd, pd)
	if errno != 0 {
		pollcache.free(pd)
		return nil, int(errno)
	}
	return pd, 0
}

// runtime/netpoll_epoll.go
func netpollopen(fd uintptr, pd *pollDesc) uintptr {
	var ev syscall.EpollEvent
	ev.Events = syscall.EPOLLIN | syscall.EPOLLOUT | syscall.EPOLLRDHUP | syscall.EPOLLET
	*(**pollDesc)(unsafe.Pointer(&ev.Data)) = pd
	return syscall.EpollCtl(epfd, syscall.EPOLL_CTL_ADD, int32(fd), &ev)
}
```

## III. Accept Process

After the server completes the Listen operation, it calls Accept. This function mainly does three things:

* Calls the `accept` system call to receive a connection
* If no connection has arrived, it blocks the current goroutine
* When a new connection arrives, it adds it to `epoll` for management and then returns

With step-by-step debugging in Go, you can observe that it goes into the `Accept` method of `TCPListener`:

```go
// net/tcpsock.go
func (l *TCPListener) Accept() (Conn, error) {
	if !l.ok() {
		return nil, syscall.EINVAL
	}
	c, err := l.accept()
	if err != nil {
		return nil, &OpError{Op: "accept", Net: l.fd.net, Source: nil, Addr: l.fd.laddr, Err: err}
	}
	return c, nil
}
```
The three steps we mentions above all done in `netFD` of `accept` function.
```go
// net/tcpsock_posix.go
func (ln *TCPListener) accept() (*TCPConn, error) {
	fd, err := ln.fd.accept()
	if err != nil {
		return nil, err
	}
	return newTCPConn(fd, ln.lc.KeepAlive, nil), nil
}

// net/fd_unix.go
func (fd *netFD) accept() (netfd *netFD, err error) {
	d, rsa, errcall, err := fd.pfd.Accept()
	if err != nil {
		if errcall != "" {
			err = wrapSyscallError(errcall, err)
		}
		return nil, err
	}

	if netfd, err = newFD(d, fd.family, fd.sotype, fd.net); err != nil {
		poll.CloseFunc(d)
		return nil, err
	}
	if err = netfd.init(); err != nil {
		netfd.Close()
		return nil, err
	}
	...
}
```

Next up, let's dissect each of these steps in detail.

### 3.1 Accepting a Connection

Through step-by-step debugging, we find that `Accept` goes into the `Accept` method of the `FD` object. Here, it calls the OS-level accept system call:

```go
// internal/poll/fd_unix.go
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {

	for {
		s, rsa, errcall, err := accept(fd.Sysfd)
		if err == nil {
			return s, rsa, "", err
		}
		switch err {
		case syscall.EINTR:
			continue
		case syscall.EAGAIN:
			if fd.pd.pollable() {
				if err = fd.pd.waitRead(fd.isFile); err == nil {
					continue
				}
			}
		case syscall.ECONNABORTED:
			// This means that a socket on the listen
			// queue was closed before we Accept()ed it;
			// it's a silly error, so try again.
			continue
		}
		return -1, nil, errcall, err
	}
	...
}

```

The internal `accept` method triggers the Linux kernel’s `accept` system call (not expanded here). The goal is to get an incoming connection from the client. If received, it returns it.


### 3.2 Blocking the Current Goroutine

Now let's consider the situation where `accept` is called but no client connections have yet arrived.

In this case, the `accept` system call will return `syscall.EAGAIN`. Go handles this status by blocking the current goroutine. The key code:
  
```go
// internal/poll/fd_poll_runtime.go
func (pd *pollDesc) waitRead(isFile bool) error {
	return pd.wait('r', isFile)
}

func (pd *pollDesc) wait(mode int, isFile bool) error {
	if pd.runtimeCtx == 0 {
		return errors.New("waiting for unsupported file type")
	}
	res := runtime_pollWait(pd.runtimeCtx, mode)
	return convertErr(res, isFile)
}
```

The `runtime_pollWait` function is found in `runtime/netpoll.go`, where the goroutine is blocked using `gopark`:

```go
// runtime/netpoll.go
//go:linkname poll_runtime_pollWait internal/poll.runtime_pollWait
func poll_runtime_pollWait(pd *pollDesc, mode int) int {

	...
	for !netpollblock(pd, int32(mode), false) {
		errcode = netpollcheckerr(pd, int32(mode))
		if errcode != pollNoError {
			return errcode
		}
	// Can happen if timeout has fired and unblocked us,
	// but before we had a chance to run, timeout has been reset.
	// Pretend it has not happened and retry.
	}
	return pollNoError
}

func netpollblock(pd *pollDesc, mode int32, waitio bool) bool {
	... 
	// need to recheck error states after setting gpp to pdWait
	// this is necessary because runtime_pollUnblock/runtime_pollSetDeadline/deadlineimpl
	// do the opposite: store to closing/rd/wd, publishInfo, load of rg/wg
	if waitio || netpollcheckerr(pd, mode) == pollNoError {
		gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5)
	}

}
```

The `gopark` function is the internal Go runtime’s way to block a goroutine.

### 3.3 Adding the New Connection to epoll
  
Now let’s consider when a client connection has already arrived. In this case, `fd.pfd.Accept` returns the new connection, and Go adds this connection to epoll for efficient event management:

```go
// net/fd_unix.go
func (fd *netFD) accept() (netfd *netFD, err error) {
	d, rsa, errcall, err := fd.pfd.Accept()
	if err != nil {
		if errcall != "" {
			err = wrapSyscallError(errcall, err)
		}
		return nil, err
	}

	if netfd, err = newFD(d, fd.family, fd.sotype, fd.net); err != nil {
		poll.CloseFunc(d)
		return nil, err
	}
	if err = netfd.init(); err != nil {
		netfd.Close()
		return nil, err
	}
	...
}
```

Let’s look at `netfd.init`:
  
```go
// internal/poll/fd_poll_runtime.go
func (pd *pollDesc) init(fd *FD) error {

	ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
	...
}
```
We already discussed `runtime_pollOpen` in Section 2.4 earlier. It adds the file descriptor to the `epoll` object:
    
```go
// runtime/netpoll.go
func poll_runtime_pollOpen(fd uintptr) (*pollDesc, int) {
	...
	errno := netpollopen(fd, pd)
	if errno != 0 {
		pollcache.free(pd)
		return nil, int(errno)
	}
	return pd, 0
}

// runtime/netpoll_epoll.go
func netpollopen(fd uintptr, pd *pollDesc) uintptr {
	var ev syscall.EpollEvent
	ev.Events = syscall.EPOLLIN | syscall.EPOLLOUT | syscall.EPOLLRDHUP | syscall.EPOLLET
	*(**pollDesc)(unsafe.Pointer(&ev.Data)) = pd
	return syscall.EpollCtl(epfd, syscall.EPOLL_CTL_ADD, int32(fd), &ev)
}
```



## IV. Internal Process of Read and Write

After a connection has been successfully accepted, the remaining task is to perform read and write operations on the connection.

### 4.1 Internal Process of Read

Let’s look at the detailed code:

```go
// net/net.go
func (c *conn) Read(b []byte) (int, error) {
	...
	n, err := c.fd.Read(b)

}
```
The Read function calls the Read method of the `FD` (file descriptor). Internally, it invokes the system read call to read data. If the data hasn't arrived yet, it blocks the current goroutine:

```go
// internal/poll/fd_unix.go
func (fd *FD) Read(p []byte) (int, error) {
	...
	for {
		n, err := ignoringEINTRIO(syscall.Read, fd.Sysfd, p)
		if err != nil {
			n = 0
			if err == syscall.EAGAIN && fd.pd.pollable() {
				if err = fd.pd.waitRead(fd.isFile); err == nil {
					continue
				}
			}
		}
		err = fd.eofError(n, err)
		return n, err
	}
}
```
The `waitRead` function blocks the current goroutine, just like we explained earlier in Section 3.2, so we won’t go into details again here.

### 4.2 Internal Process of Write

The overall process of Write is similar to that of Read. It first calls the write system call to send data. If the kernel's send buffer is full, the goroutine blocks itself and waits until it becomes writable again.

The entry point for the source code is in `net/net.go`:
  
```go
// net/net.go
func (c *conn) Write(b []byte) (int, error) {
	...
	n, err := c.fd.Write(b)
	...
}

// internal/poll/fd_unix.go
func (fd *FD) Write(p []byte) (int, error) {
	...
	for {
		max := len(p)
		if fd.IsStream && max-nn > maxRW {
			max = nn + maxRW
		}
		n, err := ignoringEINTRIO(syscall.Write, fd.Sysfd, p[nn:max])
		if n > 0 {
			nn += n
		}
		if nn == len(p) {
			return nn, err
		}
		if err == syscall.EAGAIN && fd.pd.pollable() {
			if err = fd.pd.waitWrite(fd.isFile); err == nil {
				continue
			}
		}
}

// internal/poll/fd_poll_runtime.go
func (pd *pollDesc) waitWrite(isFile bool) error {
	return pd.wait('w', isFile)
}
```
After calling `pd.wait`, the process continues just like in Section 3.2: it uses `runtime_pollWait` to block the current goroutine:

  
```go
// internal/poll/fd_poll_runtime.go
func (pd *pollDesc) wait(mode int, isFile bool) error {
	...
	res := runtime_pollWait(pd.runtimeCtx, mode)
	return convertErr(res, isFile)
}
```

## V. Goroutine Wake-up in Golang

In many of the previous steps we discussed, goroutine blocking is involved. For example, when using `Accept`, if no new connection has arrived yet, or during `Read` operations when the other side hasn’t sent any data—Golang will not let the goroutine hold onto the CPU. Instead, it blocks the goroutine.

So, when the awaited event becomes ready, how does the blocked goroutine get rescheduled? That’s probably a question you're curious about.

In Go’s runtime, there is a monitoring routine called `sysmon`, which is invoked during scheduling or system monitoring. It calls `netpoll`, which in turn continuously calls `epoll_wait` to check which file descriptors managed by the `epoll` object have events that are ready to be handled. If it finds one, it wakes up the corresponding goroutine for execution.

Besides sysmon, there are a few other points in the runtime that can wake up goroutines, such as:

* startTheWorldWithSema
* findrunnable (used in the scheduler; it has both “top” and “stop” parts. The “stop” part may lead to blocking.)
* pollWork

But for simplicity, we’ll focus on sysmon as our entry point. `sysmon` is a periodic system monitoring goroutine. Let’s look at its source code:

```go
// src/runtime/proc.go
func sysmon() {
	...
	list := netpoll(0) // non-blocking - returns list of goroutines
	...
}
```

It repeatedly calls `netpoll`. Within `netpoll`, it calls `epollwait` to check for any network events:
  
```go
// runime/netpoll_epoll.go
func netpoll(delay int64) gList {
	...
	retry:
	n, errno := syscall.EpollWait(epfd, events[:], int32(len(events)), waitms)
	if errno != 0 {
	if errno != _EINTR {
		println("runtime: epollwait on fd", epfd, "failed with", errno)
		throw("runtime: netpoll failed")
	}
	// If a timed sleep was interrupted, just return to
	// recalculate how long we should sleep now.
	if waitms > 0 {
		return gList{}
	}
	goto retry
	}
	var toRun gList
	for i := int32(0); i < n; i++ {
	ev := events[i]
	if ev.Events == 0 {
		continue
	}

	if *(**uintptr)(unsafe.Pointer(&ev.Data)) == &netpollBreakRd {
		if ev.Events != syscall.EPOLLIN {
			println("runtime: netpoll: break fd ready for", ev.Events)
			throw("runtime: netpoll: break fd ready for something unexpected")
		}
		if delay != 0 {
			// netpollBreak could be picked up by a
			// nonblocking poll. Only read the byte
			// if blocking.
			var tmp [16]byte
			read(int32(netpollBreakRd), noescape(unsafe.Pointer(&tmp[0])), int32(len(tmp)))
			netpollWakeSig.Store(0)
		}
		continue
	}

	var mode int32
	if ev.Events&(syscall.EPOLLIN|syscall.EPOLLRDHUP|syscall.EPOLLHUP|syscall.EPOLLERR) != 0 {
		mode += 'r'
	}
	if ev.Events&(syscall.EPOLLOUT|syscall.EPOLLHUP|syscall.EPOLLERR) != 0 {
		mode += 'w'
	}
	if mode != 0 {
		pd := *(**pollDesc)(unsafe.Pointer(&ev.Data))
		pd.setEventErr(ev.Events == syscall.EPOLLERR)
		netpollready(&toRun, pd, mode)
	}
	}
	return toRun
}
```

When `epoll` returns, `ev.data` contains the file descriptor of the ready network `socket`. Based on the ready FD, the corresponding `pollDesc` is retrieved. In netpollready, the related goroutine is pushed into the runnable queue to wait for scheduling.
    
```go
// runime/netpoll.go
func netpollready(toRun *gList, pd *pollDesc, mode int32) {
	var rg, wg *g
	if mode == 'r' || mode == 'r'+'w' {
		rg = netpollunblock(pd, 'r', true)
	}
	if mode == 'w' || mode == 'r'+'w' {
		wg = netpollunblock(pd, 'w', true)
	}
	if rg != nil {
		toRun.push(rg)
	}
	if wg != nil {
		toRun.push(wg)
	}
}
```

## VI. In Conclusion

The advantage of synchronous programming is that it aligns with how people think in a straight, linear way. Code written in this style is easy to write and easy to understand. However, the downside is very poor performance, mainly because it leads to frequent thread context switches.

That’s why `epoll` has become the primary model for network programming on Linux. Most modern and popular network frameworks across different languages are built on top of `epoll`. The main difference lies in how each language or framework uses `epoll`. While asynchronous, non-blocking models based on `epoll` significantly improve performance, they usually rely on callback-based programming, which doesn’t align well with linear human thinking. As a result, such code tends to be harder to write and understand.

Golang introduces a new model for network programming. From the application’s point of view, it still looks like synchronous code. But under the hood, Go combines goroutines and `epoll`, avoiding the high cost of thread switching. Instead of blocking user threads, it uses lightweight goroutine switching, which has much lower overhead.


---