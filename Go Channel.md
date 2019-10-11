# Go Channel

[toc]

## hchan 数据结构

```
// channel 在 runtime 中的结构体
type hchan struct {
    // 队列中目前的元素计数
    qcount uint // total data in the queue
    // 环形队列的总大小，ch := make(chan int, 10) => 就是这里这个 10
    dataqsiz uint // size of the circular queue
    // void * 的内存 buffer 区域
    buf unsafe.Pointer // points to an array of dataqsiz elements
    
    // sizeof chan 中的数据
    elemsize uint16
    // runtime._type，代表 channel 中的元素类型的 runtime 结构体
    elemtype *_type // element type
    
    // 是否已被关闭
    closed uint32
    // 发送索引
    sendx uint // send index
    // 接收索引
    recvx uint // receive index
    
    // 接收 goroutine 对应的 sudog 队列
    recvq waitq // list of recv waiters
    // 发送 goroutine 对应的 sudog 队列
    sendq waitq // list of send waiters


    lock mutex
}

```

## 初始化

- 如果当前 Channel 中不存在缓冲区，那么就只会为 hchan 分配一段内存空间；
- 如果当前 Channel 中存储的类型不是指针类型，就会直接为当前的 Channel 和底层的数组分配一块连续的内存空间；
- 在默认情况下会单独为 hchan 和缓冲区分配内存；

```
func makechan(t *chantype, size int) *hchan {
    elem := t.elem

    // 如果 hchan 中的元素不包含有指针，那么就没什么和 GC 相关的信息了
    var c *hchan

    switch {
    case size == 0 || elem.size == 0:
        // 如果 channel 的缓冲区大小是 0: var a = make(chan int)
        // 或者 channel 中的元素大小是 0: struct{}{}
        // Queue or element size is zero.
        c = (*hchan)(mallocgc(hchanSize, nil, true))
        // Race detector uses this location for synchronization.
        c.buf = unsafe.Pointer(c)
    case elem.kind&kindNoPointers != 0:
        // Elements do not contain pointers.
        // Allocate hchan and buf in one call.
        // 通过位运算知道 channel 中的元素不包含指针
        // 占用的空间比较容易计算
        // 直接用 元素数*元素大小 + channel 必须的空间就行了
        // 这种情况下 gc 不会对 channel 中的元素进行 scan
        c = (*hchan)(mallocgc(hchanSize+uintptr(size)*elem.size, nil, true))
        c.buf = add(unsafe.Pointer(c), hchanSize)
    default:
        // Elements contain pointers.
        // 和上面那个 case 的写法的区别:调用了两次分配空间的函数 new/mallocgc
        c = new(hchan)
        c.buf = mallocgc(uintptr(size)*elem.size, elem, true)
    }

    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size)

    return c
}

```

## send 发送

- 直接发送:如果目标 Channel 没有被关闭并且已经有处于读等待的 Goroutine，那么chansend 函数会通过 dequeue 从 recvq 中取出最先陷入等待的 Goroutine 并直接向它发送数据;
- 缓冲区:向 Channel 中发送数据时遇到的第二种情况就是创建的 Channel 包含缓冲区并且 Channel 中的数据没有装满. 在这里我们首先会使用 chanbuf 计算出下一个可以放置待处理变量的位置，然后通过 typedmemmove 将发送的消息拷贝到缓冲区中并增加 sendx 索引和 qcount 计数器，在函数的最后会释放持有的锁。
- 阻塞发送:
	- 调用 getg 获取发送操作时使用的 Goroutine 协程；
	- 执行 acquireSudog 函数获取一个 sudog 结构体并设置这一次阻塞发送的相关信息，例如发送的 Channel、是否在 Select 控制结构中、发送数据所在的地址等；
	- 将刚刚创建并初始化的 sudog 结构体加入 sendq 等待队列，并设置到当前 Goroutine 的 waiting 上，表示 Goroutine 正在等待该 sudog 准备就绪；
	- 调用 goparkunlock 函数将当前的 Goroutine 更新成 Gwaiting 状态并解锁，该 Goroutine 可以被调用 goready 再次唤醒；

```
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	lock(&c.lock)

	if sg := c.recvq.dequeue(); sg != nil {
		// 寻找一个等待中的 receiver
		// 越过 channel 的 buffer
		// 直接把要发的数据拷贝给这个 receiver
		// 然后就返
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

    // qcount 是 buffer 中已塞进的元素数量
    // dataqsize 是 buffer 的总大小
    // 说明还有余量
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		
		// 将 goroutine 的数据拷贝到 buffer 中
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		
		// 环形队列，所以如果已经加到最大了，就回 0
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		
		// 将 buffer 的元素计数 +1
		c.qcount++
		unlock(&c.lock)
		return true
	}

	// 在 channel 上阻塞，receiver 会帮我们完成后续的工作
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0

	// 打包 sudog
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	
	// 将当前这个发送 goroutine 打包后的 sudog 入队到 channel 的 sendq 队列中
	c.sendq.enqueue(mysg)
	
	// 将这个发送 g 从 Grunning -> Gwaiting
    // 进入休眠
	goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)
	
	// 这里是被唤醒后要执行的代码
	KeepAlive(ep)

	gp.waiting = nil
	gp.param = nil

	mysg.c = nil
	releaseSudog(mysg)
	return true
}

func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    // receiver 的 sudog 已经在对应区域分配过空间
    // 我们只要把数据拷贝过去
    if sg.elem != nil {
        sendDirect(c.elemtype, sg, ep)
        sg.elem = nil
    }
    gp := sg.g
    unlockf()
    gp.param = unsafe.Pointer(sg)

    // Gwaiting -> Grunnable
    goready(gp, skip+1)
}

func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	// src is on our stack, dst is a slot on another stack.

	// Once we read sg.elem out of sg, it will no longer
	// be updated if the destination's stack gets copied (shrunk).
	// So make sure that no preemption points can happen between read & use.
	dst := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	// No need for cgo write barrier checks because dst is always
	// Go memory.
	memmove(dst, src, t.size)
}
```

## receive 接收

- 直接接收:当 Channel 的 sendq 队列中包含处于等待状态的 Goroutine 时，我们其实就会直接取出队列头的 Goroutine，这里处理的逻辑和发送时所差无几，只是发送数据时调用的是 send 函数，而这里是 recv 函数
- 缓冲区:另一种接收数据时遇到的情况就是，Channel 的缓冲区中已经包含了一些元素，在这时如果使用 <-ch 从 Channel 中接收元素，我们就会直接从缓冲区中 recvx 的索引位置中取出数据进行处理
- 阻塞接收:当 Channel 的 sendq 队列中不存在等待的 Goroutine 并且缓冲区中也不存在任何数据时，从管道中接收数据的操作在大多数时候就会变成一个阻塞的操作.

```
func chanrecv1(c *hchan, elem unsafe.Pointer) {
    chanrecv(c, elem, true)
}

//go:nosplit
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
    _, received = chanrecv(c, elem, true)
    return
}


func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    lock(&c.lock)

    // sender 队列中有 sudog 在等待
    // 直接从该 sudog 中获取数据拷贝到当前 g 即可
    if sg := c.sendq.dequeue(); sg != nil {
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true
    }

    if c.qcount > 0 {
        // Receive directly from queue
        qp := chanbuf(c, c.recvx)

        // 直接从 buffer 里拷贝数据
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        typedmemclr(c.elemtype, qp)
        // 接收索引 +1
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        // buffer 元素计数 -1
        c.qcount--
        unlock(&c.lock)
        return true, true
    }

    // no sender available: block on this channel.
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    // 打包成 sudog
    mysg.elem = ep
    mysg.waitlink = nil
    gp.waiting = mysg
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.param = nil
    // 进入 recvq 队列
    c.recvq.enqueue(mysg)

    // Grunning -> Gwaiting
    goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv, 3)

    // someone woke us up
    // 被唤醒
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    if mysg.releasetime > 0 {
        blockevent(mysg.releasetime-t0, 2)
    }
    closed := gp.param == nil
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    // 如果 channel 未被关闭，那就是真的 recv 到数据了
    return true, !closed
}


func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    if c.dataqsiz == 0 {
        if ep != nil {
            // copy data from sender
            recvDirect(c.elemtype, sg, ep)
        }
    } else {
        // Queue is full. Take the item at the
        // head of the queue. Make the sender enqueue
        // its item at the tail of the queue. Since the
        // queue is full, those are both the same slot.
        qp := chanbuf(c, c.recvx)

        // copy data from queue to receiver
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }

        // 虽然数据已经给了接受者，但是还是要在 chan 中记录一下
        typedmemmove(c.elemtype, qp, sg.elem)
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
    }
    sg.elem = nil
    gp := sg.g
    unlockf()
    gp.param = unsafe.Pointer(sg)
    if sg.releasetime != 0 {
        sg.releasetime = cputicks()
    }

    // Gwaiting -> Grunnable
    goready(gp, skip+1)
}


```

## close 关闭

在函数执行的最后会为所有被阻塞的 Goroutine 调用 goready 函数重新对这些协程进行调度.

```
func closechan(c *hchan) {
    // 上锁，这个锁的粒度比较大，一直到释放完所有的 sudog 才解锁
    lock(&c.lock)

    c.closed = 1

    var glist *g

    // release all readers
    for {
        sg := c.recvq.dequeue()
        // 弹出的 sudog 是 nil
        // 说明读队列已经空了
        if sg == nil {
            break
        }

        // sg.elem unsafe.Pointer，指向 sudog 的数据元素
        // 该元素可能在堆上分配，也可能在栈上
        if sg.elem != nil {
            // 释放对应的内存
            typedmemclr(c.elemtype, sg.elem)
            sg.elem = nil
        }
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }

        // 将 goroutine 入 glist
        // 为最后将全部 goroutine 都 ready 做准备
        gp := sg.g
        gp.param = nil
        gp.schedlink.set(glist)
        glist = gp
    }

    // release all writers (they will panic)
    // 将所有挂在 channel 上的 writer 从 sendq 中弹出
    // 该操作会使所有 writer panic
    for {
        sg := c.sendq.dequeue()
        if sg == nil {
            break
        }
        sg.elem = nil
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }

        // 将 goroutine 入 glist
        // 为最后将全部 goroutine 都 ready 做准备
        gp := sg.g
        gp.param = nil
        gp.schedlink.set(glist)
        glist = gp
    }

    // 在释放所有挂在 channel 上的读或写 sudog 时
    // 是一直在临界区的
    unlock(&c.lock)

    // Ready all Gs now that we've dropped the channel lock.
    for glist != nil {
        gp := glist
        glist = glist.schedlink.ptr()
        gp.schedlink = 0
        // 使 g 的状态切换到 Grunnable
        goready(gp, 3)
    }
}
```
