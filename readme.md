# Socket sharding in Linux with Go

I bet there have been many times that you were working on the terminal with multiple tabs and you launched an HTTP server, and then you forgot that the server was already being executed, and then you tried to relaunch it from another tab getting the known error:

```bash
go run main.go 
listen tcp :8080: bind: address already in use
```
This is because we cannot open a socket with the same source address and port by default in Linux and the vast majority of operating systems.

### Socket options

When we create a new TCP socket on Linux, we can set options that affect the behaviour of the socket. For example, one of these options is `SO_REUSEPORT`, which allows multiple sockets to bind to the same IP address and port. With this feature, the Linux kernel distributes incoming requests across all the sockets that share the same address and port combination, getting a load balancing inside the Kernel.

> SO_REUSEPORT
> 
> For TCP sockets, this option allows accept(2) load distribution in a multi-threaded server to be improved by using a distinct listener socket for each thread.  This provides improved load distribution as compared to traditional techniques such using a single accept(2)ing thread that distributes connections, or having multiple threads that compete to accept(2) from the same socket.
> 
> For UDP sockets, the use of this option can provide better distribution of incoming datagrams to multiple processes (or threads) as compared to the traditional technique of having multiple processes compete to receive datagrams on the same socket.
> 
> https://man7.org/linux/man-pages/man7/socket.7.html

![](https://user-images.githubusercontent.com/8400576/127754771-6a789f00-7022-4602-a628-a3d124b43421.png)

As we can notice, we not only get the super power to create more than one socket with the same IP: Port combination, but we also obtain a kind of load balancer in the kernel mode.

### Go sockets

When we invoke the `net.Listen()` function in Go, this function use the `ListenConfig` struct to create the Listener.

```go
func Listen(network, address string) (Listener, error) {
	var lc ListenConfig
	return lc.Listen(context.Background(), network, address)
}
```

If we inspect `ListenConfig` we can find the method `Control` and the documentation says: <mark>If Control is not nil, it is called after creating the network connection but before binding it to the operating system.</mark>

```go
// ListenConfig contains options for listening to an address.
type ListenConfig struct {
	// If Control is not nil, it is called after creating the network
	// connection but before binding it to the operating system.
	// ...
	Control func(network, address string, c syscall.RawConn) error
	...
}
```

The function `Control` receives the `syscall.RawConn` which is a raw network connection that has a method also called control (`Control(f func(fd uintptr))`) where it will invoke the function `f` on the underlying connection's file descriptor.

Having the file descriptor, now we can use the `golang.org/x/sys/unix` package to set the socket options.

```go
// fd -> the underlying connection's file descriptor.
// unix.SOL_SOCKET  -> to set options at the socket level, we have to specify the level argument as SOL_SOCKET.
unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)
```

So, we could create a instance of `ListenConfig` with a control function to set the `SO_REUSEPORT` socket options to our sockets.

```go
var lc = net.ListenConfig{
	Control: func(network, address string, c syscall.RawConn) error {
		var opErr error
		if err := c.Control(func(fd uintptr) {
			opErr = unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)
		}); err != nil {
			return err
		}
		return opErr
	},
}
```

### Security

One question we might have at this point is, what about security? I mean, if we can open a socket with the same IP: Port of a specific app, for example, Nginx, we could hijack part of the requests that the kernel will send to us through the socket. Right?

Well, to prevent this "port hijacking," Linux has special protections or mechanisms to prevent these problems, such as:

* Both sockets must have been created with the SO_REUSEPORT socket option. If there is a socket running without SO_REUSEPORT and we try to create another socket even with the SO_REUSEPORT socket option, it will fail with the error `already in use`. 
* All sockets that want to listen to the same IP and port combination must have the same effective userID. For example, if you want to hijack the Nginx port and it is running under the ownership of the user Pepito, a new process can listen to the same port only if it is also owned by the user Pepito. So one user cannot "steal" ports of other users.

The following are super simple use cases for `SO_REUSEPORT`, of course omitting all the complexity required to achieve them:

* We could run multiple instances of our app to take advantage of our resources without the necessity of running a proxy in front of them (to have an ultra simple LB). Having multiple threads/processes/instances will have better performance than having a single one.
* Can give us the possibility of zero downtime updates. Since we can launch a new instance to receive requests and, after that, kill the old one with a graceful shutdown.

### Simple demo

In this [repository](https://github.com/douglasmakey/socket-sharding/blob/master/cmd/http-example/main.go), you will find the complete code example in Go to test this, but as it is a fairly simple code and something short, you will also have it below:

```go
package main

import (
	"context"
	"fmt"
	"net"
	"net/http"
	"os"
	"syscall"

	"golang.org/x/sys/unix"
)

var lc = net.ListenConfig{
	Control: func(network, address string, c syscall.RawConn) error {
		var opErr error
		if err := c.Control(func(fd uintptr) {
			opErr = unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)
		}); err != nil {
			return err
		}
		return opErr
	},
}

func main() {
	pid := os.Getpid()
	l, err := lc.Listen(context.Background(), "tcp", "127.0.0.1:8080")
	if err != nil {
		panic(err)
	}
	server := &http.Server{}
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		fmt.Fprintf(w, "Hello from PID %d \n", pid)
	})

	fmt.Printf("HTTP Server with PID: %d is running \n", pid)
	panic(server.Serve(l))
}
```

Using the above code, we can open a terminal with 3 tabs. In the first one, we will run the program:

```bash
$ go run main.go
HTTP Server with PID: 8183 is running
```

In the second one, we will have another instance of our program.

```bash
$ go run main.go
HTTP Server with PID: 8298 is running
```

Then, in the last one, we will run a simple loop to hit our servers, and you should have a similar result:

```bash
$ for i in {1..20}; do curl localhost:8080; done
Hello from PID 8183
Hello from PID 8183
Hello from PID 8183
Hello from PID 8183
Hello from PID 8183
Hello from PID 8183
Hello from PID 8183
Hello from PID 8183
Hello from PID 8183
Hello from PID 8298
Hello from PID 8298
Hello from PID 8183
Hello from PID 8183
Hello from PID 8183
Hello from PID 8298
Hello from PID 8298
Hello from PID 8298
Hello from PID 8183
Hello from PID 8183
Hello from PID 8298
```

I hope you enjoyed this little article, for me this was very interesting and that is why I decided to share it with you <3

Related link:

* https://lwn.net/Articles/542629/
* https://man7.org/linux/man-pages/man7/socket.7.html
* https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/
