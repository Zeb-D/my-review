本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

有一次午饭碰上一位大佬，讨论到使用channel实现一种简单的通知机制，前面的思想交流都没什么火花，后面关于空值是选用`interface{}`还是`struct{}`；

众所周知，`struct{}`是不允许`nil`值的，`struct{}{}`是不用占内存的；还有一点就是`interface{} 允许nil`在 `close(channel)`还有`buf`和`没有buf`取出的值有二义性；



另外，在学习一些框架时

```
// CanSkipFuncs will skip valid if RequiredFirst is true and the struct field's value is empty
var CanSkipFuncs = map[string]struct{}{
    "Email":   {},
    "IP":      {},
    "Mobile":  {},
    "Tel":     {},
    "Phone":   {},
    "ZipCode": {},
}
```

或将一个空结构体写入到通道中的使用：

```
w.ch <- struct{}{}
```

那为什么要这样使用空结构体呢？让我们一起来学习下空结构体的应用以及底层原理。



### 什么是空结构体

首先来看看空结构体是什么。空结构体也是结构体类型，具有结构体的一切特性。但该结构体中没有任何字段组合。所以，**该空结构体类型的变量占用的空间为0**。

我们通过unsafe.Sizeof函数来验证一下。unsafe.Sizeof函数的作用是返回一个数据类型所占的空间大小。我们验证一下：

```
var s struct{}
fmt.Println(unsafe.Sizeof(s)) // prints 0
```

我们看到打印的结果是0，表明struct{}的类型占用的空间是0。

我们还可以通过reflect的类型来验证。

```
var s struct{}
typ := reflect.TypeOf(s)
fmt.Println(typ.Size()) // 0
```

我们看到，通过映射变量s的类型，输出空类型的空间大小也是0。



### 原理

#### 特殊变量：zerobase 

我们知道，在编程语言中，变量的作用就是在内存中，标记和存储数据的。也就是说每个变量会对应着一块内存空间，既然是内存空间，那就应该有对应的内存地址。

那空结构体类型变量的地址是什么呢？我们通过如下代码来看下：

```
package main
 
import (
    "fmt"
    "unsafe"
)
 
type emptyStruct struct{}
 
func main() {
    a := struct{}{}
    b := struct{}{}
 
    c := emptyStruct{}
 
    fmt.Println(a)
    fmt.Printf("%pn", &a) //0x116be80
    fmt.Printf("%pn", &b) //0x116be80
    fmt.Printf("%pn", &c) //0x116be80
 
    fmt.Println(a == b) //true
}
```

我们发现，所有空结构体类型的变量地址都是一样的。那这是为什么呢？

在底层实现中，这和一个很重要的 zerobase 变量有关(在runtime里多次使用到了这个变量），而zerobase 变量是一个 uintptr 的全局变量，占用8个字节。在go源码src/runtime/malloc.go中有如下定义：

```
// base address for all 0-byte allocations
var zerobase uintptr
```

只要你将struct{} 赋值给一个或者多个变量，它都返回这个 zerobase 的地址，这点我们上面已经证实过这一点了。

#### **mallocgc**

在golang中大量的地方使用到了这个 zerobase 变量，只要分配的内存为0，就返回这个变量地址，在go源码src/runtime/malloc.go的mallocgc函数中定义如下：

```
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    if gcphase == _GCmarktermination {
  throw("mallocgc called with gcphase == _GCmarktermination")
    }

    if size == 0 {
  return unsafe.Pointer(&zerobase)
    }
    ...
}
```



### 定义的各种姿势

#### 原生定义

```
a := struct{}{}
```

`struct{}`  可以就认为是一种类型，a 变量就是 `struct {}`  类型的一种变量，地址为 `runtime.zerobase` ，大小为 0 ，不占内存。

#### 重定义类型

golang 使用 `type`  关键字定义新的类型，比如：

```
type emptyStruct struct{}
```

定义出来的 `emptyStruct`  是新的类型，具有对应的 `type` 结构，但是性质 `struct{}` 完全一致，编译器对于 `emptryStruct`类型的内存分配，也是直接给 `zerobase` 地址的。

#### 匿名嵌套类型

`struct{}`  作为一个匿名字段，内嵌其他结构体。这种情况是怎么样的？

**匿名嵌套方式一**

```
type emptyStruct struct{}
type Object struct {
    emptyStruct
}
```

**匿名嵌套方式二**

```
type Object1 struct {
    _ struct {}
}
```

记住一点，空结构体还是空结构体，类型变量本身绝对不分配内存（ size=0 ），所以编译器对以上的 `Object`，`Object1` 两种类型的处理和空结构体类型是一致的，分配地址为 `runtime.zerobase` 地址，变量大小为0，不占任何内存大小。

#### 内置字段

内置字段的场景没有什么特殊的，主要是地址和长度的对齐要考虑。还是只需要注意 3 个要点：

- 空结构体的类型不占内存大小；
- 地址偏移要和自身类型对齐；
- 整体类型长度要和最长的字段类型长度对齐；

**场景一：struct {}  在最前面**

这种场景非常好理解，`struct {}` 字段类型在最前面，这种类型不占空间，所以自然第二个字段的地址和整个变量的地址一致。

```
// Object1 类型变量占用 1 个字节
type Object1 struct {
 s struct {}
 b byte
}

// Object2 类型变量占用 8 个字节
type Object2 struct {
 s struct {}
 n int64
}

o1 := Object1{ }
o2 := Object2{ }
```

内存怎么分配？

- `&o1`  和 `&o1.s` 是一致的，变量 `o1`  的内存大小对齐到 1 字节；
- `&o2`  和 `&o2.s` 是一致的，变量 `o2`  的内存大小对齐到 8 字节；

这种分配是满足对齐规则的，编译器也不会对这种 `struct {}` 字段做任何特殊的字节填充。

**场景二：struct {} 在中间**

```
// Object1 类型变量占用 16 个字节
type Object1 struct {
 b  byte
 s  struct{}
 b1 int64
}

o1 := Object1{ }
```

- 按照对齐规则，变量 `o1` 占用 16 个字节；
- `&o1.s` 和 `&o1.b1`  相同；

编译器不会对 `struct { }` 做任何字节填充。

**场景三：struct {} 在最后**

这个场景稍微注意下，因为编译器遇到之后会做特殊的字节填充补齐，如下；

```
type Object1 struct {
 b byte
 s struct{}
}

type Object2 struct {
 n int64
 s struct{}
}

type Object3 struct {
 n int16
 m int16
 s struct{}
}

type Object4 struct {
 n  int16
 m  int64
 s  struct{}
}

o1 := Object1 { }
o2 := Object2 { }
o3 := Object3 { }
o4 := Object4 { }
```

编译器在遇到这种 `struct {}`  在**最后一个字段**的场景，会进行特殊填充，`struct { }` 作为最后一个字段，会被填充对齐到前一个字段的大小，地址偏移对齐规则不变；

可以现在心里思考下，`o1`，`o2`，`o3`，`o4`  这四个对象的内存分配分别占多少空间？下面解密：

- 变量 `o1`  大小为 2 字节；
- 变量 `o2`  大小为 16 字节；
- 变量 `o3`  大小为 6 字节；
- 变量 `o4`  大小为 24 字节；

这种情况，需要先把 `struct {}`  按照前一个字段的长度分配 padding 内存，然后整个变量按照地址和长度的对齐规则不变。

#### `struct {}` 作为 receiver

receiver 这个是 golang 里 struct 具有的基础特点。空结构体本质上作为结构体也是一样的，可以作为 receiver 来定义方法。

```
type emptyStruct struct{}

func (e *emptyStruct) FuncB(n, m int) {
}
func (e emptyStruct) FuncA(n, m int) {
}

func main() {
 a := emptyStruct{}

 n := 1
 m := 2

 a.FuncA(n, m)
 a.FuncB(n, m)
}
```

receiver 这种写法是 golang 支撑面向对象的基础，本质上的实现也是非常简单，常规情况（普通的结构体）可以翻译成：

```
func FuncA (e *emptyStruct, n, m int) {
}
func FuncB (e  emptyStruct, n, m int) {
}
```

**编译器只是把对象的值或地址作为第一个参数传给这个参数而已，就这么简单。** 但是在这里要提一点，空结构体稍微有一点点不一样，空结构体应该翻译成：

```
func FuncA (e *emptyStruct, n, m int) {
}
func FuncB (n, m int) {
}
```

极其简单的代码，对应的汇编实际代码如下：

FuncA，FuncB 就这么简单，如下：

```
00000000004525b0 <main.(*emptyStruct).FuncB>:
  4525b0: c3                    retq   

00000000004525c0 <main.emptyStruct.FuncA>:
  4525c0: c3                    retq    
```

main 函数

```
00000000004525d0 <main.main>:
  4525d0: 64 48 8b 0c 25 f8 ff  mov    %fs:0xfffffffffffffff8,%rcx
  4525d9: 48 3b 61 10           cmp    0x10(%rcx),%rsp
  4525dd: 76 63                 jbe    452642 <main.main+0x72>
  4525df: 48 83 ec 30           sub    $0x30,%rsp
  4525e3: 48 89 6c 24 28        mov    %rbp,0x28(%rsp)
  4525e8: 48 8d 6c 24 28        lea    0x28(%rsp),%rbp
  4525ed: 48 c7 44 24 18 01 00  movq   $0x1,0x18(%rsp)
  4525f6: 48 c7 44 24 20 02 00  movq   $0x2,0x20(%rsp)
  4525ff: 48 8b 44 24 18        mov    0x18(%rsp),%rax
  452604: 48 89 04 24           mov    %rax,(%rsp)   // n 变量值压栈（第一个参数）
  452608: 48 c7 44 24 08 02 00  movq   $0x2,0x8(%rsp)  // m 变量值压栈（第二个参数）
  452611: e8 aa ff ff ff        callq  4525c0 <main.emptyStruct.FuncA>
  452616: 48 8d 44 24 18        lea    0x18(%rsp),%rax
  45261b: 48 89 04 24           mov    %rax,(%rsp)   // $rax 里面是 zerobase 的值，压栈（第一个参数）；
  45261f: 48 8b 44 24 18        mov    0x18(%rsp),%rax
  452624: 48 89 44 24 08        mov    %rax,0x8(%rsp)  // n 变量值压栈（第二个参数）
  452629: 48 8b 44 24 20        mov    0x20(%rsp),%rax
  45262e: 48 89 44 24 10        mov    %rax,0x10(%rsp)  // m 变量值压栈（第三个参数）
  452633: e8 78 ff ff ff        callq  4525b0 <main.(*emptyStruct).FuncB>
  452638: 48 8b 6c 24 28        mov    0x28(%rsp),%rbp
  45263d: 48 83 c4 30           add    $0x30,%rsp
  452641: c3                    retq   
  452642: e8 b9 7a ff ff        callq  44a100 <runtime.morestack_noctxt>
  452647: eb 87                 jmp    4525d0 <main.main>
```

通过这段代码证实几个点：

1. receiver 其实就是一种语法糖，本质上就是作为第一个参数传入函数；
2. receiver 为值的场景，不需要传空结构体做第一个参数，因为空结构体没有值；
3. receiver 为一个指针的场景，对象地址作为第一个参数传入函数，函数调用的时候，编译器传入 `zerobase` 的值（编译期间就可以确认）；

在二进制编译之后，一般 `e.FuncA`  的调用，第一个参数是直接压入 `&zerobase` 到栈里。

总结几个知识点：

- receiver 本质上是非常简单的一个通用思路，就是把对象值或地址作为第一参数传入函数；
- 函数参数压栈方式从前往后（可以调试看下）；
- 对象值作为 receiver 的时候，涉及到一次值拷贝；
- golang 对于值做 receiver 的函数定义，会根据现实需要情况可能会生成了两个函数，一个值版本，一个指针版本（思考：什么是“需要情况”？就是有 `interface` 的场景 ）；
- 空结构体在编译期间就能识别出来的场景，编译器会对既定的事实，可以做特殊的代码生成；

可以这么说，编译期间，关于空结构体的参数基本都能确定，那么代码生成的时候，就可以生成对应的静态代码。

程序 debug 技巧和工具介绍可以翻看：[golang 调试分析的高阶技巧](http://mp.weixin.qq.com/s?__biz=MzU2MDcwNTg3OA==&mid=2247484104&idx=1&sn=082bfb51db063d80aaa1ff2fb05bcae1&chksm=fc02baf1cb7533e767bc7c39dcc4178157dc36ad62efc90c0e084d2f13f86b571a00bbff0a8e&scene=21#wechat_redirect)。



### **空结构体的应用场景**

一般我们用在用户不关注值内容的情况下，只是作为一个信号或一个占位符来使用。

- 基于map实现集合功能。

- 与channel组合使用，实现一个信号

- ### `slice` & `struct{}`

基于map实现集合功能就是我们开头提到的。使用空结构体**不占用存储空间外，还有一个语义上的原因**。例如：

```
var CanSkipFuncs = map[string]bool{
    "Email":   true,
    "IP":      true,
    "Mobile":  true,
    "Tel":     false,
    "Phone":   false,
    "ZipCode": false,
}
```

我们这里将空结构体类型更换成布尔类型。首先，声明下，CanSkipFuncs集合代表的是所有要跳过的函数。所以这里的值设置成true还是false是没有任何影响的。

那么当阅读或review代码的时候，很有可能带来疑惑，对于值所表达的意图就有所怀疑，增加了理解代码的难度。就会理解成当值为true时会执行一个分支，当值为false时会执行另一段逻辑。

而相比使用一个空结构体strcut{}理解起来会更容易，一看空结构体struct{}就知道要表达的意思是不需要关心值是什么，只需要关心键值即可。

我们再来看下和channel组合使用的例子。在etcd项目中，就有通过往channel中写入一个空结构体作为信号的，源码位于/etcd/server/auth/simple_token.go中，如下：

```
func (tm *simpleTokenTTLKeeper) stop() {
    select {
    case tm.stopc <- struct{}{}:
    case <-tm.donec:
    }
    <-tm.donec
}
```

还有一种是基于缓冲channel实现并发限速。如下：

```
var limit = make(chan struct{}, 3)

func main() {
    // …………
    for _, w := range work {
        go func() {
            limit <- struct{}{}
            w()
            <-limit
        }()
    }
    // …………
}
```

还有一直基于`slice` & `struct{}`

形式上，`slice` 也结合 `struct{}` 。

```
s := make([]struct{}, 100)
```

我们创建一个数组，无论分配多大，所占内存只有 24 字节（addr, len, cap），但实话说，这种用法没啥实用价值。

创建 slice 其实调用的是 `makeslice` 来分配内存，其中是调用 `malllocgc` ，而 `mallocgc` 我们知道在分配 size 为 0 的内存则是直接返回 `zerobase` 的地址而已。

而 slice 在扩展的时候在遇到这种 size 为 0 的时候，也是直接返回 zerobase 的地址。

```
func growslice(et *_type, old slice, cap int) slice {
    // 如果元素的 size 为 0，那么还是直接赋值了 zerobase 的地址；
    if et.size == 0 {
        return slice{unsafe.Pointer(&zerobase), old.len, cap}
    }
}
```



### 总结

空结构体是一种不包含任何字段的结构体类型，不仅具有结构体类型的一切属性，而且该结构体类型占用的空间为0。

1. 空结构体也是结构体，只是 size 为 0 的类型而已；
2. 所有的空结构体都有一个共同的地址：`zerobase` 的地址；
3. 空结构体可以作为 receiver ，receiver 是空结构体作为值的时候，编译器其实直接忽略了第一个参数的传递，编译器在编译期间就能确认生成对应的代码；
4. `map` 和 `struct{}` 结合使用常常用来节省一点点内存，使用的场景一般用来判断 key 存在于 `map`；
5. `chan` 和 `struct{}` 结合使用是一般用于信号同步的场景，用意并不是节省内存，而是我们真的并不关心 chan 元素的值；
6. `slice` 和 `struct{}` 结合好像真的没啥用

