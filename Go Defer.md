# Go Defer

## 用法

### 常见使用

首先要介绍的就是使用 defer 最常见的场景，也就是在 defer 关键字中完成一些收尾的工作，例如在 defer 中回滚一个数据库的事务：

```
func createPost(db *gorm.DB) error {
    tx := db.Begin()
    defer tx.Rollback()

    if err := tx.Create(&Post{Author: "Draveness"}).Error; err != nil {
        return err
    }

    return tx.Commit().Error
}
```

在使用数据库事务时，我们其实可以使用如上所示的代码在创建事务之后就立刻调用 Rollback 保证事务一定会回滚，哪怕事务真的执行成功了，那么在调用 tx.Commit() 之后再执行 tx.Rollback() 其实也不会影响已经提交的事务。

### 作用域

当我们在一个 for 循环中使用 defer 时也会在退出函数之前执行其中的代码，下面的代码总共调用了五次 defer 关键字：

```
func main() {
    for i := 0; i < 5; i++ {
        defer fmt.Println(i)
    }
}

$ go run main.go
4
3
2
1
0

```

### 传值

Go 语言中所有的函数调用其实都是值传递的，defer 虽然是一个关键字，但是也继承了这个特性，假设我们有以下的代码，在运行这段代码时会打印出 0：

```
type Test struct {
    value int
}

func (t Test) print() {
    println(t.value)
}

func main() {
    test := Test{}
    defer test.print()
    test.value += 1
}

$ go run main.go
0

```

这其实表明当 defer 调用时其实会对函数中引用的外部参数进行拷贝，所以 test.value += 1 操作并没有修改被 defer 捕获的 test 结构体，不过如果我们修改 print 函数签名的话，其实结果就会稍有不同：

```
type Test struct {
    value int
}

func (t *Test) print() {
    println(t.value)
}

func main() {
    test := Test{}
    defer test.print()
    test.value += 1
}

$ go run main.go
1

```

### defer 调用时机

```
func f() (result int) { 
    defer func() {
	    result++ 
	}()
	
	return 0 
}

func f() (r int) { 
    t := 5
    defer func() { 
       t=t+5
    }()

    return t 
}

func f() (r int) {
    defer func(r int) {
        r=r+5
    }(r)
    
    return 1
}
```

使用defer时，用一个简单的转换规则改写一下，就不会迷糊了。改写规则是将return语句拆成两句写，return xxx会被改 写成:

```
返回值 = xxx 
调用defer函数 
空的return

```

先看例1，它可以改写成这样:

```
func f() (result int) {
    result = 0 //return语句不是一条原子调用，return xxx其实是赋值+ret指令 
    func() { //defer被插入到return之前执行，也就是赋返回值和ret指令之间
        result++ 
    }()
 
    return 
}
```

所以这个返回值是1。

再看例2，它可以改写成这样:

```
func f() (r int) { 
    t := 5
    r = t
    func() { 
       t=t+5
    }()

    return
}

```

所以这个的结果是5。 

最后看例3，它改写后变成:

```
func f() (r int) {
    r = 1
    func(r int) {
        r=r+5
    }(r)
    
    return 1
}
```

所以这个例子的结果是1。

## 数据结构

```
type _defer struct {
    siz     int32
    started bool
    sp      uintptr
    pc      uintptr
    fn      *funcval
    _panic  *_panic
    link    *_defer
}
```

在 _defer 结构中的 sp 和 pc 分别指向了栈指针和调用方的程序计数器，fn 存储的就是向 defer 关键字中传入的函数了。

## 原理

在 Go 语言的编译期间，编译器不仅将 defer 转换成了 deferproc 的函数调用，还在所有调用 defer 的函数结尾（返回之前）插入了 deferreturn。

每一个 defer 关键字都会被转换成 deferproc，在这个函数中我们会为 defer 创建一个新的 _defer 结构体并设置它的 fn、pc 和 sp 参数，除此之外我们会将 defer 相关的函数都拷贝到紧挨着结构体的内存空间中：

```
func deferproc(siz int32, fn *funcval) {
    sp := getcallersp()
    argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
    callerpc := getcallerpc()

    d := newdefer(siz)
    if d._panic != nil {
        throw("deferproc: d.panic != nil after newdefer")
    }
    d.fn = fn
    d.pc = callerpc
    d.sp = sp
    switch siz {
    case 0:
    case sys.PtrSize:
        *(*uintptr)(deferArgs(d)) = *(*uintptr)(unsafe.Pointer(argp))
    default:
        memmove(deferArgs(d), unsafe.Pointer(argp), uintptr(siz))
    }

    return0()
}

```

上述函数最终会使用 return0 返回，这个函数的主要作用就是避免在 deferproc 函数中使用 return 返回时又会导致 deferreturn 函数的执行，这也是唯一一个不会触发 defer 的函数了。

deferproc 中调用的 newdefer 主要作用就是初始化或者取出一个新的 _defer 结构体：

```
func newdefer(siz int32) *_defer {
    var d *_defer
    sc := deferclass(uintptr(siz))
    gp := getg()
    if sc < uintptr(len(p{}.deferpool)) {
        pp := gp.m.p.ptr()
        if len(pp.deferpool[sc]) == 0 && sched.deferpool[sc] != nil {
            lock(&sched.deferlock)
            for len(pp.deferpool[sc]) < cap(pp.deferpool[sc])/2 && sched.deferpool[sc] != nil {
                d := sched.deferpool[sc]
                sched.deferpool[sc] = d.link
                d.link = nil
                pp.deferpool[sc] = append(pp.deferpool[sc], d)
            }
            unlock(&sched.deferlock)
        }
        if n := len(pp.deferpool[sc]); n > 0 {
            d = pp.deferpool[sc][n-1]
            pp.deferpool[sc][n-1] = nil
            pp.deferpool[sc] = pp.deferpool[sc][:n-1]
        }
    }
    if d == nil {
        total := roundupsize(totaldefersize(uintptr(siz)))
        d = (*_defer)(mallocgc(total, deferType, true))
    }
    d.siz = siz
    d.link = gp._defer
    gp._defer = d
    return d
}

```

从最后的一小段代码我们可以看出，所有的 `_defer` 结构体都会关联到所在的 Goroutine 上并且每创建一个新的 `_defer` 都会追加到协程持有的 `_defer` 链表的最前面。

deferreturn 其实会从 Goroutine 的链表中取出链表最前面的 _defer 结构体并调用 jmpdefer 函数并传入需要执行的函数和参数：

```
func deferreturn(arg0 uintptr) {
    gp := getg()
    d := gp._defer
    if d == nil {
        return
    }
    sp := getcallersp()

    switch d.siz {
    case 0:
    case sys.PtrSize:
        *(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
    default:
        memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz))
    }
    fn := d.fn
    d.fn = nil
    gp._defer = d.link
    freedefer(d)
    jmpdefer(fn, uintptr(unsafe.Pointer(&arg0)))
}

```

jmpdefer 其实是一个用汇编语言实现的函数，在不同的处理器架构上的实现稍有不同，但是具体的执行逻辑都差不太多，它们的工作其实就是跳转到并执行 defer 所在的代码段并在执行结束之后跳转回 defereturn 函数。

```
TEXT runtime·jmpdefer(SB), NOSPLIT, $0-8
    MOVL    fv+0(FP), DX    // fn
    MOVL    argp+4(FP), BX    // caller sp
    LEAL    -4(BX), SP    // caller sp after CALL
#ifdef GOBUILDMODE_shared
    SUBL    $16, (SP)    // return to CALL again
#else
    SUBL    $5, (SP)    // return to CALL again
#endif
    MOVL    0(DX), BX
    JMP    BX    // but first run the deferred function

```

