# Go 网络调用 netpoll

[TOC]

## 初始

socket，connect，listen，getsockopt 都有一个全局函数变量来表示。

hook_unix.go :

```go
    // Placeholders for socket system calls.
    socketFunc        func(int, int, int) (int, error)  = syscall.Socket
    connectFunc       func(int, syscall.Sockaddr) error = syscall.Connect
    listenFunc        func(int, int) error              = syscall.Listen
    getsockoptIntFunc func(int, int, int) (int, error)  = syscall.GetsockoptInt
```

这些 hook 主要是为了能够写测试，在测试代码中，socketFunc，connectFunc ... 都会被替换成测试专用函数，main_unix_test.go:

```go
func installTestHooks() {
    socketFunc = sw.Socket
    poll.CloseFunc = sw.Close
    connectFunc = sw.Connect
    listenFunc = sw.Listen
    poll.AcceptFunc = sw.Accept
    getsockoptIntFunc = sw.GetsockoptInt

    for _, fn := range extraTestHookInstallers {
        fn()
    }
}
```

用这种全局函数 hook，或者叫注册表的方式，可以实现类似于面向对象中的 interface 功能。不过因为不同平台提供的网络编程函数差别有些大，所以这里这些全局网络函数也就只是用来方便测试。

## 数据结构

### 编程接口

```
func Listen(net, laddr string) (Listener, error) 
func (*TCPListener) Accept (c Conn, err error) 

func Dial(network, address string) (Conn, error)

func (c *conn) Read(b []byte) (int, error) 
func (c *conn) Write(b []byte) (int, error) 

```

可以看到，对外接口重要的数据结构就是 `Listener`、`Conn`，一个是用户监听的描述符，一个是 `accept` 返回的用于读写的描述符。

### Listener 接口与 TCPListener

#### Listener 接口

```
type Listener interface {
	// Accept waits for and returns the next connection to the listener.
	Accept() (Conn, error)

	// Close closes the listener.
	// Any blocked Accept operations will be unblocked and return errors.
	Close() error

	// Addr returns the listener's network address.
	Addr() Addr
}

```

为了了解这个接口，我们先从 `Listen` 这个函数开始：

```
func Listen(network, address string) (Listener, error) {
	var lc ListenConfig
	return lc.Listen(context.Background(), network, address)
}

func (lc *ListenConfig) Listen(ctx context.Context, network, address string) (Listener, error) {
	addrs, err := DefaultResolver.resolveAddrList(ctx, "listen", network, address, nil)
	
	sl := &sysListener{
		ListenConfig: *lc,
		network:      network,
		address:      address,
	}
	var l Listener
	la := addrs.first(isIPv4)
	
	switch la := la.(type) {
	case *TCPAddr:
		l, err = sl.listenTCP(ctx, la)
	case *UnixAddr:
		l, err = sl.listenUnix(ctx, la)
	}
	
	return l, nil
}

```
#### ListenConfig

`ListenConfig` 是一个用于配置 `Listener` 的数据结构：

```
type ListenConfig struct {
	Control func(network, address string, c syscall.RawConn) error

	KeepAlive time.Duration
}


func (lc *ListenConfig) Listen(ctx context.Context, network, address string) (Listener, error) 

func (lc *ListenConfig) ListenPacket(ctx context.Context, network, address string) (PacketConn, error)
```

它仅仅有两个函数，分别用于 stream 数据与 datagram 数据。结构体里面的两个成语变量，一个用于在 bind 之前初始化用于 listen 的文件描述符，一个用于控制 accept 之后新的连接的 `KeepAlive` 属性。

#### sysListener

接下来，就是 sysListener 这个数据结构：

```
type sysListener struct {
	ListenConfig
	network, address string
}
```
它专门用于为各种网络类型建立连接：

```
func (sl *sysListener) listenTCP(ctx context.Context, laddr *TCPAddr) (*TCPListener, error) 

func (sl *sysListener) listenUDP(ctx context.Context, laddr *UDPAddr) (*UDPConn, error)

func (sl *sysListener) listenUnix(ctx context.Context, laddr *UnixAddr) (*UnixListener, error)

func (sl *sysListener) listenUnixgram(ctx context.Context, laddr *UnixAddr) (*UnixConn, error)

func (sl *sysListener) listenIP(ctx context.Context, laddr *IPAddr) (*IPConn, error)

```

#### TCPListener

对于 TCP 的连接来说，最重要的那就是 `TCPListener`:

```
func (sl *sysListener) listenTCP(ctx context.Context, laddr *TCPAddr) (*TCPListener, error) {
	...
	return &TCPListener{fd: fd, lc: sl.ListenConfig}, nil
}

type TCPListener struct {
	fd *netFD
	lc ListenConfig
}

```

我们暂时先不需要了解 internetSocket 建立监听描述符的过程，只关注总体的流程。下面我们看看拿到 TCPListener 后，使用 accept 函数的过程。

### Conn 接口与 TCPConn

#### Conn 接口

我们先开始研究编程接口：

```
func (*TCPListener) Accept (c Conn, err error) 

```

我们发现该函数返回的是 Conn 的类型，实际上它也是一个接口：


```
type Conn interface {
	Read(b []byte) (n int, err error)

	Write(b []byte) (n int, err error)

	Close() error

	LocalAddr() Addr

	RemoteAddr() Addr

	SetDeadline(t time.Time) error

	SetReadDeadline(t time.Time) error

	SetWriteDeadline(t time.Time) error
}

```

#### TCPConn

TCPListener 的 Accept 函数正是 Listener 接口的 Accept 实现。

```
func (l *TCPListener) Accept() (Conn, error) {
	c, err := l.accept()
	
	return c, nil
}

func (ln *TCPListener) accept() (*TCPConn, error) {
	fd, err := ln.fd.accept()

	tc := newTCPConn(fd)
	if ln.lc.KeepAlive >= 0 {
		setKeepAlive(fd, true)
		
		ka := ln.lc.KeepAlive
		if ln.lc.KeepAlive == 0 {
			ka = defaultTCPKeepAlive
		}
		
		setKeepAlivePeriod(fd, ka)
	}
	
	return tc, nil
}

type TCPConn struct {
	conn
}

type conn struct {
	fd *netFD
}
```

上文所说的 ListenConfig 长连接的配置，在这里的 accept 之后新生成的 fd 上进行设置。

我们看到实际上，TCPListener 返回的是 TCPConn 类型，它实现了 Conn 接口。

### Dialer 客户端

```
type Dialer struct {
	Timeout time.Duration

	Deadline time.Time

	// 真正dial时的本地地址，兼容各种类型(TCP、UDP...),如果为nil，则系统自动选择一个地址
	LocalAddr Addr
	
	DualStack bool // 双协议栈，即是否同时支持ipv4和ipv6.当network值为tcp时，dial函数会向host主机的v4和v6地址都发起连接
	
	FallbackDelay time.Duration // 当DualStack为真，ipv6会延后于ipv4发起，此字段即为延迟时间，默认为300ms

	KeepAlive time.Duration
	
	Cancel <-chan struct{} // 用于取消dial

	Control func(network, address string, c syscall.RawConn) error
}

func Dial(network, address string) (Conn, error)

```

### netFD

服务端通过 Listen 方法返回的 Listener 接口的实现和通过 listener 的 Accept 方法返回的 Conn 接口的实现都包含一个网络文件描述符 netFD，

netFD 中包含一个 poll.FD 数据结构，而 poll.FD 中包含两个重要的数据结构 Sysfd 和 pollDesc，前者是真正的系统文件描述符，后者对是底层事件驱动的封装，所有的读写超时等操作都是通过调用后者的对应方法实现的。

- 服务端的netFD在listen时会创建epoll的实例，并将listenFD加入epoll的事件队列
- netFD在accept时将返回的connFD也加入epoll的事件队列
- netFD在读写时出现syscall.EAGAIN错误，通过pollDesc将当前的goroutine park住，直到ready，从pollDesc的waitRead中返回

pollDesc 包含两个二元信号量, rg 和 wg, 分别用来 park 读、写的 goroutine
信号量可以是下面几种状态:

- nil：初始化状态
- pdWait：当前连接 fd 描述符已经添加到 poller，在 gopark 之前设置
- G pointer：gopark 调用 mcall 切换到 m0 后，pdWait 被替换为当前正在等待的 Goroutine
- pdReady： netpoll 已经通知完毕，程序已经将 rg、wg 的 G 链表取出

```go
const (
    pdReady uintptr = 1
    pdWait  uintptr = 2
)

const pollBlockSize = 4 * 1024

// Network file descriptor.
type netFD struct {
    pfd poll.FD

    // 下面这些元素在 Close 之前都是不可变的
    family      int
    sotype      int
    
    isConnected bool
    net         string
    
    laddr       Addr
    raddr       Addr
}

// FD 是对 file descriptor 的一个包装，内部的 Sysfd 就是 linux 下的
// file descriptor。net 和 os 包中使用这个类型来代表一个网络连接或者一个 OS 文件
type FD struct {
    // 对 sysfd 加锁，以使 Read 和 Write 方法串行执行
    fdmu fdMutex

    // 操作系统的 file descriptor。在关闭之前是不可变的
    Sysfd int

    // I/O poller.
    pd pollDesc

    // 不可变。表示当前这个 fd 是否是一个流，或者是一个基于包的 fd
    // 用来区分是 TCP 还是 UDP
    IsStream bool

    // 这个文件是否被设置为了 blocking 模式
    isBlocking bool
}

type pollDesc struct {
    link *pollDesc // in pollcache, protected by pollcache.lock

    lock    mutex // protects the following fields
    fd      uintptr

    rg      uintptr // pdReady, pdWait, G waiting for read or nil
    rt      timer   // read deadline timer (set if rt.f != nil)
    rd      int64   // read deadline
    
    wg      uintptr // pdReady, pdWait, G waiting for write or nil
    wt      timer   // write deadline timer
    wd      int64   // write deadline
}
```

## listen 流程

### Listen

listen 的入口函数就是 Listen:

```go
func Listen(network, address string) (Listener, error) {
    var lc ListenConfig
    return lc.Listen(context.Background(), network, address)
}

func (lc *ListenConfig) Listen(ctx context.Context, network, address string) (Listener, error) {
    addrs, err := DefaultResolver.resolveAddrList(ctx, "listen", network, address, nil)
    
    sl := &sysListener{
        ListenConfig: *lc,
        network:      network,
        address:      address,
    }
    var l Listener
    la := addrs.first(isIPv4)
    
    switch la := la.(type) {
    case *TCPAddr:
        l, err = sl.listenTCP(ctx, la)
    case *UnixAddr:
        l, err = sl.listenUnix(ctx, la)
    }
    
    return l, nil
}

```

```go
func (sl *sysListener) listenTCP(ctx context.Context, laddr *TCPAddr) (*TCPListener, error) {
	fd, err := internetSocket(ctx, sl.network, laddr, nil, syscall.SOCK_STREAM, 0, "listen", sl.ListenConfig.Control)
	if err != nil {
		return nil, err
	}
	return &TCPListener{fd: fd, lc: sl.ListenConfig}, nil
}
```

```go
func internetSocket(ctx context.Context, net string, laddr, raddr sockaddr, sotype, proto int, mode string) (fd *netFD, err error) {
    if (runtime.GOOS == "windows" || runtime.GOOS == "openbsd" || runtime.GOOS == "nacl") && mode == "dial" && raddr.isWildcard() {
        raddr = raddr.toLocal(net)
    }
    family, ipv6only := favoriteAddrFamily(net, laddr, raddr, mode)
    return socket(ctx, net, family, sotype, proto, ipv6only, laddr, raddr)
}
```

### socket 创建套接字

这个函数非常重要，它首先使用 sysSocket 进行系统调用，创建一个监听套接字。

然后使用 newFD 创建一个 netFD 类型的对象。

调用 fd.listenStream 执行监听系统调用。

```go
// socket returns a network file descriptor that is ready for
// asynchronous I/O using the network poller.
func socket(ctx context.Context, net string, family, sotype, proto int, ipv6only bool, laddr, raddr sockaddr) (fd *netFD, err error) {
    s, err := sysSocket(family, sotype, proto)
    if err != nil {
        return nil, err
    }
    if err = setDefaultSockopts(s, family, sotype, ipv6only); err != nil {
        poll.CloseFunc(s)
        return nil, err
    }
    if fd, err = newFD(s, family, sotype, net); err != nil {
        poll.CloseFunc(s)
        return nil, err
    }

    if laddr != nil && raddr == nil {
        switch sotype {
        // 基于流的协议
        case syscall.SOCK_STREAM, syscall.SOCK_SEQPACKET:
            if err := fd.listenStream(laddr, listenerBacklog); err != nil {
                fd.Close()
                return nil, err
            }
            return fd, nil
        // 基于数据报的协议
        case syscall.SOCK_DGRAM:
            if err := fd.listenDatagram(laddr); err != nil {
                fd.Close()
                return nil, err
            }
            return fd, nil
        }
    }
    if err := fd.dial(ctx, laddr, raddr); err != nil {
        fd.Close()
        return nil, err
    }
    return fd, nil
}
```

### listenStream 执行 listen 系统调用

这个函数中的 listenFunc 执行 listen 系统调用。

```go
func (fd *netFD) listenStream(laddr sockaddr, backlog int) error {
    if err := setDefaultListenerSockopts(fd.pfd.Sysfd); err != nil {
        return err
    }
    if lsa, err := laddr.sockaddr(fd.family); err != nil {
        return err
    } else if lsa != nil {
        // bind()
        if err := syscall.Bind(fd.pfd.Sysfd, lsa); err != nil {
            return os.NewSyscallError("bind", err)
        }
    }

    // listenFunc 是全局函数值，在 linux 下非测试环境被绑定到 syscall.Listen
    if err := listenFunc(fd.pfd.Sysfd, backlog); err != nil {
        return os.NewSyscallError("listen", err)
    }
    if err := fd.init(); err != nil {
        return err
    }
    lsa, _ := syscall.Getsockname(fd.pfd.Sysfd)
    fd.setAddr(fd.addrFunc()(lsa), nil)
    return nil
}
```

Go 的 listenTCP 一个函数就把 c 网络编程中 `socket()`，`bind()`，`listen()` 三步都完成了。大大减小了用户的心智负担。

这里有一点需要注意，listenStream 虽然提供了 backlog 的参数，但用户层是没有办法通过 Go 的代码来修改 listen 的 backlog 的。

```go
func maxListenerBacklog() int {
    fd, err := open("/proc/sys/net/core/somaxconn")
    if err != nil {
        return syscall.SOMAXCONN
    }
    defer fd.close()
    l, ok := fd.readLine()
    if !ok {
        return syscall.SOMAXCONN
    }
    f := getFields(l)
    n, _, ok := dtoi(f[0])
    if n == 0 || !ok {
        return syscall.SOMAXCONN
    }
    // Linux stores the backlog in a uint16.
    // Truncate number to avoid wrapping.
    // See issue 5030.
    if n > 1<<16-1 {
        n = 1<<16 - 1
    }
    return n
}
```

如上，在 linux 中，如果配置了 /proc/sys/net/core/somaxconn，那么就用这个值，如果没有配置，那么就使用 syscall 中的 SOMAXCONN:

```go
const (
    SOMAXCONN = 0x80 // 128
)
```

社区里有很多人吐槽，希望能有手段能修改这个值，不过看起来官方并不打算支持。所以现阶段只能通过修改 /proc/sys/net/core/somaxconn 来修改 listen 的 backlog。

### netFD 添加到 poller

在上面的 listen 流程的 socket 函数中会调用 newFD 来初始化一个 fd。

```go
func newFD(sysfd, family, sotype int, net string) (*netFD, error) {
    ret := &netFD{
        pfd: poll.FD{
            Sysfd:         sysfd,
            IsStream:      sotype == syscall.SOCK_STREAM,
            ZeroReadIsEOF: sotype != syscall.SOCK_DGRAM && sotype != syscall.SOCK_RAW,
        },
        family: family,
        sotype: sotype,
        net:    net,
    }
    return ret, nil
}
```

在 socket、bind、listen 三连发，都没有出错的情况下，会调用 fd.init():

```go
func (fd *netFD) init() error {
    return fd.pfd.Init(fd.net, true)
}
```

```go
// Init initializes the FD. The Sysfd field should already be set.
// This can be called multiple times on a single FD.
// The net argument is a network name from the net package (e.g., "tcp"),
// or "file".
// Set pollable to true if fd should be managed by runtime netpoll.
func (fd *FD) Init(net string, pollable bool) error {
    // We don't actually care about the various network types.
    if net == "file" {
        fd.isFile = true
    }
    if !pollable {
        fd.isBlocking = true
        return nil
    }
    return fd.pd.init(fd)
}
```

```go
func (pd *pollDesc) init(fd *FD) error {
    serverInit.Do(runtime_pollServerInit)
    ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
    if errno != 0 {
        if ctx != 0 {
            runtime_pollUnblock(ctx)
            runtime_pollClose(ctx)
        }
        return syscall.Errno(errno)
    }
    pd.runtimeCtx = ctx
    return nil
}
```

```go
//go:linkname poll_runtime_pollOpen internal/poll.runtime_pollOpen
func poll_runtime_pollOpen(fd uintptr) (*pollDesc, int) {
    pd := pollcache.alloc()
    lock(&pd.lock)
    if pd.wg != 0 && pd.wg != pdReady {
        throw("runtime: blocked write on free polldesc")
    }
    if pd.rg != 0 && pd.rg != pdReady {
        throw("runtime: blocked read on free polldesc")
    }
    pd.fd = fd
    pd.closing = false
    pd.seq++
    pd.rg = 0
    pd.rd = 0
    pd.wg = 0
    pd.wd = 0
    unlock(&pd.lock)

    var errno int32
    errno = netpollopen(fd, pd)
    return pd, int(errno)
}
```

每一个 fd 对会都应一个 pollDesc 结构，可以看到有 pollcache 提供一定程度的复用。

```go
func netpollopen(fd uintptr, pd *pollDesc) int32 {
    var ev epollevent
    ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
    *(**pollDesc)(unsafe.Pointer(&ev.data)) = pd
    return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}
```

pollDesc 初始化好之后，会当作 epoll event 的数据存储到 ev.data 中。 当有事件就续时，会取 ev.data，以判断是哪个 fd 可读/可写。



## accept 流程

### TCPListener.Accept 函数

```go
// Accept implements the Accept method in the Listener interface; it
// waits for the next call and returns a generic Conn.
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

```go
func (ln *TCPListener) accept() (*TCPConn, error) {
    fd, err := ln.fd.accept()
    if err != nil {
        return nil, err
    }
    return newTCPConn(fd), nil
}

func newTCPConn(fd *netFD) *TCPConn {
    c := &TCPConn{conn{fd}}
    setNoDelay(c.fd, true)
    return c
}
```

### netFD.Accept

```go
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
        fd.Close()
        return nil, err
    }
    lsa, _ := syscall.Getsockname(netfd.pfd.Sysfd)
    netfd.setAddr(netfd.addrFunc()(lsa), netfd.addrFunc()(rsa))
    return netfd, nil
}
```

### FD.Accept 执行非阻塞 accept

```go
// Accept wraps the accept network call.
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {
    if err := fd.readLock(); err != nil {
        return -1, nil, "", err
    }
    defer fd.readUnlock()

    if err := fd.pd.prepareRead(fd.isFile); err != nil {
        return -1, nil, "", err
    }
    for {
        s, rsa, errcall, err := accept(fd.Sysfd)
        if err == nil {
            return s, rsa, "", err
        }
        switch err {
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
}
```

```go
// Wrapper around the accept system call that marks the returned file
// descriptor as nonblocking and close-on-exec.
func accept(s int) (int, syscall.Sockaddr, string, error) {
    ns, sa, err := Accept4Func(s, syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC)
    // On Linux the accept4 system call was introduced in 2.6.28
    // kernel and on FreeBSD it was introduced in 10 kernel. If we
    // get an ENOSYS error on both Linux and FreeBSD, or EINVAL
    // error on Linux, fall back to using accept.
    switch err {
    case nil:
        return ns, sa, "", nil
    default: // errors other than the ones listed
        return -1, sa, "accept4", err
    case syscall.ENOSYS: // syscall missing
    case syscall.EINVAL: // some Linux use this instead of ENOSYS
    case syscall.EACCES: // some Linux use this instead of ENOSYS
    case syscall.EFAULT: // some Linux use this instead of ENOSYS
    }

    // See ../syscall/exec_unix.go for description of ForkLock.
    // It is probably okay to hold the lock across syscall.Accept
    // because we have put fd.sysfd into non-blocking mode.
    // However, a call to the File method will put it back into
    // blocking mode. We can't take that risk, so no use of ForkLock here.
    ns, sa, err = AcceptFunc(s)
    if err == nil {
        syscall.CloseOnExec(ns)
    }
    if err != nil {
        return -1, nil, "accept", err
    }
    if err = syscall.SetNonblock(ns, true); err != nil {
        CloseFunc(ns)
        return -1, nil, "setnonblock", err
    }
    return ns, sa, "", nil
}

```

可以看到，最终还是用 syscall 中的 accept4 或 accept 完成了系统调用。accept4 对比 accept 的优势是，可以通过一次系统调用完成 accept 和 nonblock flag 的两个目的。而使用 accept 的话，还要手动 syscall.SetNonblock。

### fd.pd.waitRead 非阻塞失败开始切换协程

这一部分和 read 阻塞流程相同，详见下文。

### netpollblock

### netfd.init 连接套接字添加到 poller

## Connect 客户端连接

### Dial

```
func Dial(network, address string) (Conn, error) {
	var d Dialer
	return d.Dial(network, address)
}

func (d *Dialer) Dial(network, address string) (Conn, error) {
	return d.DialContext(context.Background(), network, address)
}

func (d *Dialer) DialContext(ctx context.Context, network, address string) (Conn, error) {
    //d.deadline() 比较d.deadline、ctx.deadline、now+timeout，返回其中最小.如果都为空，返回0
	deadline := d.deadline(ctx, time.Now())
	
	if !deadline.IsZero() {
		if d, ok := ctx.Deadline(); !ok || deadline.Before(d) {
			subCtx, cancel := context.WithDeadline(ctx, deadline) // 设置新的超时context，deadline 时间一到，subCtx.Done() 立刻返回
			defer cancel()
			ctx = subCtx
		}
	}
	
	if oldCancel := d.Cancel; oldCancel != nil {
		subCtx, cancel := context.WithCancel(ctx)  // 使用新的 context
		defer cancel()
		go func() {
			select {
			case <-oldCancel:
				cancel()
			case <-subCtx.Done():
			}
		}()
		ctx = subCtx
	}

    // 解析IP地址，返回值是一个切片
	addrs, err := d.resolver().resolveAddrList(resolveCtx, "dial", network, address, d.LocalAddr)

	sd := &sysDialer{
		Dialer:  *d,
		network: network,
		address: address,
	}

	var primaries, fallbacks addrList
	if d.dualStack() && network == "tcp" {
		primaries, fallbacks = addrs.partition(isIPv4) // 将addrs分成两个切片，前者包含ipv4地址，后者包含ipv6地址
	} else {
		primaries = addrs
	}

	var c Conn
	if len(fallbacks) > 0 { //有ipv6的情况，v4和v6一起dial
		c, err = sd.dialParallel(ctx, primaries, fallbacks)
	} else {
		c, err = sd.dialSerial(ctx, primaries)
	}
	if err != nil {
		return nil, err
	}

	if tc, ok := c.(*TCPConn); ok && d.KeepAlive >= 0 {
		setKeepAlive(tc.fd, true)
		ka := d.KeepAlive
		if d.KeepAlive == 0 {
			ka = defaultTCPKeepAlive
		}
		setKeepAlivePeriod(tc.fd, ka)
		testHookSetKeepAlive(ka)
	}
	return c, nil
}

```
从上面代码看到，DialContext最终调用的是dialParallel和dialSerial,先看dialParallel，该函数将v4地址和v6地址分开，先尝试v4地址组，在dialer.fallbackDelay 时间后开始尝试v6地址组，每一组都是调用dialSerial(),让两组竞争：

```
func (sd *sysDialer) dialParallel(ctx context.Context, primaries, fallbacks addrList) (Conn, error) {
	if len(fallbacks) == 0 {
		return sd.dialSerial(ctx, primaries)
	}

	returned := make(chan struct{})
	defer close(returned)

	type dialResult struct {
		Conn
		error
		primary bool
		done    bool
	}
	results := make(chan dialResult) // unbuffered

	startRacer := func(ctx context.Context, primary bool) {
		ras := primaries
		if !primary {
			ras = fallbacks
		}
		c, err := sd.dialSerial(ctx, ras)
		select {
		case results <- dialResult{Conn: c, error: err, primary: primary, done: true}:
		case <-returned://提取返回，取消连接
			if c != nil {
				c.Close()
			}
		}
	}

	var primary, fallback dialResult

	// Start the main racer.
	primaryCtx, primaryCancel := context.WithCancel(ctx)
	defer primaryCancel()
	go startRacer(primaryCtx, true)//先尝试ipv4地址组

	// Start the timer for the fallback racer.
	fallbackTimer := time.NewTimer(sd.fallbackDelay())
	defer fallbackTimer.Stop()

	for {
		select {
		case <-fallbackTimer.C: // ipv6延迟时间到，开始尝试ipv6地址组
			fallbackCtx, fallbackCancel := context.WithCancel(ctx)
			defer fallbackCancel()
			go startRacer(fallbackCtx, false)

		case res := <-results://表示至少有一组已经建立连接
			if res.error == nil {
				return res.Conn, nil
			}
			if res.primary {
				primary = res
			} else {
				fallback = res
			}
			if primary.done && fallback.done {
				return nil, primary.error
			}
			if res.primary && fallbackTimer.Stop() {
				// If we were able to stop the timer, that means it
				// was running (hadn't yet started the fallback), but
				// we just got an error on the primary path, so start
				// the fallback immediately (in 0 nanoseconds).
				fallbackTimer.Reset(0)
			}
		}
	}
}

```

继续看dialSerial：

```
func (sd *sysDialer) dialSerial(ctx context.Context, ras addrList) (Conn, error) {
	var firstErr error // The error from the first address is most relevant.

	for i, ra := range ras {
		select {
		case <-ctx.Done(): // 先观察是否已经被取消
			return nil, &OpError{Op: "dial", Net: sd.network, Source: sd.LocalAddr, Addr: ra, Err: mapErr(ctx.Err())}
		default:
		}

		deadline, _ := ctx.Deadline()
		partialDeadline, err := partialDeadline(time.Now(), deadline, len(ras)-i)

		dialCtx := ctx
		if partialDeadline.Before(deadline) {
			var cancel context.CancelFunc
			dialCtx, cancel = context.WithDeadline(ctx, partialDeadline)
			defer cancel()
		}

		c, err := sd.dialSingle(dialCtx, ra)
		if err == nil {
			return c, nil
		}
		if firstErr == nil {
			firstErr = err
		}
	}

	if firstErr == nil {
		firstErr = &OpError{Op: "dial", Net: sd.network, Source: nil, Addr: nil, Err: errMissingAddress}
	}
	return nil, firstErr
}

```

### sysDialer.dialSingle

```
func (sd *sysDialer) dialSingle(ctx context.Context, ra Addr) (c Conn, err error) {
	la := sd.LocalAddr
	switch ra := ra.(type) {
	case *TCPAddr:
		la, _ := la.(*TCPAddr)
		c, err = sd.dialTCP(ctx, la, ra)
	case *UDPAddr:
		la, _ := la.(*UDPAddr)
		c, err = sd.dialUDP(ctx, la, ra)
	case *IPAddr:
		la, _ := la.(*IPAddr)
		c, err = sd.dialIP(ctx, la, ra)
	case *UnixAddr:
		la, _ := la.(*UnixAddr)
		c, err = sd.dialUnix(ctx, la, ra)
	default:
		return nil, &OpError{Op: "dial", Net: sd.network, Source: la, Addr: ra, Err: &AddrError{Err: "unexpected address type", Addr: sd.address}}
	}
	if err != nil {
		return nil, &OpError{Op: "dial", Net: sd.network, Source: la, Addr: ra, Err: err} // c is non-nil interface containing nil pointer
	}
	return c, nil
}

func (sd *sysDialer) dialTCP(ctx context.Context, laddr, raddr *TCPAddr) (*TCPConn, error) {
	return sd.doDialTCP(ctx, laddr, raddr)
}

func (sd *sysDialer) doDialTCP(ctx context.Context, laddr, raddr *TCPAddr) (*TCPConn, error) {
	fd, err := internetSocket(ctx, sd.network, laddr, raddr, syscall.SOCK_STREAM, 0, "dial", sd.Dialer.Control)
	
	for i := 0; i < 2 && (laddr == nil || laddr.Port == 0) && (selfConnect(fd, err) || spuriousENOTAVAIL(err)); i++ {
		if err == nil {
			fd.Close()
		}
		fd, err = internetSocket(ctx, sd.network, laddr, raddr, syscall.SOCK_STREAM, 0, "dial", sd.Dialer.Control)
	}

	return newTCPConn(fd), nil
}
```

参数里的ctx自然不言而喻了，是为了控制请求超时取消请求释放资源的；laddr是 local address ， raddr是指 remote address；返回值这里会得到 TCPConn。代码不长，就是调用了 internetSocket得到一个文件描述符，并用其新建一个conn返回。但这里我想多说几句，因为不难发现， internetSocket可能会被调用多次，为什么呢？

首先我们需要知道 Tcp 有一个极少使用的机制，叫simultaneous connection（同时连接）。正常的连接是：A主机 dial B主机，B主机 listen。 而同时连接则是： A 向 B dial 同时 B 向 A dial，那么 A 和 B 都不需要监听。

我们知道，当 传入 dial 函数的参数laddr==raddr时，内核会拒绝dial。但如果传入的laddr为nil，kernel 会自动选择一个本机端口，这时候有可能会使得新的laddr==raddr,这个时候，kernel不会拒绝dial，并且这个dial会成功，原因是就simultaneous connection，这可能是kernel的bug。所以会判断是否是 selfConnect或者spuriousENOTAVAIL(spurious error not avail)来判断上一次调用internetSocket返回的 err 类型，在特定的情况下重新尝试internetSocket.

```
func internetSocket(ctx context.Context, net string, laddr, raddr sockaddr, sotype, proto int, mode string, ctrlFn func(string, string, syscall.RawConn) error) (fd *netFD, err error) {
	family, ipv6only := favoriteAddrFamily(net, laddr, raddr, mode)
	return socket(ctx, net, family, sotype, proto, ipv6only, laddr, raddr, ctrlFn)
}

func socket(ctx context.Context, net string, family, sotype, proto int, ipv6only bool, laddr, raddr sockaddr, ctrlFn func(string, string, syscall.RawConn) error) (fd *netFD, err error) {
	s, err := sysSocket(family, sotype, proto)
	if err != nil {
		return nil, err
	}
	if err = setDefaultSockopts(s, family, sotype, ipv6only); err != nil {
		poll.CloseFunc(s)
		return nil, err
	}
	if fd, err = newFD(s, family, sotype, net); err != nil {
		poll.CloseFunc(s)
		return nil, err
	}

	if err := fd.dial(ctx, laddr, raddr, ctrlFn); err != nil {
		fd.Close()
		return nil, err
	}
	return fd, nil
}
```

```
func (fd *netFD) dial(ctx context.Context, laddr, raddr sockaddr, ctrlFn func(string, string, syscall.RawConn) error) error {
	if ctrlFn != nil {
		c, err := newRawConn(fd)
		if err != nil {
			return err
		}
		var ctrlAddr string
		if raddr != nil {
			ctrlAddr = raddr.String()
		} else if laddr != nil {
			ctrlAddr = laddr.String()
		}
		if err := ctrlFn(fd.ctrlNetwork(), ctrlAddr, c); err != nil {
			return err
		}
	}
	
	var rsa syscall.Sockaddr  // remote address from the user
	var crsa syscall.Sockaddr // remote address we actually connected to
	if raddr != nil {
		if rsa, err = raddr.sockaddr(fd.family); err != nil {
			return err
		}
		if crsa, err = fd.connect(ctx, lsa, rsa); err != nil {
			return err
		}
		fd.isConnected = true
	} else {
		if err := fd.init(); err != nil {
			return err
		}
	}

	lsa, _ = syscall.Getsockname(fd.pfd.Sysfd)
	if crsa != nil {
		fd.setAddr(fd.addrFunc()(lsa), fd.addrFunc()(crsa))
	} else if rsa, _ = syscall.Getpeername(fd.pfd.Sysfd); rsa != nil {
		fd.setAddr(fd.addrFunc()(lsa), fd.addrFunc()(rsa))
	} else {
		fd.setAddr(fd.addrFunc()(lsa), raddr)
	}
	return nil
}

```

### fd.connect

```
func (fd *netFD) connect(ctx context.Context, la, ra syscall.Sockaddr) (rsa syscall.Sockaddr, ret error) {
    // 先尝试非阻塞 connect
	switch err := connectFunc(fd.pfd.Sysfd, ra); err {
	case syscall.EINPROGRESS, syscall.EALREADY, syscall.EINTR:
	case nil, syscall.EISCONN:
		select {
		case <-ctx.Done():
			return nil, mapErr(ctx.Err())
		default:
		}
		if err := fd.pfd.Init(fd.net, true); err != nil { // 初始化到 poller
			return nil, err
		}
		runtime.KeepAlive(fd)
		return nil, nil // 成功建立连接，返回
	case syscall.EINVAL:
		fallthrough
	default:
		return nil, os.NewSyscallError("connect", err)
	}
	
	// 初始化，放置到 netpoll
	if err := fd.pfd.Init(fd.net, true); err != nil {
		return nil, err
	}

	for {
	    // 阻塞调度，等待建立连接
		if err := fd.pfd.WaitWrite(); err != nil {
			select {
			case <-ctx.Done():
				return nil, mapErr(ctx.Err())
			default:
			}
			return nil, err
		}
		nerr, err := getsockoptIntFunc(fd.pfd.Sysfd, syscall.SOL_SOCKET, syscall.SO_ERROR)
		if err != nil {
			return nil, os.NewSyscallError("getsockopt", err)
		}
		switch err := syscall.Errno(nerr); err {
		case syscall.EINPROGRESS, syscall.EALREADY, syscall.EINTR:
		case syscall.EISCONN:
			return nil, nil
		case syscall.Errno(0):
			// The runtime poller can wake us up spuriously;
			// see issues 14548 and 19289. Check that we are
			// really connected; if not, wait again.
			if rsa, err := syscall.Getpeername(fd.pfd.Sysfd); err == nil { // 建立连接成功
				return rsa, nil
			}
		default:
			return nil, os.NewSyscallError("connect", err)
		}
		runtime.KeepAlive(fd)
	}
}

```

## Read 流程

### conn.Read

```go
func (c *conn) ok() bool { return c != nil && c.fd != nil }

// Implementation of the Conn interface.

// Read implements the Conn Read method.
func (c *conn) Read(b []byte) (int, error) {
    if !c.ok() {
        return 0, syscall.EINVAL
    }
    n, err := c.fd.Read(b)
    if err != nil && err != io.EOF {
        err = &OpError{Op: "read", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
    }
    return n, err
}

```

### netFD.Read

```go
func (fd *netFD) Read(buf []byte) (int, error) {
    n, err := fd.pfd.Read(buf)
    runtime.KeepAlive(fd)
    return n, wrapSyscallError("wsarecv", err)
}
```

### FD.Read 尝试非阻塞读

```go
// Read implements io.Reader.
func (fd *FD) Read(p []byte) (int, error) {
    if err := fd.readLock(); err != nil {
        return 0, err
    }
    defer fd.readUnlock()
    if len(p) == 0 {
        return 0, nil
    }

    if err := fd.pd.prepareRead(fd.isFile); err != nil {
        return 0, err
    }
    if fd.IsStream && len(p) > maxRW {
        p = p[:maxRW]
    }
    for {
        // 第一次调用 syscall.Read 之后，如果读到了数据
        // 那么直接就返回了
        n, err := syscall.Read(fd.Sysfd, p)
        if err != nil {
            n = 0
            // 如果 os 返回 EAGAIN，说明可能暂时没数据
            // 判断 fd 是 pollable 的话，说明可以走 poll 流程
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

### pollDesc.waitRead 阻塞调度

waitRead 并不是真正的阻塞，而是直接从当前的 G 调度到其他可运行的 G 去运行，等待着 netpoll 的通知，再回来。

```go
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

runtime_pollWait 是用 `go:linkname` 来链接期链接到的函数，实现在 `runtime/netpoll.go` 中:

```go
//go:linkname poll_runtime_pollWait internal/poll.runtime_pollWait
func poll_runtime_pollWait(pd *pollDesc, mode int) int {
    err := netpollcheckerr(pd, int32(mode))
    if err != 0 {
        return err
    }

    for !netpollblock(pd, int32(mode), false) {
        err = netpollcheckerr(pd, int32(mode))
        if err != 0 {
            return err
        }
    }
    return 0
}
```
### netpollblock 设置 rg/wg 为 pdWait

本函数主要工作有两个：

- 一个是将 pd.rg 或者 pd.wg 从初始状态转换为 pdWait。
- 调用 gopark 调度

```go
// returns true if IO is ready, or false if timedout or closed
// waitio - wait only for completed IO, ignore errors
func netpollblock(pd *pollDesc, mode int32, waitio bool) bool {
    gpp := &pd.rg
    if mode == 'w' {
        gpp = &pd.wg
    }

    // set the gpp semaphore to WAIT
    for {
        old := *gpp
        if old == pdReady {
            *gpp = 0
            return true
        }
        if old != 0 {
            throw("runtime: double wait")
        }
        if atomic.Casuintptr(gpp, 0, pdWait) {
            break
        }
    }

    // need to recheck error states after setting gpp to WAIT
    // this is necessary because runtime_pollUnblock/runtime_pollSetDeadline/deadlineimpl
    // do the opposite: store to closing/rd/wd, membarrier, load of rg/wg
    if waitio || netpollcheckerr(pd, mode) == 0 {
        gopark(netpollblockcommit, unsafe.Pointer(gpp), "IO wait", traceEvGoBlockNet, 5)
    }
    // be careful to not lose concurrent READY notification
    old := atomic.Xchguintptr(gpp, 0)
    if old > pdWait {
        throw("runtime: corrupted polldesc")
    }
    return old == pdReady
}
```

gopark 将当前 g 挂起，等待就绪事件到达之后再继续执行。

### netpollblockcommit 设置 rg/wg 为当前 Goroutine

在上面读写流程，syscall.Read 或者 syscall.Write 返回 EAGAIN 时，会挂起当前正在进行这个读/写操作的 g，具体是调用 gopark，并执行 netpollblockcommit，并将 gpp 挂起，netpollblockcommit 比较简单:

```go
func netpollblockcommit(gp *g, gpp unsafe.Pointer) bool {
    r := atomic.Casuintptr((*uintptr)(gpp), pdWait, uintptr(unsafe.Pointer(gp)))
    if r {
        // Bump the count of goroutines waiting for the poller.
        // The scheduler uses this to decide whether to block
        // waiting for the poller if there is nothing else to do.
        atomic.Xadd(&netpollWaiters, 1)
    }
    return r
}
```

EAGAIN 的时候:

```go
gopark(netpollblockcommit, unsafe.Pointer(gpp), "IO wait", traceEvGoBlockNet, 5)
```

至于唤醒流程，当调度器在 findrunnable、startTheWorldWithSema 或者 sysmon 中调用 netpoll 函数时，会获取到上面说的就绪的 g 列表。把这些 g 的 bp/sp/pc 都从 g.gobuf 中恢复出来，就可以继续执行它们的 Read/Write 操作了。

因为调度中有讲，这里就不赘述了。

## Write 流程

### conn.Write

```go
// Write implements the Conn Write method.
func (c *conn) Write(b []byte) (int, error) {
    if !c.ok() {
        return 0, syscall.EINVAL
    }
    n, err := c.fd.Write(b)
    if err != nil {
        err = &OpError{Op: "write", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
    }
    return n, err
}

```

### netFD.Write

```go
func (fd *netFD) Write(buf []byte) (int, error) {
    n, err := fd.pfd.Write(buf)
    runtime.KeepAlive(fd)
    return n, wrapSyscallError("wsasend", err)
}
```

### FD.Write 循环非阻塞写

```go
// Write implements io.Writer.
func (fd *FD) Write(p []byte) (int, error) {
    if err := fd.writeLock(); err != nil {
        return 0, err
    }
    defer fd.writeUnlock()
    if err := fd.pd.prepareWrite(fd.isFile); err != nil {
        return 0, err
    }
    var nn int
    for {
        max := len(p)
        if fd.IsStream && max-nn > maxRW {
            max = nn + maxRW
        }
        n, err := syscall.Write(fd.Sysfd, p[nn:max])
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
        if err != nil {
            return nn, err
        }
        if n == 0 {
            return nn, io.ErrUnexpectedEOF
        }
    }
}
```

### pd.waitWrite 阻塞调度

内核的写缓冲区满，这里的 syscall.Write 就会返回 EAGAIN。

```go
func (pd *pollDesc) waitWrite(isFile bool) error {
    return pd.wait('w', isFile)
}

func (pd *pollDesc) wait(mode int, isFile bool) error {
    if pd.runtimeCtx == 0 {
        return errors.New("waiting for unsupported file type")
    }
    res := runtime_pollWait(pd.runtimeCtx, mode)
    return convertErr(res, isFile)
}
```

后面的流程就和 Read 完全一致了。

### netpollblock

## 就续通知

### netpoll

```go
// poll 已经就绪的网络连接
// 返回那些已经可以跑的 goroutine 列表
func netpoll(block bool) *g {
    if epfd == -1 {
        return nil
    }
    waitms := int32(-1)
    if !block {
        waitms = 0
    }
    var events [128]epollevent
retry:
    n := epollwait(epfd, &events[0], int32(len(events)), waitms)
    if n < 0 {
        if n != -_EINTR {
            println("runtime: epollwait on fd", epfd, "failed with", -n)
            throw("runtime: netpoll failed")
        }
        goto retry
    }
    var gp guintptr
    for i := int32(0); i < n; i++ {
        ev := &events[i]
        if ev.events == 0 {
            continue
        }
        var mode int32
        if ev.events&(_EPOLLIN|_EPOLLRDHUP|_EPOLLHUP|_EPOLLERR) != 0 {
            mode += 'r'
        }
        if ev.events&(_EPOLLOUT|_EPOLLHUP|_EPOLLERR) != 0 {
            mode += 'w'
        }
        if mode != 0 {
            pd := *(**pollDesc)(unsafe.Pointer(&ev.data))

            netpollready(&gp, pd, mode)
        }
    }
    if block && gp == 0 {
        goto retry
    }
    return gp.ptr()
}

// 让 pd 就续，新的可以运行的 goroutine 会 set 到 wg/rg
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
### netpollunblock 获取 netpoll 中的 rg/wg 并转为 pdReady

```
// 按照 mode 把 pollDesc 的 wg 或者 rg 捞出来，返回
func netpollunblock(pd *pollDesc, mode int32, ioready bool) *g {
	gpp := &pd.rg
	if mode == 'w' {
		gpp = &pd.wg
	}

	for {
		old := *gpp
		if old == pdReady {
			return nil
		}
		if old == 0 && !ioready {
			// Only set READY for ioready. runtime_pollWait
			// will check for timeout/cancel before waiting.
			return nil
		}
		var new uintptr
		if ioready {
			new = pdReady
		}
		if atomic.Casuintptr(gpp, old, new) {
			if old == pdReady || old == pdWait {
				old = 0
			}
			return (*g)(unsafe.Pointer(old))
		}
	}
}
```

三个函数配合完成就续后唤醒对应的 g 的工作，netpollunblock 从 pollDesc 中捞出 rg/wg，netpollready 然后再把所有的 rg/wg 通过 schedlink 串成一个链表。findrunnable 之类需要 g 的场景下，调度器会主动调用 netpoll 函数来寻找是否有已经就绪的网络事件对应的 g。

netpoll 这个函数是平台相关的，实现在对应的 netpoll_epoll、netpoll_kqueue 文件中。

## 读写超时

### Conn.SetReadDeadline/SetWriteDeadline

```go
// 设置底层连接的读超时
// 超时时间是 0 值的话永远都不会超时
func (c *Conn) SetReadDeadline(t time.Time) error {
    return c.conn.SetReadDeadline(t)
}

// 设置底层连接的读超时
// 超时时间是 0 值的话永远都不会超时
// 写超时发生之后， TLS 状态会被破坏，未来的所有写都会返回相同的错误
func (c *Conn) SetWriteDeadline(t time.Time) error {
    return c.conn.SetWriteDeadline(t)
}
```

```go
// 实现 Conn 接口中的方法
func (c *conn) SetReadDeadline(t time.Time) error {
	if !c.ok() {
		return syscall.EINVAL
	}
	if err := c.fd.SetReadDeadline(t); err != nil {
		return &OpError{Op: "set", Net: c.fd.net, Source: nil, Addr: c.fd.laddr, Err: err}
	}
	return nil
}

// 实现 Conn 接口中的方法
func (c *conn) SetWriteDeadline(t time.Time) error {
	if !c.ok() {
		return syscall.EINVAL
	}
	if err := c.fd.SetWriteDeadline(t); err != nil {
		return &OpError{Op: "set", Net: c.fd.net, Source: nil, Addr: c.fd.laddr, Err: err}
	}
	return nil
}
```

### FD.SetReadDeadline/SetWriteDeadline

```go
// 设置关联 fd 的读取 deadline
func (fd *FD) SetReadDeadline(t time.Time) error {
    return setDeadlineImpl(fd, t, 'r')
}

// 设置关联 fd 的写入 deadline
func (fd *FD) SetWriteDeadline(t time.Time) error {
    return setDeadlineImpl(fd, t, 'w')
}
```

```go
func setDeadlineImpl(fd *FD, t time.Time, mode int) error {
    diff := int64(time.Until(t))
    d := runtimeNano() + diff
    if d <= 0 && diff > 0 {
        // 如果用户提供了未来的 deadline，但是 delay 计算溢出了，那么设置 dealine 到最大的可能的值
        d = 1<<63 - 1
    }
    if t.IsZero() {
        // IsZero reports whether t represents the zero time instant,
        // January 1, year 1, 00:00:00 UTC.
        // func (t Time) IsZero() bool {
        //     return t.sec() == 0 && t.nsec() == 0
        // }
        d = 0
    }
    if err := fd.incref(); err != nil {
        return err
    }
    defer fd.decref()
    if fd.pd.runtimeCtx == 0 {
        return ErrNoDeadline
    }
    runtime_pollSetDeadline(fd.pd.runtimeCtx, d, mode)
    return nil
}
```

### poll_runtime_pollSetDeadline 设置超时 timer

```go
//go:linkname poll_runtime_pollSetDeadline internal/poll.runtime_pollSetDeadline
func poll_runtime_pollSetDeadline(pd *pollDesc, d int64, mode int) {
	lock(&pd.lock)
	if pd.closing {
		unlock(&pd.lock)
		return
	}
	rd0, wd0 := pd.rd, pd.wd
	combo0 := rd0 > 0 && rd0 == wd0
	if d > 0 {
		d += nanotime()
		if d <= 0 {
			// If the user has a deadline in the future, but the delay calculation
			// overflows, then set the deadline to the maximum possible value.
			d = 1<<63 - 1
		}
	}
	if mode == 'r' || mode == 'r'+'w' {
		pd.rd = d
	}
	if mode == 'w' || mode == 'r'+'w' {
		pd.wd = d
	}
	combo := pd.rd > 0 && pd.rd == pd.wd
	rtf := netpollReadDeadline
	if combo {
		rtf = netpollDeadline
	}
	if pd.rt.f == nil {
		if pd.rd > 0 {
			pd.rt.f = rtf
			pd.rt.when = pd.rd
			// Copy current seq into the timer arg.
			// Timer func will check the seq against current descriptor seq,
			// if they differ the descriptor was reused or timers were reset.
			pd.rt.arg = pd
			pd.rt.seq = pd.rseq
			addtimer(&pd.rt)
		}
	} else if pd.rd != rd0 || combo != combo0 {
		pd.rseq++ // invalidate current timers
		if pd.rd > 0 {
			modtimer(&pd.rt, pd.rd, 0, rtf, pd, pd.rseq)
		} else {
			deltimer(&pd.rt)
			pd.rt.f = nil
		}
	}
	if pd.wt.f == nil {
		if pd.wd > 0 && !combo {
			pd.wt.f = netpollWriteDeadline
			pd.wt.when = pd.wd
			pd.wt.arg = pd
			pd.wt.seq = pd.wseq
			addtimer(&pd.wt)
		}
	} else if pd.wd != wd0 || combo != combo0 {
		pd.wseq++ // invalidate current timers
		if pd.wd > 0 && !combo {
			modtimer(&pd.wt, pd.wd, 0, netpollWriteDeadline, pd, pd.wseq)
		} else {
			deltimer(&pd.wt)
			pd.wt.f = nil
		}
	}

	// 如果发现超时时间已经是过去了，那么提前取出
	var rg, wg *g
	if pd.rd < 0 || pd.wd < 0 {
		atomic.StorepNoWB(noescape(unsafe.Pointer(&wg)), nil) // full memory barrier between stores to rd/wd and load of rg/wg in netpollunblock
		if pd.rd < 0 {
			rg = netpollunblock(pd, 'r', false)
		}
		if pd.wd < 0 {
			wg = netpollunblock(pd, 'w', false)
		}
	}
	unlock(&pd.lock)
	if rg != nil {
		netpollgoready(rg, 3)
	}
	if wg != nil {
		netpollgoready(wg, 3)
	}
}
```

根据 read deadline 和 write deadline 给要插入时间堆的 timer 设置不同的回调函数。

```go
func netpollDeadline(arg interface{}, seq uintptr) {
    netpolldeadlineimpl(arg.(*pollDesc), seq, true, true)
}

func netpollReadDeadline(arg interface{}, seq uintptr) {
    netpolldeadlineimpl(arg.(*pollDesc), seq, true, false)
}

func netpollWriteDeadline(arg interface{}, seq uintptr) {
    netpolldeadlineimpl(arg.(*pollDesc), seq, false, true)
}
```

### netpolldeadlineimpl 超时回调函数(被调用表明已超时)

调用最终的实现函数:

```go
func netpolldeadlineimpl(pd *pollDesc, seq uintptr, read, write bool) {
    lock(&pd.lock)
    // Seq arg is seq when the timer was set.
    // If it's stale, ignore the timer event.
    if seq != pd.seq {
        // The descriptor was reused or timers were reset.
        unlock(&pd.lock)
        return
    }
    var rg *g
    if read {
        if pd.rd <= 0 || pd.rt.f == nil {
            throw("runtime: inconsistent read deadline")
        }
        pd.rd = -1
        atomicstorep(unsafe.Pointer(&pd.rt.f), nil) // full memory barrier between store to rd and load of rg in netpollunblock
        rg = netpollunblock(pd, 'r', false)
    }
    var wg *g
    if write {
        if pd.wd <= 0 || pd.wt.f == nil && !read {
            throw("runtime: inconsistent write deadline")
        }
        pd.wd = -1
        atomicstorep(unsafe.Pointer(&pd.wt.f), nil) // full memory barrier between store to wd and load of wg in netpollunblock
        wg = netpollunblock(pd, 'w', false)
    }
    unlock(&pd.lock)
    // rg 和 wg 是通过 netpollunblock 从 pollDesc 结构中捞出来的
    if rg != nil {
        // 恢复 goroutine 执行现场
        // 继续执行
        netpollgoready(rg, 0)
    }
    if wg != nil {
        // 恢复 goroutine 执行现场
        // 继续执行
        netpollgoready(wg, 0)
    }
}
```

## 连接关闭

### conn.Close

```go
// Close closes the connection.
func (c *conn) Close() error {
    if !c.ok() {
        return syscall.EINVAL
    }
    err := c.fd.Close()
    if err != nil {
        err = &OpError{Op: "close", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
    }
    return err
}
```

### netFD.Close

```go
func (fd *netFD) Close() error {
    runtime.SetFinalizer(fd, nil)
    return fd.pfd.Close()
}
```

### FD.Close

```go
// Close closes the FD. The underlying file descriptor is closed by the
// destroy method when there are no remaining references.
func (fd *FD) Close() error {
    if !fd.fdmu.increfAndClose() {
        return errClosing(fd.isFile)
    }

    // Unblock any I/O.  Once it all unblocks and returns,
    // so that it cannot be referring to fd.sysfd anymore,
    // the final decref will close fd.sysfd. This should happen
    // fairly quickly, since all the I/O is non-blocking, and any
    // attempts to block in the pollDesc will return errClosing(fd.isFile).
    fd.pd.evict()

    // The call to decref will call destroy if there are no other
    // references.
    err := fd.decref()

    // Wait until the descriptor is closed. If this was the only
    // reference, it is already closed. Only wait if the file has
    // not been set to blocking mode, as otherwise any current I/O
    // may be blocking, and that would block the Close.
    if !fd.isBlocking {
        runtime_Semacquire(&fd.csema)
    }

    return err
}
```

### pollDesc.evict

```go
// Evict evicts fd from the pending list, unblocking any I/O running on fd.
func (pd *pollDesc) evict() {
    if pd.runtimeCtx == 0 {
        return
    }
    runtime_pollUnblock(pd.runtimeCtx)
}
```

### poll_runtime_pollUnblock 提取取出被阻塞的 G

```go
//go:linkname poll_runtime_pollUnblock internal/poll.runtime_pollUnblock
func poll_runtime_pollUnblock(pd *pollDesc) {
    lock(&pd.lock)
    if pd.closing {
        throw("runtime: unblock on closing polldesc")
    }
    pd.closing = true
    pd.seq++
    var rg, wg *g
    atomicstorep(unsafe.Pointer(&rg), nil) // full memory barrier between store to closing and read of rg/wg in netpollunblock
    rg = netpollunblock(pd, 'r', false)
    wg = netpollunblock(pd, 'w', false)
    if pd.rt.f != nil {
        deltimer(&pd.rt)
        pd.rt.f = nil
    }
    if pd.wt.f != nil {
        deltimer(&pd.wt)
        pd.wt.f = nil
    }
    unlock(&pd.lock)
    if rg != nil {
        netpollgoready(rg, 3)
    }
    if wg != nil {
        netpollgoready(wg, 3)
    }
}
```

### fd.decref 销毁连接套接字

```
func (fd *FD) decref() error {
	if fd.fdmu.decref() {
		return fd.destroy()
	}
	return nil
}

func (fd *FD) destroy() error {
	// Poller may want to unregister fd in readiness notification mechanism,
	// so this must be executed before CloseFunc.
	fd.pd.close()
	err := CloseFunc(fd.Sysfd)
	fd.Sysfd = -1
	runtime_Semrelease(&fd.csema)
	return err
}

var CloseFunc func(int) error = syscall.Close

func (pd *pollDesc) close() {
	if pd.runtimeCtx == 0 {
		return
	}
	runtime_pollClose(pd.runtimeCtx)
	pd.runtimeCtx = 0
}
```

### poll_runtime_pollClose 取消 netpoll 

```
func poll_runtime_pollClose(pd *pollDesc) {
	if !pd.closing {
		throw("runtime: close polldesc w/o unblock")
	}
	if pd.wg != 0 && pd.wg != pdReady {
		throw("runtime: blocked write on closing polldesc")
	}
	if pd.rg != 0 && pd.rg != pdReady {
		throw("runtime: blocked read on closing polldesc")
	}
	netpollclose(pd.fd)
	pollcache.free(pd)
}

func netpollclose(fd uintptr) int32 {
	var ev epollevent
	return -epollctl(epfd, _EPOLL_CTL_DEL, int32(fd), &ev)
}

```
