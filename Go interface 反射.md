# Go interface 反射
[TOC]

## 反射用法

### 反射定律

#### 从接口值到反射对象的反射

反射是一种检查存储在接口变量中的（类型，值）对的机制。作为一个开始，我们需要知道reflect包中的两个类型：Type和Value。这两种类型给了我们访问一个接口变量中所包含的内容的途径，另外两个简单的函数reflect.Typeof和reflect.Valueof可以检索一个接口值的reflect.Type和reflect.Value部分。

```
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var x float64 = 3.4
    fmt.Println("type:", reflect.TypeOf(x))
}

```

reflect.Typeof 签名里就包含了一个空接口：

```
func TypeOf(i interface{}) Type

```

当我们调用reflect.Typeof(x)的时候，x首先被保存到一个空接口中，这个空接口然后被作为参数传递。reflect.Typeof 会把这个空接口拆包（unpack）恢复出类型信息。

当然，reflect.Valueof可以把值恢复出来

```
var x float64 = 3.4
fmt.Println("value:", reflect.ValueOf(x))//Valueof方法会返回一个Value类型的对象

```

reflect.Type和reflect.Value这两种类型都提供了大量的方法让我们可以检查和操作这两种类型。一个重要的例子是:

- Value类型有一个 Type 方法可以返回reflect.Value类型的Type（这个方法返回的是值的静态类型即static type，也就是说如果定义了type MyInt int64，那么这个函数返回的是MyInt类型而不是int64
- Type 和 Value 都有一个Kind方法可以返回一个常量用于指示一个项到底是以什么形式（也就是底层类型即underlying type，继续前面括号里提到的，Kind返回的是int64而不是MyInt）存储的，这些常量包括：Unit, Float64, Slice等等。而且，有关Value类型的带有名字诸如Int和Float的方法可让让我们获取存在里面的值（比如int64和float64)：
    
    ```
    var x float64 = 3.4
	v := reflect.ValueOf(x)
	fmt.Println("type:", v.Type())
	fmt.Println("kind is float64:", v.Kind() == reflect.Float64)
	fmt.Println("value:", v.Float())
    
    type: float64
	kind is float64: true
	value: 3.4
    ```
反射库里有俩性质值得单独拿出来说说。第一个性质是，为了保持API简单，Value的”setter”和“getter”类型的方法操作的是可以包含某个值的最大类型：比如，所有的有符号整型，只有针对int64类型的方法，因为它是所有的有符号整型中最大的一个类型。也就是说，Value的Int方法返回的是一个int64，同时SetInt的参数类型采用的是一个int64；所以，必要时要转换成实际类型：

```
var x uint8 = 'x'
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())                            // uint8.
fmt.Println("kind is uint8: ", v.Kind() == reflect.Uint8) // true.
x = uint8(v.Uint())// v.Uint returns a uint64.看到啦嘛？这个地方必须进行强制类型转换！

```

第二个性质是，反射对象（reflection object）的Kind描述的是底层类型（underlying type）

#### 从反射队形到接口值的反射

就像物理学上的反射，Go中到反射可以生成它的逆。

给定一个reflect.Value，我们能用Interface方法把它恢复成一个接口值；效果上就是这个Interface方法把类型和值的信息打包成一个接口表示并且返回结果：

```
func (v Value) Interface() interface{}

```

```
y := v.Interface().(float64) // y will have type float64.
fmt.Println(y)

```
我们甚至可以做得更好一些，fmt.Println等方法的参数是一个空接口类型的值，所以我们可以让fmt包自己在内部完成我们在上面代码中做的工作。因此，为了正确打印一个reflect.Value，我们只需把Interface方法的返回值直接传递给这个格式化输出例程：

```
fmt.Println(v.Interface())

```

```
fmt.Printf("value is %7.1e\n", v.Interface())

3.4e+00
```
还有就是，我们不需要对v.Interface方法的结果调用类型断言（type-assert)为float64；空接口类型值内部包含有具体值的类型信息，并且Printf方法会把它恢复出来。

简要的说，Interface方法是Valueof函数的逆，除了它的返回值的类型总是interface{}静态类型。

#### 为了修改一个反射对象，值必须是settable的

下面是一些不能正常运行的代码，但是很值得研究：

```
var x float64 = 3.4
v := reflect.ValueOf(x)
v.SetFloat(7.1) // Error: will panic.

```
问题不是出在值7.1不是可以寻址的，而是出在v不是settable的。Settability是Value的一条性质，而且，不是所有的Value都具备这条性质。

Value的CanSet方法用与测试一个Value的settablity；在我们的例子中，

```
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("settability of v:", v.CanSet())

settability of v: false
```
如果对一个non-settable的Value调用Set方法会出现错误。但是，settability到底是什么呢？

settability有点像addressability，但是更加严格。

settability是一个性质，描述的是一个反射对象能够修改创造它的那个实际存储的值的能力。settability由反射对象是否保存原始项（original item）而决定。

```
var x float64 = 3.4
v := reflect.ValueOf(x)

```

我们传递了x的一个副本给reflect.Valueof函数，所以作为reflect.Valueof参数被创造出来的接口值只是x的一个副本，而不是x本身。

因为，如果下面这条语句

```
v.SetFloat(7.1)

```

执行成功（当然不可能执行成功啦，假设而已），它不会更新x，即使v看起来像是从x创造而来，所以它更新的只是存储在反射值内部的x的一个副本，而x本身不受丝毫影响，所以如果真这样的话，将会非常那令人困惑，而且一点用都没有！所以，这么干是非法的，而settability就是用来阻止这种哦给你非法状况出现的。

如果我们想通过反射来修改x，我们必须把我们想要修改的值的指针传给一个反射库。

首先，我们像平常一样初始化x，然后创造一个指向它的反射值，叫做p.

```
var x float64 = 3.4
p := reflect.ValueOf(&x) // Note: take the address of x.注意这里哦！我们把x地址传进去了！
fmt.Println("type of p:", p.Type())
fmt.Println("settability of p:", p.CanSet())

type of p: *float64
settability of p: false
```

反射对象p不是settable的，但是我们想要设置的不是p，而是（效果上来说）*p。为了得到p指向的东西，我们调用Value的Elem方法，这样就能迂回绕过指针，同时把结果保存在叫v的Value中：

```
v := p.Elem()
fmt.Println("settability of v:", v.CanSet())

settability of v: true

```

现在v就是一个settable的反射对象了,并且因为v表示x，我们最终能够通过v.SetFloat方法来修改x的值：

```
v.SetFloat(7.1)
fmt.Println(v.Interface())
fmt.Println(x)

```

输出正是我们所期待的，反射理解起来有点困难，但是它确实正在做编程语言要做的，尽管是通过掩盖了所发生的一切的反射Types和Vlues来实现的。这样好了，你就直接记住反射Values为了修改它们所表示的东西必须要有这些东西的地址。

### type 的方法集

来源 ：[Golang学习 - reflect 包](https://www.cnblogs.com/golove/p/5909541.html)

```
type Type interface {
	// Methods applicable to all types.

	// 获取 t 类型的值在分配内存时的字节对齐值。
	Align() int

	// 获取 t 类型的值作为结构体字段时的字节对齐值。
	FieldAlign() int

	// 根据索引获取 t 类型的方法，如果方法不存在，则 panic。
   // 如果 t 是一个实际的类型，则返回值的 Type 和 Func 字段会列出接收者。
   // 如果 t 只是一个接口，则返回值的 Type 不列出接收者，Func 为空值。
	Method(int) Method

	// 根据名称获取 t 类型的方法。
	MethodByName(string) (Method, bool)

	// 获取 t 类型的方法数量。
	NumMethod() int

	// 获取 t 类型在其包中定义的名称，未命名类型则返回空字符串。
	Name() string

	// 获取 t 类型所在包的名称，未命名类型则返回空字符串。
	PkgPath() string

	// 获取 t 类型的值在分配内存时的大小，功能和 unsafe.SizeOf 一样。
	Size() uintptr

	// 获取 t 类型的字符串描述，不要通过 String 来判断两种类型是否一致。
	String() string

	// 获取 t 类型的类别。
	Kind() Kind

	// 判断 t 类型是否实现了 u 接口。
	Implements(u Type) bool

	// 判断 t 类型的值可否赋值给 u 类型。
	AssignableTo(u Type) bool

	// 判断 t 类型的值可否转换为 u 类型。
	ConvertibleTo(u Type) bool

	// 判断 t 类型的值可否进行比较操作
	Comparable() bool

	// Methods applicable only to some types, depending on Kind.
	// 特定类型的函数：
	//
	//	Int*, Uint*, Float*, Complex*: Bits
	//	Array: Elem, Len
	//	Chan: ChanDir, Elem
	//	Func: In, NumIn, Out, NumOut, IsVariadic.
	//	Map: Key, Elem
	//	Ptr: Elem
	//	Slice: Elem
	//	Struct: Field, FieldByIndex, FieldByName, FieldByNameFunc, NumField

	// 获取数值类型的位宽，t 必须是整型、浮点型、复数型
	Bits() int

	// 获取通道的方向
	ChanDir() ChanDir

	// For concreteness, if t represents func(x int, y ... float64), then
	//
	//	t.NumIn() == 2
	//	t.In(0) is the reflect.Type for "int"
	//	t.In(1) is the reflect.Type for "[]float64"
	//	t.IsVariadic() == true

	// 判断函数是否具有可变参数。
    // 如果有可变参数，则 t.In(t.NumIn()-1) 将返回一个切片。
	IsVariadic() bool

	// 数组、切片、映射、通道、指针、接口
    // 获取元素类型、获取指针所指对象类型，获取接口的动态类型
	Elem() Type

	// 根据索引获取字段
	Field(i int) StructField

	// 根据索引链获取嵌套字段
	FieldByIndex(index []int) StructField

	// 根据名称获取字段
	FieldByName(name string) (StructField, bool)

	// 根据指定的匹配函数 math 获取字段
	FieldByNameFunc(match func(string) bool) (StructField, bool)

	// 根据索引获取函数的参数信息
	In(i int) Type

	// Key returns a map type's key type.
	// It panics if the type's Kind is not Map.
	Key() Type

	// Len returns an array type's length.
	// It panics if the type's Kind is not Array.
	Len() int

	// 获取字段数量
	NumField() int

	// 获取函数的参数数量
	NumIn() int

	// 获取函数的返回值数量
	NumOut() int

	// 根据索引获取函数的返回值信息
	Out(i int) Type

	common() *rtype
	uncommon() *uncommonType
}

```

### value 方法集

```
// 特殊


// 判断 v 值是否可寻址
// 1、指针的 Elem() 可寻址
// 2、切片的元素可寻址
// 3、可寻址数组的元素可寻址
// 4、可寻址结构体的字段可寻址，方法不可寻址
// 也就是说，如果 v 值是指向数组的指针“&数组”，通过 v.Elem() 获取该指针指向的数组，那么
// 该数组就是可寻址的，同时该数组的元素也是可寻址的，如果 v 就是一个普通数组，不是通过解引
// 用得到的数组，那么该数组就不可寻址，其元素也不可寻址。结构体亦然。
func (v Value) CanAddr() bool

// 获取 v 值的地址，相当于 & 取地址操作。v 值必须可寻址。
func (v Value) Addr() reflect.Value

// 判断 v 值是否可以被修改。只有可寻址的 v 值可被修改。
// 结构体中的非导出字段（通过 Field() 等方法获取的）不能修改，所有方法不能修改。
func (v Value) CanSet() bool

// 判断 v 值是否可以转换为接口类型
// 结构体中的非导出字段（通过 Field() 等方法获取的）不能转换为接口类型
func (v Value) CanInterface() bool

// 将 v 值转换为空接口类型。v 值必须可转换为接口类型。
func (v Value) Interface() interface{}

// 使用一对 uintptr 返回接口的数据
func (v Value) InterfaceData() [2]uintptr






// 指针
// 将 v 值转换为 uintptr 类型，v 值必须是切片、映射、通道、函数、指针、自由指针。
func (v Value) Pointer() uintptr

// 获取 v 值的地址。v 值必须是可寻址类型（CanAddr）。
func (v Value) UnsafeAddr() uintptr

// 将 UnsafePointer 类别的 v 值修改为 x，v 值必须是 UnsafePointer 类别，必须可修改。
func (v Value) SetPointer(x unsafe.Pointer)

// 判断 v 值是否为 nil，v 值必须是切片、映射、通道、函数、接口、指针。
// IsNil 并不总等价于 Go 的潜在比较规则，比如对于 var i interface{}，i == nil 将返回
// true，但是 reflect.ValueOf(i).IsNil() 将 panic。
func (v Value) IsNil() bool

// 获取“指针所指的对象”或“接口所包含的对象”
func (v Value) Elem() reflect.Value


// 通用

// 获取 v 值的字符串描述
func (v Value) String() string

// 获取 v 值的类型
func (v Value) Type() reflect.Type

// 返回 v 值的类别，如果 v 是空值，则返回 reflect.Invalid。
func (v Value) Kind() reflect.Kind

// 获取 v 的方法数量
func (v Value) NumMethod() int

// 根据索引获取 v 值的方法，方法必须存在，否则 panic
// 使用 Call 调用方法的时候不用传入接收者，Go 会自动把 v 作为接收者传入。
func (v Value) Method(int) reflect.Value

// 根据名称获取 v 值的方法，如果该方法不存在，则返回空值（reflect.Invalid）。
func (v Value) MethodByName(string) reflect.Value

// 判断 v 本身（不是 v 值）是否为零值。
// 如果 v 本身是零值，则除了 String 之外的其它所有方法都会 panic。
func (v Value) IsValid() bool

// 将 v 值转换为 t 类型，v 值必须可转换为 t 类型，否则 panic。
func (v Value) Convert(t Type) reflect.Value

// 获取

// 获取 v 值的内容，如果 v 值不是有符号整型，则 panic。
func (v Value) Int() int64

// 获取 v 值的内容，如果 v 值不是无符号整型（包括 uintptr），则 panic。
func (v Value) Uint() uint64

// 获取 v 值的内容，如果 v 值不是浮点型，则 panic。
func (v Value) Float() float64

// 获取 v 值的内容，如果 v 值不是复数型，则 panic。
func (v Value) Complex() complex128

// 获取 v 值的内容，如果 v 值不是布尔型，则 panic。
func (v Value) Bool() bool

// 获取 v 值的长度，v 值必须是字符串、数组、切片、映射、通道。
func (v Value) Len() int

// 获取 v 值的容量，v 值必须是数值、切片、通道。
func (v Value) Cap() int

// 获取 v 值的第 i 个元素，v 值必须是字符串、数组、切片，i 不能超出范围。
func (v Value) Index(i int) reflect.Value

// 获取 v 值的内容，如果 v 值不是字节切片，则 panic。
func (v Value) Bytes() []byte

// 获取 v 值的切片，切片长度 = j - i，切片容量 = v.Cap() - i。
// v 必须是字符串、数值、切片，如果是数组则必须可寻址。i 不能超出范围。
func (v Value) Slice(i, j int) reflect.Value

// 获取 v 值的切片，切片长度 = j - i，切片容量 = k - i。
// i、j、k 不能超出 v 的容量。i <= j <= k。
// v 必须是字符串、数值、切片，如果是数组则必须可寻址。i 不能超出范围。
func (v Value) Slice3(i, j, k int) reflect.Value

// 根据 key 键获取 v 值的内容，v 值必须是映射。
// 如果指定的元素不存在，或 v 值是未初始化的映射，则返回零值（reflect.ValueOf(nil)）
func (v Value) MapIndex(key Value) reflect.Value

// 获取 v 值的所有键的无序列表，v 值必须是映射。
// 如果 v 值是未初始化的映射，则返回空列表。
func (v Value) MapKeys() []reflect.Value

// 判断 x 是否超出 v 值的取值范围，v 值必须是有符号整型。
func (v Value) OverflowInt(x int64) bool

// 判断 x 是否超出 v 值的取值范围，v 值必须是无符号整型。
func (v Value) OverflowUint(x uint64) bool

// 判断 x 是否超出 v 值的取值范围，v 值必须是浮点型。
func (v Value) OverflowFloat(x float64) bool

// 判断 x 是否超出 v 值的取值范围，v 值必须是复数型。
func (v Value) OverflowComplex(x complex128) bool

------------------------------

// 设置（这些方法要求 v 值必须可修改）

// 设置 v 值的内容，v 值必须是有符号整型。
func (v Value) SetInt(x int64)

// 设置 v 值的内容，v 值必须是无符号整型。
func (v Value) SetUint(x uint64)

// 设置 v 值的内容，v 值必须是浮点型。
func (v Value) SetFloat(x float64)

// 设置 v 值的内容，v 值必须是复数型。
func (v Value) SetComplex(x complex128)

// 设置 v 值的内容，v 值必须是布尔型。
func (v Value) SetBool(x bool)

// 设置 v 值的内容，v 值必须是字符串。
func (v Value) SetString(x string)

// 设置 v 值的长度，v 值必须是切片，n 不能超出范围，不能为负数。
func (v Value) SetLen(n int)

// 设置 v 值的内容，v 值必须是切片，n 不能超出范围，不能小于 Len。
func (v Value) SetCap(n int)

// 设置 v 值的内容，v 值必须是字节切片。x 可以超出 v 值容量。
func (v Value) SetBytes(x []byte)

// 设置 v 值的键和值，如果键存在，则修改其值，如果键不存在，则添加键和值。
// 如果将 val 设置为零值（reflect.ValueOf(nil)），则删除该键。
// 如果 v 值是一个未初始化的 map，则 panic。
func (v Value) SetMapIndex(key, val reflect.Value)

// 设置 v 值的内容，v 值必须可修改，x 必须可以赋值给 v 值。
func (v Value) Set(x reflect.Value)

------------------------------

// 结构体

// 获取 v 值的字段数量，v 值必须是结构体。
func (v Value) NumField() int

// 根据索引获取 v 值的字段，v 值必须是结构体。如果字段不存在则 panic。
func (v Value) Field(i int) reflect.Value

// 根据索引链获取 v 值的嵌套字段，v 值必须是结构体。
func (v Value) FieldByIndex(index []int) reflect.Value

// 根据名称获取 v 值的字段，v 值必须是结构体。
// 如果指定的字段不存在，则返回零值（reflect.ValueOf(nil)）
func (v Value) FieldByName(string) reflect.Value

// 根据匹配函数 match 获取 v 值的字段，v 值必须是结构体。
// 如果没有匹配的字段，则返回零值（reflect.ValueOf(nil)）
func (v Value) FieldByNameFunc(match func(string) bool) Value


// 函数

// 通过参数列表 in 调用 v 值所代表的函数（或方法）。函数的返回值存入 r 中返回。
// 要传入多少参数就在 in 中存入多少元素。
// Call 即可以调用定参函数（参数数量固定），也可以调用变参函数（参数数量可变）。
func (v Value) Call(in []Value) (r []Value)

// 通过参数列表 in 调用 v 值所代表的函数（或方法）。函数的返回值存入 r 中返回。
// 函数指定了多少参数就在 in 中存入多少元素，变参作为一个单独的参数提供。
// CallSlice 只能调用变参函数。
func (v Value) CallSlice(in []Value) []Value



// 通道

// 发送数据（会阻塞），v 值必须是可写通道。
func (v Value) Send(x reflect.Value)

// 接收数据（会阻塞），v 值必须是可读通道。
func (v Value) Recv() (x reflect.Value, ok bool)

// 尝试发送数据（不会阻塞），v 值必须是可写通道。
func (v Value) TrySend(x reflect.Value) bool

// 尝试接收数据（不会阻塞），v 值必须是可读通道。
func (v Value) TryRecv() (x reflect.Value, ok bool)

// 关闭通道，v 值必须是通道。
func (v Value) Close()


```

```

// 示例
var f1 = func(a int, b []int) { fmt.Println(a, b) }
var f2 = func(a int, b ...int) { fmt.Println(a, b) }

func main() {
	v1 := reflect.ValueOf(f1)
	v2 := reflect.ValueOf(f2)

	a := reflect.ValueOf(1)
	b := reflect.ValueOf([]int{1, 2, 3})

	v1.Call([]reflect.Value{a, b})
	v2.Call([]reflect.Value{a, a, a, a, a, a})

	//v1.CallSlice([]reflect.Value{a, b}) // 非变参函数，不能用 CallSlice。
	v2.CallSlice([]reflect.Value{a, b})
}

```

### 样例

- 类型的字段标识

下面是分析一个struct值，t，的简单例子。我们用这个struct的地址创建一个反射对象，因为我们想一会改变它的值。然后我们把typeofT变量设置为这个反射对象的类型，接着使用一些直接的方法调用（细节请见reflect包）来迭代各个域。注意，我们从struct类型中提取了各个域的名字，但是这些域本身都是rreflect.Value对象。

```
type T struct {
    A int
    B string
}
t := T{23, "skidoo"}
s := reflect.ValueOf(&t).Elem()
typeOfT := s.Type()//把s.Type()返回的Type对象复制给typeofT，typeofT也是一个反射。
for i := 0; i < s.NumField(); i++ {
    f := s.Field(i)//迭代s的各个域，注意每个域仍然是反射。
    fmt.Printf("%d: %s %s = %v\n", i,
        typeOfT.Field(i).Name, f.Type(), f.Interface())//提取了每个域的名字
}

```

```
0: A int = 23
1: B string = skidoo

```

reflect.Type的Field方法将返回一个reflect.StructField，里面含有每个成员的名字、类型和可选的成员标签等信息。



因为s包含了一个settable的反射对象，所以我们可以修改这个structure的各个域。

```
s.Field(0).SetInt(77)
s.Field(1).SetString("Sunset Strip")
fmt.Println("t is now", t)

t is now {77 Sunset Strip}

```

- 类型的方法集

```
func Print(x interface{}) {
    v := reflect.ValueOf(x)
    t := v.Type()
    fmt.Printf("type %s\n", t)

    for i := 0; i < v.NumMethod(); i++ {
        methType := v.Method(i).Type()
        fmt.Printf("func (%s) %s%s\n", t, t.Method(i).Name,
            strings.TrimPrefix(methType.String(), "func"))
    }
}

```

reflect.Type和reflect.Value都提供了一个Method方法。每次t.Method(i)调用将一个reflect.Method的实例，对应一个用于描述一个方法的名称和类型的结构体。每次v.Method(i)方法调用都返回一个reflect.Value以表示对应的值（§6.4），也就是一个方法是帮到它的接收者的。使用reflect.Value.Call方法（我们之类没有演示），将可以调用一个Func类型的Value，但是这个例子中只用到了它的类型。

```
methods.Print(time.Hour)
// Output:
// type time.Duration
// func (time.Duration) Hours() float64
// func (time.Duration) Minutes() float64
// func (time.Duration) Nanoseconds() int64
// func (time.Duration) Seconds() float64
// func (time.Duration) String() string

methods.Print(new(strings.Replacer))
// Output:
// type *strings.Replacer
// func (*strings.Replacer) Replace(string) string
// func (*strings.Replacer) WriteString(io.Writer, string) (int, error)

```

## 反射的原理

### Typeof


Typeof 函数非常简单，在调用 Typeof 函数的时候，变量就已经被转化为 interface 类型，Typeof 只需要将它的 typ 属性取出来即可。

```
func TypeOf(i interface{}) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	return toType(eface.typ)
}

func toType(t *rtype) Type {
	if t == nil {
		return nil
	}
	return t
}
```
#### type.Name 函数

解析类型的名称是一个反射很基础的功能，它和 String 方法的不同在于，它不会包含类型所在包的名字，例如 `main.Cat` 与 `Cat`，所以一定不要用 name 来区分类型。

从实现来看，Name 是建立在 String 函数的基础上的，它找到了 `.` 这个字符然后分割了字符串。

从下面的代码中可以看到，rtype 的 str(nameoff) 属性并不是简单的距离，而是距离各个模块 types 的距离。

```
func (t *rtype) Name() string {
	if t.tflag&tflagNamed == 0 {
		return ""
	}
	s := t.String()
	i := len(s) - 1
	for i >= 0 && s[i] != '.' {
		i--
	}
	return s[i+1:]
}

func (t *rtype) String() string {
	s := t.nameOff(t.str).name()
	if t.tflag&tflagExtraStar != 0 {
		return s[1:]
	}
	return s
}

func (t *rtype) nameOff(off nameOff) name {
	return name{(*byte)(resolveNameOff(unsafe.Pointer(t), int32(off)))}
}

// reflect_resolveNameOff resolves a name offset from a base pointer.
//go:linkname reflect_resolveNameOff reflect.resolveNameOff
func reflect_resolveNameOff(ptrInModule unsafe.Pointer, off int32) unsafe.Pointer {
	return unsafe.Pointer(resolveNameOff(ptrInModule, nameOff(off)).bytes)
}

func resolveNameOff(ptrInModule unsafe.Pointer, off nameOff) name {
	if off == 0 {
		return name{}
	}
	base := uintptr(ptrInModule)
	for md := &firstmoduledata; md != nil; md = md.next {
		if base >= md.types && base < md.etypes {
			res := md.types + uintptr(off)
			if res > md.etypes {
				println("runtime: nameOff", hex(off), "out of range", hex(md.types), "-", hex(md.etypes))
				throw("runtime: name offset out of range")
			}
			return name{(*byte)(unsafe.Pointer(res))}
		}
	}

	// No module found. see if it is a run time name.
	reflectOffsLock()
	res, found := reflectOffs.m[int32(off)]
	reflectOffsUnlock()
	if !found {
		println("runtime: nameOff", hex(off), "base", hex(base), "not in ranges:")
		for next := &firstmoduledata; next != nil; next = next.next {
			println("\ttypes", hex(next.types), "etypes", hex(next.etypes))
		}
		throw("runtime: name offset base pointer out of range")
	}
	return name{(*byte)(res)}
}
```

### type.Field

```
func (t *rtype) Field(i int) StructField {
	if t.Kind() != Struct {
		panic("reflect: Field of non-struct type")
	}
	tt := (*structType)(unsafe.Pointer(t))
	return tt.Field(i)
}

func (t *structType) Field(i int) (f StructField) {
	if i < 0 || i >= len(t.fields) {
		panic("reflect: Field index out of bounds")
	}
	p := &t.fields[i]
	f.Type = toType(p.typ)
	f.Name = p.name.name()
	f.Anonymous = p.embedded()
	if !p.name.isExported() {
		f.PkgPath = t.pkgPath.name()
	}
	if tag := p.name.tag(); tag != "" {
		f.Tag = StructTag(tag)
	}
	f.Offset = p.offset()

	// NOTE(rsc): This is the only allocation in the interface
	// presented by a reflect.Type. It would be nice to avoid,
	// at least in the common cases, but we need to make sure
	// that misbehaving clients of reflect cannot affect other
	// uses of reflect. One possibility is CL 5371098, but we
	// postponed that ugliness until there is a demonstrated
	// need for the performance. This is issue 2320.
	f.Index = []int{i}
	return
}

```

### type.Method 方法

对于 golang 里面的类型，它们的方法都是存储在 uncommon 的部分当中，而且他们的数据结构是：

```
type method struct {
	name nameOff // name of method
	mtyp typeOff // method type (without receiver)
	ifn  textOff // fn used in interface call (one-word receiver)
	tfn  textOff // fn used for normal method call
}

```

数据结构中，mtyp 是 method 类型的地址，ifn 是接口函数的地址，tfn 是普通函数的地址。

它会被 Method 函数转换为 Method 类型：

```
type Method struct {
	// Name is the method name.
	// PkgPath is the package path that qualifies a lower case (unexported)
	// method name. It is empty for upper case (exported) method names.
	// The combination of PkgPath and Name uniquely identifies a method
	// in a method set.
	// See https://golang.org/ref/spec#Uniqueness_of_identifiers
	Name    string
	PkgPath string

	Type  Type  // method type
	Func  Value // func with receiver as first argument
	Index int   // index for Type.Method
}

```
Method 的 Type 由 mtyp 而来，Func 由 tfn/ifn 而来，而 Func 是 Value 类型，Func.typ 还是 mtyp，ptr 是 tfn/ifn。


```
func (t *rtype) Method(i int) (m Method) {
	if t.Kind() == Interface {
		tt := (*interfaceType)(unsafe.Pointer(t))
		return tt.Method(i)
	}
	methods := t.exportedMethods()
	if i < 0 || i >= len(methods) {
		panic("reflect: Method index out of range")
	}
	p := methods[i]
	pname := t.nameOff(p.name)
	m.Name = pname.name()
	fl := flag(Func)
	mtyp := t.typeOff(p.mtyp)
	ft := (*funcType)(unsafe.Pointer(mtyp))
	in := make([]Type, 0, 1+len(ft.in()))
	in = append(in, t)
	for _, arg := range ft.in() {
		in = append(in, arg)
	}
	out := make([]Type, 0, len(ft.out()))
	for _, ret := range ft.out() {
		out = append(out, ret)
	}
	mt := FuncOf(in, out, ft.IsVariadic())
	m.Type = mt
	tfn := t.textOff(p.tfn)
	fn := unsafe.Pointer(&tfn)
	m.Func = Value{mt.(*rtype), fn, fl}

	m.Index = i
	return m
}


func (t *rtype) exportedMethods() []method {
	ut := t.uncommon()
	if ut == nil {
		return nil
	}
	return ut.exportedMethods()
}

func (t *uncommonType) exportedMethods() []method {
	if t.xcount == 0 {
		return nil
	}
	return (*[1 << 16]method)(add(unsafe.Pointer(t), uintptr(t.moff), "t.xcount > 0"))[:t.xcount:t.xcount]
}
```

### ValueOf


```
func ValueOf(i interface{}) Value {
	if i == nil {
		return Value{}
	}

	// TODO: Maybe allow contents of a Value to live on the stack.
	// For now we make the contents always escape to the heap. It
	// makes life easier in a few places (see chanrecv/mapassign
	// comment below).
	escapes(i)

	return unpackEface(i)
}

func unpackEface(i interface{}) Value {
	e := (*emptyInterface)(unsafe.Pointer(&i))
	// NOTE: don't read e.word until we know whether it is really a pointer or not.
	t := e.typ
	if t == nil {
		return Value{}
	}
	f := flag(t.Kind())
	if ifaceIndir(t) {
		f |= flagIndir
	}
	return Value{t, e.word, f}
}
```

### value.Field

通过 value 的 Field 可以获取到结构体的内部属性值，结构体的内部属性都是 structField 类型的，每个 structField.offsetEmbed 是该属性值距离结构体地址的偏移量。

```
func (v Value) Field(i int) Value {
	if v.kind() != Struct {
		panic(&ValueError{"reflect.Value.Field", v.kind()})
	}
	tt := (*structType)(unsafe.Pointer(v.typ))
	if uint(i) >= uint(len(tt.fields)) {
		panic("reflect: Field index out of range")
	}
	field := &tt.fields[i]
	typ := field.typ

	// Inherit permission bits from v, but clear flagEmbedRO.
	fl := v.flag&(flagStickyRO|flagIndir|flagAddr) | flag(typ.Kind())
	// Using an unexported field forces flagRO.
	if !field.name.isExported() {
		if field.embedded() {
			fl |= flagEmbedRO
		} else {
			fl |= flagStickyRO
		}
	}
	// Either flagIndir is set and v.ptr points at struct,
	// or flagIndir is not set and v.ptr is the actual struct data.
	// In the former case, we want v.ptr + offset.
	// In the latter case, we must have field.offset = 0,
	// so v.ptr + field.offset is still the correct address.
	ptr := add(v.ptr, field.offset(), "same as non-reflect &v.field")
	return Value{typ, ptr, fl}
}

type structField struct {
	name        name    // name is always non-empty
	typ         *rtype  // type of field
	offsetEmbed uintptr // byte offset of field<<1 | isEmbedded
}

func (f *structField) offset() uintptr {
	return f.offsetEmbed >> 1
}

```

### value.Method

我们从下面的代码中可以看到，Method 也是返回一个 Value，但是这个 Value 的 ptr 并不是第 i 个函数的地址，而是原封不动的将原 value 的 ptr 返回了，仅仅是对 flag 设置比特位而已。


```
func (v Value) Method(i int) Value {
	if v.typ == nil {
		panic(&ValueError{"reflect.Value.Method", Invalid})
	}
	if v.flag&flagMethod != 0 || uint(i) >= uint(v.typ.NumMethod()) {
		panic("reflect: Method index out of range")
	}
	if v.typ.Kind() == Interface && v.IsNil() {
		panic("reflect: Method on nil interface value")
	}
	fl := v.flag & (flagStickyRO | flagIndir) // Clear flagEmbedRO
	fl |= flag(Func)
	fl |= flag(i)<<flagMethodShift | flagMethod
	return Value{v.typ, v.ptr, fl}
}

```

### value.Call

Call 函数目的是调用 value 的相应的函数，这里和 Method 是相互呼应的，使用了 flag 的 flagMethodShift，得到了相应的函数地址。

```
func (v Value) Call(in []Value) []Value {
	v.mustBe(Func)
	v.mustBeExported()
	return v.call("Call", in)
}

func (v Value) call(op string, in []Value) []Value {
	// Get function pointer, type.
	t := (*funcType)(unsafe.Pointer(v.typ))
	var (
		fn       unsafe.Pointer
		rcvr     Value
		rcvrtype *rtype
	)
	if v.flag&flagMethod != 0 {
		rcvr = v
		rcvrtype, t, fn = methodReceiver(op, v, int(v.flag)>>flagMethodShift)
	} else if v.flag&flagIndir != 0 {
		fn = *(*unsafe.Pointer)(v.ptr)
	} else {
		fn = v.ptr
	}

	if fn == nil {
		panic("reflect.Value.Call: call of nil function")
	}

	isSlice := op == "CallSlice"
	n := t.NumIn()
	if isSlice {
		if !t.IsVariadic() {
			panic("reflect: CallSlice of non-variadic function")
		}
		if len(in) < n {
			panic("reflect: CallSlice with too few input arguments")
		}
		if len(in) > n {
			panic("reflect: CallSlice with too many input arguments")
		}
	} else {
		if t.IsVariadic() {
			n--
		}
		if len(in) < n {
			panic("reflect: Call with too few input arguments")
		}
		if !t.IsVariadic() && len(in) > n {
			panic("reflect: Call with too many input arguments")
		}
	}
	for _, x := range in {
		if x.Kind() == Invalid {
			panic("reflect: " + op + " using zero Value argument")
		}
	}
	for i := 0; i < n; i++ {
		if xt, targ := in[i].Type(), t.In(i); !xt.AssignableTo(targ) {
			panic("reflect: " + op + " using " + xt.String() + " as type " + targ.String())
		}
	}
	if !isSlice && t.IsVariadic() {
		// prepare slice for remaining values
		m := len(in) - n
		slice := MakeSlice(t.In(n), m, m)
		elem := t.In(n).Elem()
		for i := 0; i < m; i++ {
			x := in[n+i]
			if xt := x.Type(); !xt.AssignableTo(elem) {
				panic("reflect: cannot use " + xt.String() + " as type " + elem.String() + " in " + op)
			}
			slice.Index(i).Set(x)
		}
		origIn := in
		in = make([]Value, n+1)
		copy(in[:n], origIn)
		in[n] = slice
	}

	nin := len(in)
	if nin != t.NumIn() {
		panic("reflect.Value.Call: wrong argument count")
	}
	nout := t.NumOut()

	// Compute frame type.
	frametype, _, retOffset, _, framePool := funcLayout(t, rcvrtype)

	// Allocate a chunk of memory for frame.
	var args unsafe.Pointer
	if nout == 0 {
		args = framePool.Get().(unsafe.Pointer)
	} else {
		// Can't use pool if the function has return values.
		// We will leak pointer to args in ret, so its lifetime is not scoped.
		args = unsafe_New(frametype)
	}
	off := uintptr(0)

	// Copy inputs into args.
	if rcvrtype != nil {
		storeRcvr(rcvr, args)
		off = ptrSize
	}
	for i, v := range in {
		v.mustBeExported()
		targ := t.In(i).(*rtype)
		a := uintptr(targ.align)
		off = (off + a - 1) &^ (a - 1)
		n := targ.size
		if n == 0 {
			// Not safe to compute args+off pointing at 0 bytes,
			// because that might point beyond the end of the frame,
			// but we still need to call assignTo to check assignability.
			v.assignTo("reflect.Value.Call", targ, nil)
			continue
		}
		addr := add(args, off, "n > 0")
		v = v.assignTo("reflect.Value.Call", targ, addr)
		if v.flag&flagIndir != 0 {
			typedmemmove(targ, addr, v.ptr)
		} else {
			*(*unsafe.Pointer)(addr) = v.ptr
		}
		off += n
	}

	// Call.
	call(frametype, fn, args, uint32(frametype.size), uint32(retOffset))

	// For testing; see TestCallMethodJump.
	if callGC {
		runtime.GC()
	}

	var ret []Value
	if nout == 0 {
		typedmemclr(frametype, args)
		framePool.Put(args)
	} else {
		// Zero the now unused input area of args,
		// because the Values returned by this function contain pointers to the args object,
		// and will thus keep the args object alive indefinitely.
		typedmemclrpartial(frametype, args, 0, retOffset)

		// Wrap Values around return values in args.
		ret = make([]Value, nout)
		off = retOffset
		for i := 0; i < nout; i++ {
			tv := t.Out(i)
			a := uintptr(tv.Align())
			off = (off + a - 1) &^ (a - 1)
			if tv.Size() != 0 {
				fl := flagIndir | flag(tv.Kind())
				ret[i] = Value{tv.common(), add(args, off, "tv.Size() != 0"), fl}
				// Note: this does introduce false sharing between results -
				// if any result is live, they are all live.
				// (And the space for the args is live as well, but as we've
				// cleared that space it isn't as big a deal.)
			} else {
				// For zero-sized return value, args+off may point to the next object.
				// In this case, return the zero value instead.
				ret[i] = Zero(tv)
			}
			off += tv.Size()
		}
	}

	return ret
}
```
