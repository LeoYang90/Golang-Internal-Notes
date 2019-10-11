# Go Slice

[TOC]

## 数组

数组是由相同类型元素的集合组成的数据结构，计算机会为数组分配一块连续的内存来保存数组中的元素，我们可以利用数组中元素的索引快速访问元素对应的存储地址，常见的数组大多都是一维的线性数组。

数组作为一种数据类型，一般情况下由两部分组成，其中一部分表示了数组中存储的元素类型，另一部分表示数组最大能够存储的元素个数。

Go 语言中数组的大小在初始化之后就无法改变，数组存储元素的类型相同，但是大小不同的数组类型在 Go 语言看来也是完全不同的，只有两个条件都相同才是同一个类型。

```
func NewArray(elem *Type, bound int64) *Type {
    if bound < 0 {
        Fatalf("NewArray: invalid bound %v", bound)
    }
    t := New(TARRAY)
    t.Extra = &Array{Elem: elem, Bound: bound}
    t.SetNotInHeap(elem.NotInHeap())
    return t
}

```

编译期间的数组类型 Array 就包含两个结构，一个是元素类型 Elem，另一个是数组的大小上限 Bound，这两个字段构成了数组类型，而当前数组是否应该在堆栈中初始化也在编译期间就确定了。

### 创建

Go 语言中的数组有两种不同的创建方式，一种是我们显式指定数组的大小，另一种是编译器通过源代码自行推断数组的大小：

```
arr1 := [3]int{1, 2, 3}
arr2 := [...]int{1, 2, 3}

```

后一种声明方式在编译期间就会被『转换』成为前一种，下面我们先来介绍数组大小的编译期推导过程。

这两种不同的方式会导致编译器做出不同的处理，如果我们使用第一种方式 [10]T，那么变量的类型在编译进行到 类型检查 阶段就会被推断出来，在这时编译器会使用 NewArray 创建包含数组大小的 Array 类型，而如果使用 [...]T 的方式，虽然在这一步也会创建一个 Array 类型 Array{Elem: elem, Bound: -1}，但是其中的数组大小上限会是 -1 的结构，这意味着还需要后面的 typecheckcomplit 函数推导该数组的大小：

```
func typecheckcomplit(n *Node) (res *Node) {
    // ...

    switch t.Etype {
    case TARRAY, TSLICE:
        var length, i int64
        nl := n.List.Slice()
        for i2, l := range nl {
            i++
            if i > length {
                length = i
            }
        }

        if t.IsDDDArray() {
            t.SetNumElem(length)
        }
    }
}

func (t *Type) SetNumElem(n int64) {
	t.wantEtype(TARRAY)
	at := t.Extra.(*Array)
	if at.Bound >= 0 {
		Fatalf("SetNumElem array %v already has bound %d", t, at.Bound)
	}
	at.Bound = n
}
```

这个删减后的 typecheckcomplit 函数通过遍历元素来推导当前数组的长度，我们能看出 [...]T 类型的声明不是在运行时被推导的，它会在类型检查期间就被推断出正确的数组大小。

对于一个由字面量组成的数组，根据数组元素数量的不同，编译器会在负责初始化字面量的 anylit 函数中做两种不同的优化：如果数组中元素的个数小于或者等于 4 个，那么所有的变量会直接在栈上初始化，如果数组元素大于 4 个，变量就会在静态存储区初始化然后拷贝到栈上。

### 访问和赋值

无论是在栈上还是静态存储区，数组在内存中其实就是一连串的内存空间，表示数组的方法就是一个指向数组开头的指针，这一片内存空间不知道自己存储的是什么变量。

数组访问越界的判断也都是在编译期间由静态类型检查完成的。

无论是编译器还是字符串，它们的越界错误都会在编译期间发现，但是数组访问操作 OINDEX 会在编译期间被转换成两个 SSA 指令：

```
PtrIndex <t> ptr idx
Load <t> ptr mem

```

编译器会先获取数组的内存地址和访问的下标，然后利用 PtrIndex 计算出目标元素的地址，再使用 Load 操作将指针中的元素加载到内存中。

数组的赋值和更新操作 a[i] = 2 也会生成 SSA 期间就计算出数组当前元素的内存地址，然后修改当前内存地址的内容，其实会被转换成如下所示的 SSA 操作：

```
LocalAddr {sym} base _
PtrIndex <t> ptr idx
Store {t} ptr val mem

```

在这个过程中会确实能够目标数组的地址，再通过 PtrIndex 获取目标元素的地址，最后将数据存入地址中，从这里我们可以看出无论是数组的寻址还是赋值都是在编译阶段完成的，没有运行时的参与。

## 切片

在 Golang 中，切片类型的声明与数组有一些相似，由于切片是『动态的』，它的长度并不固定，所以声明类型时只需要指定切片中的元素类型.

切片在编译期间的类型应该只会包含切片中的元素类型，NewSlice 就是编译期间用于创建 Slice 类型的函数：

```
func NewSlice(elem *Type) *Type {
    if t := elem.Cache.slice; t != nil {
        if t.Elem() != elem {
            Fatalf("elem mismatch")
        }
        return t
    }

    t := New(TSLICE)
    t.Extra = Slice{Elem: elem}
    elem.Cache.slice = t
    return t
}

```

我们可以看到上述方法返回的类型 TSLICE 的 Extra 字段是一个只包含切片内元素类型的 Slice{Elem: elem} 结构，也就是说切片内元素的类型是在编译期间确定的。

### 结构

编译期间的切片其实就是一个 Slice 类型，但是在运行时切片其实由如下的 SliceHeader 结构体表示，其中 Data 字段是一个指向数组的指针，Len 表示当前切片的长度，而 Cap 表示当前切片的容量，也就是 Data 数组的大小：

```
type SliceHeader struct {
    Data uintptr
    Len  int
    Cap  int
}

```

Data 作为一个指针指向的数组其实就是一片连续的内存空间，这片内存空间可以用于存储切片中保存的全部元素，数组其实就是一片连续的内存空间，数组中的元素只是逻辑上的概念，底层存储其实都是连续的，所以我们可以将切片理解成一片连续的内存空间加上长度与容量标识。

切片与数组不同，获取数组大小、对数组中的元素的访问和更新在编译期间就已经被转换成了数字和对内存的直接操作，但是切片是运行时才会确定的结构，所有的操作还需要依赖 Go 语言的运行时来完成，我们接下来就会介绍切片的一些常见操作的实现原理。

### 初始化

首先需要介绍的就是切片的创建过程，Go 语言中的切片总共有两种初始化的方式，一种是使用字面量初始化新的切片，另一种是使用关键字 make 创建切片：

```
slice := []int{1, 2, 3}
slice := make([]int, 10)

```

对于字面量 slice，会在 SSA 代码生成阶段被转换成 OpSliceMake 操作

对于关键字 make，如果当前的切片不会发生逃逸并且切片非常小的时候，仍然会被转为 OpSliceMake 操作。否则会调用：

```
makeslice(type, len, cap)

```

当切片的容量和大小不能使用 int 来表示时，就会实现 makeslice64 处理容量和大小更大的切片，无论是 makeslice 还是 makeslice64，这两个方法都是在结构逃逸到堆上初始化时才需要调用的。

接下来，我们回到用于创建切片的 makeslice 函数，这个函数的实现其实非常简单：

```
func makeslice(et *_type, len, cap int) unsafe.Pointer {
    mem, overflow := math.MulUintptr(et.size, uintptr(cap))
    if overflow || mem > maxAlloc || len < 0 || len > cap {
        mem, overflow := math.MulUintptr(et.size, uintptr(len))
        if overflow || mem > maxAlloc || len < 0 {
            panicmakeslicelen()
        }
        panicmakeslicecap()
    }

    return mallocgc(mem, et, true)
}

```

上述代码的主要工作就是用切片中元素大小和切片容量相乘计算出切片占用的内存空间，如果内存空间的大小发生了溢出、申请的内存大于最大可分配的内存、传入的长度小于 0 或者长度大于容量，那么就会直接报错，当然大多数的错误都会在编译期间就检查出来，mallocgc 就是用于申请内存的函数，这个函数的实现还是比较复杂，如果遇到了比较小的对象会直接初始化在 Golang 调度器里面的 P 结构中，而大于 32KB 的一些对象会在堆上初始化。

### 访问

对切片常见的操作就是获取它的长度或者容量，这两个不同的函数 len 和 cap 其实被 Go 语言的编译器看成是两种特殊的操作 OLEN 和 OCAP，它们会在 SSA 生成阶段 被转换成 OpSliceLen 和 OpSliceCap 操作.

除了获取切片的长度和容量之外，访问切片中元素使用的 OINDEX 操作也都在 SSA 中间代码生成期间就转换成对地址的获取操作.

### 追加

向切片中追加元素应该是最常见的切片操作，在 Go 语言中我们会使用 append 关键字向切片中追加元素，追加元素会根据是否 inplace 在中间代码生成阶段转换成以下的两种不同流程，如果 append 之后的切片不需要赋值回原有的变量，也就是如 append(slice, 1, 2, 3) 所示的表达式会被转换成如下的过程：

```
ptr, len, cap := slice
newlen := len + 3
if newlen > cap {
    ptr, len, cap = growslice(slice, newlen)
    newlen = len + 3
}
*(ptr+len) = 1
*(ptr+len+1) = 2
*(ptr+len+2) = 3
return makeslice(ptr, newlen, cap)

```

我们会先对切片结构体进行解构获取它的数组指针、大小和容量，如果新的切片大小大于容量，那么就会使用 growslice 对切片进行扩容并将新的元素依次加入切片并创建新的切片，但是 slice = apennd(slice, 1, 2, 3) 这种 inplace 的表达式就只会改变原来的 slice 变量：

```
a := &slice
ptr, len, cap := slice
newlen := len + 3
if uint(newlen) > uint(cap) {
   newptr, len, newcap = growslice(slice, newlen)
   vardef(a)
   *a.cap = newcap
   *a.ptr = newptr
}
newlen = len + 3
*a.len = newlen
*(ptr+len) = 1
*(ptr+len+1) = 2
*(ptr+len+2) = 3

```

上述两段代码的逻辑其实差不多，最大的区别在于最后的结果是不是赋值会原有的变量，不过从 inplace 的代码可以看出 Go 语言对类似的过程进行了优化，所以我们并不需要担心 append 会在数组容量足够时导致发生切片的复制。

到这里我们已经了解了在切片容量足够时如何向切片中追加元素，但是如果切片的容量不足时就会调用 growslice 为切片扩容：

```
func growslice(et *_type, old slice, cap int) slice {
    newcap := old.cap
    doublecap := newcap + newcap
    if cap > doublecap {
        newcap = cap
    } else {
        if old.len < 1024 {
            newcap = doublecap
        } else {
            for 0 < newcap && newcap < cap {
                newcap += newcap / 4
            }
            if newcap <= 0 {
                newcap = cap
            }
        }
    }

```

扩容其实就是需要为切片分配一块新的内存空间，分配内存空间之前需要先确定新的切片容量，Go 语言根据切片的当前容量选择不同的策略进行扩容：

- 如果期望容量大于当前容量的两倍就会使用期望容量；
- 如果当前切片容量小于 1024 就会将容量翻倍；
- 如果当前切片容量大于 1024 就会每次增加 25% 的容量，直到新容量大于期望容量；

确定了切片的容量之后，我们就可以开始计算切片中新数组的内存占用了，计算的方法就是将目标容量和元素大小相乘：

```
    var overflow bool
    var lenmem, newlenmem, capmem uintptr
    switch {
    // ...
    default:
        lenmem = uintptr(old.len) * et.size
        newlenmem = uintptr(cap) * et.size
        capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
        capmem = roundupsize(capmem)
        newcap = int(capmem / et.size)
    }

    var p unsafe.Pointer
    if et.kind&kindNoPointers != 0 {
        p = mallocgc(capmem, nil, false)
        memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
    } else {
        p = mallocgc(capmem, et, true)
        if writeBarrier.enabled {
            bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem)
        }
    }
    memmove(p, old.array, lenmem)

    return slice{p, old.len, newcap}
}

```

如果当前切片中元素不是指针类型，那么就会调用 memclrNoHeapPointers 函数将超出当前长度的位置置空并在最后使用 memmove 将原数组内存中的内容拷贝到新申请的内存中， 不过无论是 memclrNoHeapPointers 还是 memmove 函数都使用目标机器上的汇编指令进行实现.

### 拷贝

```
func slicecopy(to, fm slice, width uintptr) int {
    if fm.len == 0 || to.len == 0 {
        return 0
    }

    n := fm.len
    if to.len < n {
        n = to.len
    }

    if width == 0 {
        return n
    }

    // ...

    size := uintptr(n) * width
    if size == 1 {
        *(*byte)(to.array) = *(*byte)(fm.array)
    } else {
        memmove(to.array, fm.array, size)
    }
    return n
}

```

上述函数的实现非常直接，它将切片中的全部元素通过 memmove 或者数组指针的方式将整块内存中的内容拷贝到目标的内存区域.