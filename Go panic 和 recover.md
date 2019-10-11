# Go panic 和 recover

## 概述

在具体介绍和分析 Go 语言中的 panic 和 recover 的实现原理之前，我们首先需要对它们有一些基本的了解；panic 和 recover 两个关键字其实都是 Go 语言中的内置函数，panic 能够改变程序的控制流，当一个函数调用执行 panic 时，它会立刻停止执行函数中其他的代码，而是会运行其中的 defer 函数，执行成功后会返回到调用方。

对于上层调用方来说，调用导致 panic 的函数其实与直接调用 panic 类似，所以也会执行所有的 defer 函数并返回到它的调用方，这个过程会一直进行直到当前 Goroutine 的调用栈中不包含任何的函数，这时整个程序才会崩溃，这个『恐慌过程』不仅会被显式的调用触发，还会由于运行期间发生错误而触发。

然而 panic 导致的『恐慌』状态其实可以被 defer 中的 recover 中止，recover 是一个只在 defer 中能够发挥作用的函数，在正常的控制流程中，调用 recover 会直接返回 nil 并且没有任何的作用，但是如果当前的 Goroutine 发生了『恐慌』，recover 其实就能够捕获到 panic 抛出的错误并阻止『恐慌』的继续传播。

```
func main() {
    defer println("in main")
    go func() {
        defer println("in goroutine")
        panic("")
    }()
    
    println("in main...")

    time.Sleep(1 * time.Second)
}

// in main...
// in goroutine
// panic:
// ...

```

当我们运行这段代码时，其实会发现 main 函数中的 defer 语句并没有执行，执行的其实只有 Goroutine 中的 defer，这其实就印证了 Go 语言在发生 panic 时只会执行当前协程中的 defer 函数，这一点从 上一节 的源代码中也有所体现。

另一个例子就不止涉及 panic 和 defer 关键字了，我们可以看一下 recover 是如何让当前函数重新『走向正轨』的：

```
func main() {
	defer println("in main")
	go func() {
		defer println("in goroutine")
		defer func() {
			if err := recover(); err != nil {
				fmt.Println(err)
			}
		}()
		panic("G panic")
	}()

	println("in main...")

	time.Sleep(1 * time.Second)
}

in main...
G panic
in goroutine
in main

```

从这个例子中我们可以看到，recover 函数其实只是阻止了当前程序的崩溃，但是当前控制流中的其他 defer 函数还会正常执行。

## 实现原理

### 数据结构

panic 在 Golang 中其实是由一个数据结构表示的，每当我们调用一次 panic 函数都会创建一个如下所示的数据结构存储相关的信息：

```
type _panic struct {
    argp      unsafe.Pointer
    arg       interface{}
    link      *_panic
    recovered bool
    aborted   bool
}
```

- argp 是指向 defer 调用时参数的指针；
- arg 是调用 panic 时传入的参数；
- link 指向了更早调用的 _panic 结构；
- recovered 表示当前 _panic 是否被 recover 恢复；
- aborted 表示当前的 panic 是否被强行终止；

从数据结构中的 link 字段我们就可以推测出以下的结论 — panic 函数可以被连续多次调用，它们之间通过 link 的关联形成一个链表。

### 崩溃

首先了解一下没有被 recover 的 panic 函数是如何终止整个程序的，我们来看一下 gopanic 函数的实现

```
func gopanic(e interface{}) {
    gp := getg()
    // ...
    var p _panic
    p.arg = e
    p.link = gp._panic
    gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

    for {
        d := gp._defer
        if d == nil {
            break
        }

        d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

        p.argp = unsafe.Pointer(getargp(0))
        reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))
        p.argp = nil

        d._panic = nil
        d.fn = nil
        gp._defer = d.link

        pc := d.pc
        sp := unsafe.Pointer(d.sp)
        freedefer(d)
        if p.recovered {
            // ...
        }
    }

    fatalpanic(gp._panic)
    *(*int)(nil) = 0
}

```

我们暂时省略了 recover 相关的代码，省略后的 gopanic 函数执行过程包含以下几个步骤：

- 获取当前 panic 调用所在的 Goroutine 协程；
- 创建并初始化一个 _panic 结构体；
- 从当前 Goroutine 中的链表获取一个 _defer 结构体；
- 如果当前 _defer 存在，调用 reflectcall 执行 _defer 中的代码；
- 将下一位的 _defer 结构设置到 Goroutine 上并回到 3；
- 调用 fatalpanic 中止整个程序；

fatalpanic 函数在中止整个程序之前可能就会通过 printpanics 打印出全部的 panic 消息以及调用时传入的参数：

```
func fatalpanic(msgs *_panic) {
    pc := getcallerpc()
    sp := getcallersp()
    gp := getg()
    var docrash bool
    systemstack(func() {
        if startpanic_m() && msgs != nil {
            atomic.Xadd(&runningPanicDefers, -1)

            printpanics(msgs)
        }
        docrash = dopanic_m(gp, pc, sp)
    })

    if docrash {
        crash()
    }

    systemstack(func() {
        exit(2)
    })

    *(*int)(nil) = 0 // not reached
}

```

在 fatalpanic 函数的最后会通过 exit 退出当前程序并返回错误码 2，不同的操作系统其实对 exit 函数有着不同的实现，其实最终都执行了 exit 系统调用来退出程序。

### 恢复

到了这里我们已经掌握了 panic 退出程序的过程，但是一个 panic 的程序也可能会被 defer 中的关键字 recover 恢复，在这时我们就回到 recover 关键字对应函数 gorecover 的实现了：

```
func gorecover(argp uintptr) interface{} {
    p := gp._panic
    if p != nil && !p.recovered && argp == uintptr(p.argp) {
        p.recovered = true
        return p.arg
    }
    return nil
}

```

这个函数的实现其实非常简单，它其实就是会修改 panic 结构体的 recovered 字段，当前函数的调用其实都发生在 gopanic 期间，我们重新回顾一下这段方法的实现：

```
func gopanic(e interface{}) {
    // ...

    for {
        // reflectcall

        pc := d.pc
        sp := unsafe.Pointer(d.sp)

        // ...
        if p.recovered {
            gp._panic = p.link
            for gp._panic != nil && gp._panic.aborted {
                gp._panic = gp._panic.link
            }
            if gp._panic == nil {
                gp.sig = 0
            }
            gp.sigcode0 = uintptr(sp)
            gp.sigcode1 = pc
            mcall(recovery)
            throw("recovery failed")
        }
    }

    fatalpanic(gp._panic)
    *(*int)(nil) = 0
}

```

上述这段代码其实从 _defer 结构体中取出了程序计数器 pc 和栈指针 sp 并调用 recovery 方法进行调度，调度之前会准备好 sp、pc 以及函数的返回值：

```
func recovery(gp *g) {
    sp := gp.sigcode0
    pc := gp.sigcode1

    gp.sched.sp = sp
    gp.sched.pc = pc
    gp.sched.lr = 0
    gp.sched.ret = 1
    gogo(&gp.sched)
}

```

这里的调度其实会将 deferproc 函数的返回值设置成 1，在这时编译器生成的代码就会帮助我们直接跳转到调用方函数 return 之前并进入 deferreturn 的执行过程.

跳转到 deferreturn 函数之后，程序其实就从 panic 的过程中跳出来恢复了正常的执行逻辑，而 gorecover 函数也从 _panic 结构体中取出了调用 panic 时传入的 arg 参数。