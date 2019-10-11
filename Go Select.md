# Go Select

[TOC]

## 数据结构

select 在 Go 语言的源代码中其实不存在任何的结构体表示，但是 select 控制结构中 case 却使用了 scase 结构体来表示。

由于非 default 的 case 中都与 Channel 的发送和接收数据有关，所以在 scase 结构体中也包含一个 c 字段用于存储 case 中使用的 Channel，elem 是用于接收或者发送数据的变量地址、kind 表示当前 case 的种类，总共包含以下四种:

```
const (
    // scase.kind
    // send 或者 recv 发生在一个 nil channel 上，就有可能出现这种情况
    caseNil = iota
    caseRecv
    caseSend
    caseDefault
)

// select 中每一个 case 的数据结构定义
type scase struct {
    // 数据元素
    elem unsafe.Pointer // data element
    // channel 本体
    c *hchan // chan
    kind uint16

    releasetime int64
    pc uintptr // return pc (for race detector / msan)
}

```

### 非阻塞的收发

如果一个 select 控制结构中包含一个 default 表达式，那么这个 select 并不会等待其它的 Channel 准备就绪，而是会非阻塞地读取或者写入数据：

```
func main() {
    ch := make(chan int)
    select {
    case i := <-ch:
        println(i)

    default:
        println("default")
    }
}
```

当我们运行上面的代码时其实也并不会阻塞当前的 Goroutine，而是会直接执行 default 条件中的内容并返回。

### 随机执行

另一个使用 select 遇到的情况其实就是同时有多个 case 就绪后，select 如何进行选择的问题,select 在遇到两个 <-ch 同时响应时其实会随机选择一个 case 执行其中的表达式。

## 编译期间

编译器在中间代码生成期间会根据 select 中 case 的不同对控制语句进行优化，这一过程其实都发生在 walkselectcases 函数中，我们在这里会分四种情况分别介绍优化的过程和结果：

- select 中不存在任何的 case；
- select 中只存在一个 case；
- select 中存在两个 case，其中一个 case 是 default 语句；
- 通用的 select 条件；

### 不存在任何的 case

首先介绍的其实就是最简单的情况，也就是当 select 结构中不包含任何的 case 时，编译器是如何进行处理的：

```
func walkselectcases(cases *Nodes) []*Node {
    n := cases.Len()

    if n == 0 {
        return []*Node{mkcall("block", nil, nil)}
    }
    // ...
}

```

这段代码非常简单并且容易理解，它直接将类似 select {} 的空语句，转换成对 block 函数的调用：

```
func block() {
    gopark(nil, nil, waitReasonSelectNoCases, traceEvGoStop, 1)
}
```

这其实也在告诉我们一个空的 select 语句会直接阻塞当前的 Goroutine。


### 只包含一个 case

如果当前的 select 条件只包含一个 case，那么就会就会执行如下的优化策略将原来的 select 语句改写成 if 条件语句，下面是在 select 中从 Channel 接受数据时被改写的情况：

```
select {
case v, ok <-ch:
    // ...    
}

if ch == nil {
    block()
}
v, ok := <-ch
// ...

```

我们可以看到如果在 select 中仅存在一个 case，那么当 case 中处理的 Channel 是空指针时，就会发生和没有 case 的 select 语句一样的情况，也就是直接挂起当前 Goroutine 并且永远不会被唤醒。

### 两个 case，default 语句

在下一次的优化策略执行之前，walkselectcases 函数会先将 case 中所有 Channel 都转换成指向 Channel 的地址以便于接下来的优化和通用逻辑的执行，改写之后就会进行最后一次的代码优化，触发的条件就是 — select 中包含两个 case，但是其中一个是 default，我们可以分成发送和接收两种情况介绍处理的过程。

- 发送

首先就是 Channel 的发送过程，也就是 case 中的表达式是 OSEND 类型，在这种情况下会使用 if/else 语句改写代码：

```
select {
case ch <- i:
    // ...
default:
    // ...
}

if selectnbsend(ch, i) {
    // ...
} else {
    // ...
}

```

这里最重要的函数其实就是 selectnbsend，它的主要作用就是非阻塞地向 Channel 中发送数据，我们在 Channel 一节曾经提到过发送数据的 chansend 函数包含一个 block 参数，这个参数会决定这一次的发送是不是阻塞的：

```
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
    return chansend(c, elem, false, getcallerpc())
}
```

在这里我们只需要知道当前的发送过程不是阻塞的，哪怕是没有接收方、缓冲区空间不足导致失败了也会立即返回。

- 接收

由于从 Channel 中接收数据可能会返回一个或者两个值，所以这里的情况会比发送时稍显复杂，不过改写的套路和逻辑确是差不多的：

```
select {
case v <- ch: // case v, received <- ch:
    // ...
default:
    // ...
}

if selectnbrecv(&v, ch) { // if selectnbrecv2(&v, &received, ch) {
    // ...
} else {
    // ...
}

```

返回值数量不同会导致最终使用函数的不同，两个用于非阻塞接收消息的函数 selectnbrecv 和 selectnbrecv2 其实只是对 chanrecv 返回值的处理稍有不同：

```
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
    selected, _ = chanrecv(c, elem, false)
    return
}

func selectnbrecv2(elem unsafe.Pointer, received *bool, c *hchan) (selected bool) {
    selected, *received = chanrecv(c, elem, false)
    return
}

```

- 通用阶段

在默认的情况下，select 语句会在编译阶段经过如下过程的处理：

- 将所有的 case 转换成包含 Channel 以及类型等信息的 scase 结构体；
- 调用运行时函数 selectgo 获取被选择的 scase 结构体索引，如果当前的 scase 是一个接收数据的操作，还会返回一个指示当前 case 是否是接收的布尔值；
- 通过 for 循环生成一组 if 语句，在语句中判断自己是不是被选中的 case

一个包含三个 case 的正常 select 语句其实会被展开成如下所示的逻辑，我们可以看到其中处理的三个部分：

```
selv := [3]scase{}
order := [6]uint16
for i, cas := range cases {
    c := scase{}
    c.kind = ...
    c.elem = ...
    c.c = ...
}
chosen, revcOK := selectgo(selv, order, 3)
if chosen == 0 {
    // ...
    break
}
if chosen == 1 {
    // ...
    break
}
if chosen == 2 {
    // ...
    break
}

```

展开后的 select 其实包含三部分，最开始初始化数组并转换 scase 结构体，使用 selectgo 选择执行的 case 以及最后通过 if 判断选中的情况并执行 case 中的表达式，需要注意的是这里其实也仅仅展开了 select 控制结构，select 语句执行最重要的过程其实也是选择 case 执行的过程，这是我们在下一节运行时重点介绍的。

## 运行时

selectgo 是会在运行期间运行的函数，这个函数的主要作用就是从 select 控制结构中的多个 case 中选择一个需要执行的 case，随后的多个 if 条件语句就会根据 selectgo 的返回值执行相应的语句。

### 初始化

selectgo 函数首先会进行执行必要的一些初始化操作，也就是决定处理 case 的两个顺序，其中一个是 pollOrder 另一个是 lockOrder：

```
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
    cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
    order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))

    scases := cas1[:ncases:ncases]
    pollorder := order1[:ncases:ncases]
    lockorder := order1[ncases:][:ncases:ncases]

    for i := range scases {
        cas := &scases[i]
        if cas.c == nil && cas.kind != caseDefault {
            *cas = scase{}
        }
    }

    for i := 1; i < ncases; i++ {
        j := fastrandn(uint32(i + 1))
        pollorder[i] = pollorder[j]
        pollorder[j] = uint16(i)
    }

    // sort the cases by Hchan address to get the locking order.
    // ...

    sellock(scases, lockorder)

    // ...
}

```

Channel 的轮询顺序是通过 fastrandn 随机生成的，这其实就导致了如果多个 Channel 同时『响应』，select 会随机选择其中的一个执行；而另一个 lockOrder 就是根据 Channel 的地址确定的，根据相同的顺序锁定 Channel 能够避免死锁的发生，最后调用的 sellock 就会按照之前生成的顺序锁定所有的 Channel。

### 循环

#### 非阻塞遍历

当我们为 select 语句确定了轮询和锁定的顺序并锁定了所有的 Channel 之后就会开始进入 select 的主循环，查找或者等待 Channel 准备就绪，循环中会遍历所有的 case 并找到需要被唤起的 sudog 结构体，在这段循环的代码中，我们会分四种不同的情况处理 select 中的多个 case：

- caseNil — 当前 case 不包含任何的 Channel，就直接会被跳过；
- caseRecv — 当前 case 会从 Channel 中接收数据；
	- 如果当前 Channel 的 sendq 上有等待的 Goroutine 就会直接跳到 recv 标签所在的代码段，从 Goroutine 中获取最新发送的数据；
	- 如果当前 Channel 的缓冲区不为空就会跳到 bufrecv 标签处从缓冲区中获取数据；
	- 如果当前 Channel 已经被关闭就会跳到 rclose 做一些清除的收尾工作；
- caseSend — 当前 case 会向 Channel 发送数据；
	- 如果当前 Channel 已经被关闭就会直接跳到 sclose 代码段；
	- 如果当前 Channel 的 recvq 上有等待的 Goroutine 就会跳到 send 代码段向 Channel 直接发送数据；
- caseDefault — 当前 case 表示默认情况，如果循环执行到了这种情况就表示前面的所有 case 都没有被执行，所以这里会直接解锁所有的 Channel 并退出 selectgo 函数，这时也就意味着当前 select 结构中的其他收发语句都是非阻塞的。

```
loop:
	// pass 1 - look for something already waiting
	var dfli int
	var dfl *scase
	var casi int
	var cas *scase
	var recvOK bool
	for i := 0; i < ncases; i++ {
		casi = int(pollorder[i])
		cas = &scases[casi]
		c = cas.c

		switch cas.kind {
		case caseNil:
			continue

		case caseRecv:
			sg = c.sendq.dequeue()
			if sg != nil {
				goto recv
			}
			if c.qcount > 0 {
				goto bufrecv
			}
			if c.closed != 0 {
				goto rclose
			}

		case caseSend:
			if raceenabled {
				racereadpc(c.raceaddr(), cas.pc, chansendpc)
			}
			if c.closed != 0 {
				goto sclose
			}
			sg = c.recvq.dequeue()
			if sg != nil {
				goto send
			}
			if c.qcount < c.dataqsiz {
				goto bufsend
			}

		case caseDefault:
			dfli = casi
			dfl = cas
		}
	}
```

这其实是循环执行的第一次遍历，主要作用就是寻找所有 case 中 Channel 是否有可以立刻被处理的情况，无论是在包含等待的 Goroutine 还是缓冲区中存在数据，只要满足条件就会立刻处理。

下面是各个

```
bufrecv:
	// can receive from buffer
	recvOK = true
	qp = chanbuf(c, c.recvx)
	if cas.elem != nil {
		typedmemmove(c.elemtype, cas.elem, qp)
	}
	typedmemclr(c.elemtype, qp)
	c.recvx++
	if c.recvx == c.dataqsiz {
		c.recvx = 0
	}
	c.qcount--
	selunlock(scases, lockorder)
	goto retc

bufsend:
	// can send to buffer
	typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
	c.sendx++
	if c.sendx == c.dataqsiz {
		c.sendx = 0
	}
	c.qcount++
	selunlock(scases, lockorder)
	goto retc

recv:
	// can receive from sleeping sender (sg)
	recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncrecv: cas0=", cas0, " c=", c, "\n")
	}
	recvOK = true
	goto retc

rclose:
	// read at end of closed channel
	selunlock(scases, lockorder)
	recvOK = false
	if cas.elem != nil {
		typedmemclr(c.elemtype, cas.elem)
	}
	goto retc

send:
	// can send to a sleeping receiver (sg)
	send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncsend: cas0=", cas0, " c=", c, "\n")
	}
	goto retc

sclose:
	// send on closed channel
	selunlock(scases, lockorder)
	panic(plainError("send on closed channel"))
```

#### 加入队列后阻塞

如果不能立刻找到活跃的 Channel 就会进入循环的下一个过程，按照需要将当前的 Goroutine 加入到所有 Channel 的 sendq 或者 recvq 队列中：

```
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
    // ...
    gp = getg()
    nextp = &gp.waiting
    for _, casei := range lockorder {
        casi = int(casei)
        cas = &scases[casi]
        if cas.kind == caseNil {
            continue
        }
        c = cas.c
        sg := acquireSudog()
        sg.g = gp
        sg.isSelect = true
        sg.elem = cas.elem
        sg.c = c
        *nextp = sg
        nextp = &sg.waitlink

        switch cas.kind {
        case caseRecv:
            c.recvq.enqueue(sg)

        case caseSend:
            c.sendq.enqueue(sg)
        }
    }

    gp.param = nil
    gopark(selparkcommit, nil, waitReasonSelect, traceEvGoBlockSelect, 1)

    // ...
}
```

这里创建 sudog 并入队的过程其实和 Channel 中直接进行发送和接收时的过程几乎完全相同，只是除了在入队之外，这些 sudog 结构体都会被串成链表附着在当前 Goroutine 上，在入队之后会调用 gopark 函数挂起当前的 Goroutine 等待调度器的唤醒。

![](img/waiting.png)

#### 唤醒

等到 select 对应的一些 Channel 准备好之后，当前 Goroutine 就会被调度器唤醒，这时就会继续执行 selectgo 函数中剩下的逻辑，也就是从上面 入队的 sudog 结构体中获取数据：

```
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
    // ...
    gp.selectDone = 0
    sg = (*sudog)(gp.param)
    gp.param = nil

    casi = -1
    cas = nil
    sglist = gp.waiting
    gp.waiting = nil

    for _, casei := range lockorder {
        k = &scases[casei]
        if sg == sglist {
            casi = int(casei)
            cas = k
        } else {
            if k.kind == caseSend {
                c.sendq.dequeueSudoG(sglist)
            } else {
                c.recvq.dequeueSudoG(sglist)
            }
        }
        sgnext = sglist.waitlink
        sglist.waitlink = nil
        releaseSudog(sglist)
        sglist = sgnext
    }

    c = cas.c

    if cas.kind == caseRecv {
        recvOK = true
    }

    selunlock(scases, lockorder)
    goto retc
    // ...
    
    
retc:
	return casi, recvOK

}

```

在第三次根据 lockOrder 遍历全部 case 的过程中，我们会先获取 Goroutine 接收到的参数 param，这个参数其实就是被唤醒的 sudog 结构，我们会依次对比所有 case 对应的 sudog 结构找到被唤醒的 case 并释放其他未被使用的 sudog 结构。

由于当前的 select 结构已经挑选了其中的一个 case 进行执行，那么剩下 case 中没有被用到的 sudog 其实就会直接忽略并且释放掉了，为了不影响 Channel 的正常使用，我们还是需要将这些废弃的 sudog 从 Channel 中出队；而除此之外的发生事件导致我们被唤醒的 sudog 结构已经在 Channel 进行收发时就已经出队了，不需要我们再次处理。

注意 gp.waiting 就是我们在阻塞睡眠之前，sodug 的链表，存储着所有的 sodug。