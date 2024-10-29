本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 序

上周写了篇[常见语法入门](https://mp.weixin.qq.com/s/iweleV9IwlPcyTwHm75tEg)，在平时敲代码也有一些常规性常识，这些常识没涉及到底层原理（要是原理篇没几十篇都写不完），何谓常识，在我理解看来类似小学的乘法表一样，是个入门；

但可能大家觉得这个比喻不太好，入门的科普性常识这些懂了，就意味着你是入门了，而且入门占用时间很短，也不会像市面上那么多文章动不动给你来个演示、原理，这些对入门来说太枯燥了，在如今快餐文化中，本文限定的范围对入门是合理的。





### 基础问题常识

#### 1、golang 中 make 和 new 的区别

**共同点：** 给变量分配内存

**不同点：**

1）作用变量类型不同，new给string,int和数组分配内存，make给切片，map，channel分配内存；

2）返回类型不一样，new返回指向变量的指针，make返回变量本身；

3）new 分配的空间被清零。make 分配空间后，会进行初始化；

4)  new是在heap（堆），make是在stack（栈），创建make函数时会栈会开辟一块栈帧，栈帧里面有栈基指针和栈顶指针，分别记录栈帧的空间，随着函数的执行完毕，栈里的栈帧会自动清空，这就是和new（）的本质区别，堆里面的不会。



#### **2、数组和切片的区别**

**相同点：**

1)只能存储一组相同类型的数据结构

2)都是通过下标来访问，并且有容量长度，长度通过 len 获取，容量通过 cap 获取

**区别：**

1）数组是定长，访问和复制不能超过数组定义的长度，否则就会下标越界，切片长度和容量可以自动扩容

2）数组是值类型，切片是引用类型，每个切片都引用了一个底层数组，切片本身不能存储任何数据，都是这底层数组存储数据，所以修改切片的时候修改的是底层数组中的数据。切片一旦扩容，指向一个新的底层数组，内存地址也就随之改变

**简洁的回答：**

1）定义方式不一样 

2）初始化方式不一样，数组需要指定大小，大小不改变 

3）在函数传递中，数组切片都是值传递。

**数组的定义**

var a1 [3]int

var a2 [...]int{1,2,3}

**切片的定义**

var a1 []int

var a2 :=make([]int,3,5)

**数组的初始化**

a1 := [...]int{1,2,3}

a2 := [5]int{1,2,3}

**切片的初始化**

b:= make([]int,3,5)



#### **3、for range 的时候它的地址会发生变化么？**

在 for a,b := range c 遍历中， a 和 b 在内存中只会存在一份，即之后每次循环时遍历到的数据都是以值覆盖的方式赋给 a 和 b，a，b 的内存地址始终不变。

由于有这个特性，for 循环里面如果开协程，不要直接把 a 或者 b 的地址传给协程。

解决办法：在每次循环时，创建一个临时变量。



#### **4、go defer，多个 defer 的顺序，defer 在什么时机会修改返回值？**

作用：defer延迟函数，释放资源，收尾工作；如释放锁，关闭文件，关闭链接；捕获panic;

避坑指南：defer函数紧跟在资源打开后面，否则defer可能得不到执行，导致内存泄露。

多个 defer 调用顺序是 LIFO（后入先出），defer后的操作可以理解为压入栈中

defer，return，return value（函数返回值） 执行顺序：首先return，其次return value，最后defer。defer可以修改函数最终返回值，修改时机：**有名返回值或者函数返回指针**



#### **5、uint 类型溢出问题**

超过最大存储值如uint8最大是255

var a uint8 =255

var b uint8 =1

a+b = 0总之类型溢出会出现难以意料的事



#### **6、能介绍下 rune 类型吗？**

相当int32

golang中的字符串底层实现是通过byte数组的，中文字符在unicode下占2个字节，在utf-8编码下占3个字节，而golang默认编码正好是utf-8；

byte 等同于int8，常用来处理ascii字符，默认情况下中文就需要3个byte；

rune 等同于int32,常用来处理unicode或utf-8字符；



#### **7、 golang 中解析 tag 是怎么实现的？反射原理是什么？**

Go 中解析的 tag 是通过反射实现的，反射是指计算机程序在运行时（Run time）可以访问、检测和修改它本身状态或行为的一种能力或动态知道给定数据对象的类型和结构，并有机会修改它。

反射将接口变量转换成反射对象 Type 和 Value；

反射可以通过反射对象 Value 还原成原先的接口变量；

反射可以用来修改一个变量的值，前提是这个值可以被修改；

tag是啥:结构体支持标记，name string `json:name-field` 就是 `json:name-field` 这部分

**gorm json yaml gRPC protobuf gin.Bind()都是通过反射来实现的**



#### **8、调用函数传入结构体时，应该传值还是指针**

Go 的函数参数传递都是值传递。

所谓值传递：指在调用函数时将实际参数复制一份传递到函数中，这样在函数中如果对参数进行修改，将不会影响到实际参数。

参数传递还有引用传递，所谓引用传递是指在调用函数时将实际参数的地址传递到函数中，那么在函数中对参数所进行的修改，将影响到实际参数

因为 Go 里面的 map，slice，chan 是引用类型。变量区分值类型和引用类型。

所谓值类型：变量和变量的值存在同一个位置。

所谓引用类型：变量和变量的值是不同的位置，变量的值存储的是对值的引用。

但并不是 map，slice，chan 的所有的变量在函数内都能被修改，不同数据类型的底层存储结构和实现可能不太一样，情况也就不一样。传递是值类型，但底层有指针，如果碰上了append行为，它们直接之间不是共享的；



#### **9、讲讲 Go 的 slice 底层数据结构和一些特性？**

Go 的 slice 底层数据结构是由一个 array 指针指向底层数组，len 表示切片长度，cap 表示切片容量。slice 的主要实现是扩容。

对于 append 向 slice 添加元素时，假如 slice 容量够用，则追加新元素进去，slice.len++，返回原来的 slice。

当原容量不够，则 slice 先扩容，扩容之后 slice 得到新的 slice，将元素追加进新的 slice，slice.len++，返回新的 slice。

对于切片的扩容规则：当切片比较小时（容量小于 1024），则采用较大的扩容倍速进行扩容（新的扩容会是原来的 2 倍），避免频繁扩容，从而减少内存分配的次数和数据拷贝的代价。

当切片较大的时（原来的 slice 的容量大于或者等于 1024），采用较小的扩容倍速（新的扩容将扩大大于或者等于原来 1.25 倍），主要避免空间浪费，网上其实很多总结的是 1.25 倍，那是在不考虑内存对齐的情况下，实际上还要考虑内存对齐，扩容是大于或者等于 1.25 倍。

（关于刚才问的 slice 为什么传到函数内可能被修改，如果 slice 在函数内没有出现扩容，函数外和函数内 slice 变量指向是同一个数组，则函数内复制的 slice 变量值出现更改，函数外这个 slice 变量值也会被修改。如果 slice 在函数内出现扩容，则函数内变量的值会新生成一个数组（也就是新的 slice，而函数外的 slice 指向的还是原来的 slice，则函数内的修改不会影响函数外的 slice。）



#### **10、讲讲 Go 的 select 一些特性**

1）select 操作至少要有一个 case 语句，出现读写 nil 的 channel 该分支会忽略，在 nil 的 channel 上操作则会报错。

2）select 仅支持管道，而且是单协程操作。

3）每个 case 语句仅能处理一个管道，要么读要么写。

4）多个 case 语句的执行顺序是随机的。

5）存在 default 语句，select 将不会阻塞，但是存在 default 会影响性能。



#### **11、讲讲 Go 的 defer 底层数据结构和一些特性？**

每个 defer 语句都对应一个_defer 实例，多个实例使用指针连接起来形成一个单连表，保存在 gotoutine 数据结构中，每次插入_defer 实例，均插入到链表的头部，函数结束再一次从头部取出，从而形成后进先出的效果。

返回值先保存在栈上，然后defer，然后返回函数.

**defer 的规则总结**：

延迟函数的参数是 defer 语句出现的时候就已经确定了的。

延迟函数执行按照后进先出的顺序执行，即先出现的 defer 最后执行。

延迟函数可能操作主函数的返回值。

申请资源后立即使用 defer 关闭资源是个好习惯。



#### 12、各个类型的变量的零值：

- 数值：所有数值类型的零值都是0

  - 整数，零值是0。byte, rune, uintptr也是整数类型，所以零值也是0。
  - 浮点数，零值是0
  - 复数，零值是0+0i
  - 整数类型的变量是可以用`untyped float constant`进行赋值的，只要不损失精度即可。

- bool，零值是false

- 字符串，零值是空串 `""`

- 指针：var a *int，零值是nil

- 切片：var a []int，零值是nil

- map：var a map[string] int，零值是nil

- 函数：var a func(string) int，零值是nil

- 结构体: var instance Struct，结构体里每个field的零值是对应类型的零值

- channel：var a chan int，通道channel，零值是nil

- 接口：var a interface_type，接口interface，零值是nil

  

#### 13、map 使用注意的点，是否并发安全？

map的类型是map[key]，key类型的ke必须是可比较的，通常情况，会选择内建的基本类型，比如整数、字符串做key的类型。如果要使用struct作为key，要保证struct对象在逻辑上是不可变的。在Go语言中，map[key]函数返回结果可以是一个值，也可以是两个值。map是无序的，如果我们想要保证遍历map时元素有序，可以使用辅助的数据结构，例如orderedmap。

**第一，**一定要先初始化，否则写时panic， 如下：

```
var m map[string]string // m := new(map[string]string)
m["a"] = "a" //panic
fmt.Println((m)["a"])
```

**第二，**map类型是容易发生并发访问问题的。不注意就容易发生程序运行时并发读写导致的panic。 Go语言内建的map对象不是线程安全的，并发读写的时候运行时会有检查，遇到并发问题就会导致panic。



#### 14、map 循环是有序的还是无序的？

无序的, map 因扩张⽽重新哈希时，各键值项存储位置都可能会发生改变，顺序自然也没法保证了，所以官方避免大家依赖顺序，直接打乱处理。就是 for range map 在开始处理循环逻辑的时候，就做了随机播种



#### 15、 map 中删除一个 key，它的内存会释放么？

如果删除的元素是值类型，如int，float，bool，string以及数组和struct，map的内存不会自动释放

如果删除的元素是引用类型，如指针，slice，map，chan等，map的内存会自动释放，但释放的内存是子元素应用类型的内存占用

将map设置为nil后，内存被回收。



#### 16、怎么处理对 map 进行并发访问？有没有其他方案？ 区别是什么？

**方式一、使用内置sync.Map**

**方式二、使用读写锁实现并发安全map**



#### 17、 nil map 和空 map 有何不同？

1）可以对未初始化的map进行取值，但取出来的东西是空：

```
var m1 map[string]string //这种方式为nil
fmt.Println(m1["1"])
```

2）不能对未初始化的map进行赋值，这样将会抛出一个异常：

```
var m1 map[string]string
m1["1"] = "1" //panic: assignment to entry in nil map
```

3) 通过fmt打印map时，空map和nil map结果是一样的，都为map[]。所以，这个时候别断定map是空还是nil，而应该通过map == nil来判断。

**nil map 未初始化，空map是长度为空**



#### 18、slices能作为map类型的key吗？

其实是这个问题的变种：golang 哪些类型可以作为map key？

**在golang规范中，可比较的类型都可以作为map key；**这个问题又延伸到在：golang规范中，哪些数据类型可以比较？

**不能作为map key 的类型包括：**

- slices
- maps
- functions

在golang规范中，可比较的类型都可以作为map key，包括：

类型			说明
boolean			 布尔值	
numeric			 数字	包括整型、浮点型，以及复数
string			 字符串	
pointer			 指针	两个指针类型相等，表示两指针指向同一个变量或者同为nil
channel			 通道	两个通道类型相等，表示两个通道是被相同的make调用创建的或者同为nil
interface			 接口	两个接口类型相等，表示两个接口类型 的动态类型 和 动态值都相等 或者 两接口类型 同为 nil
structs、arrays				只包含以上类型元素



### 基础用法

#### 1、slice

长度确定的切片，必须提前定义好长度。

```go
//反例
reslDs := []string{}
for reslD := range data {
	reslDs = append(reslDs, reslD)
}


//正例
reslDs := makel([]string, 0, len(data))
for reslD := range data {
	resiDs = append(reslDs, reslD)
}
```

解释：在高性能场景下，避免频繁数组扩容多次分配内存，也可以使用sync.Pool



#### 2、chan

chan用来跨goroutine通信时，出于最小权限要求，优先转换为单向chan，禁止一个goroutine中，对一个chan既发送又接收。

```
//正例
go func(ch <-chan int) {

}(ch)

//反例
go func(ch chan int) {

}(ch)
```

解释：单向chan智能用于发送或接收数据，这样可以避免在同一个协程出现死锁或者其他问题。同时，单向chan可以提供程序可读性和可靠性，因为明确了chan的使用方式和限制。



#### 3、interface

使用小接口通过组合实现复杂接口，不建议直接定义复杂接口。面向接口编程。

解释：可以提供代码的可读性和可维护性。方便排查问题。



#### 4、errors

自定义错误时，如果不需要format字符串，用errors.New，而不是fmt.Errorf。

解释：使用errors.Newt比fmt.Errorf节省99%以上的时间，

这条建议并不是不让用fmt.Errorf， 该用的时候就用。

打印错读到日志中时，使用%v占位符，博以看到底层错误信息。

Log.lnfof(" Found error. %v " , er)



#### 5、make

make 切片时，注意len 与 cap。

解释: make建切片时需要注意len以及cap这两个参数，因为sice底层添加数据会进行扩容，频繁扩容是会影响代码性能。



#### 6、单元测试

不要再引入 `bou.ke/monkey`，要用 gomonkey 替代原因是monkey的功能单一，gomonkey是monkey的超集。

另外推荐使用 `github.com/stretchr/testify/assert` 的系列方法(如 assertEqual等)，当断言失败时，这个库能打印更有用的输出(帮助快速找出差异)

`github.com/smartystreets/goconvey/convey` 测试用例比较多时，用convey可以方便的组织用例，看起来更整洁。上一篇文章专门讲了[golang 单元测试姿势你学会了吗](https://mp.weixin.qq.com/s?__biz=Mzg5NDc0Mzc4MQ==&mid=2247483929&idx=1&sn=001a623c083dfa6363bd2dcd9d6a3ae1&chksm=c01ba793f76c2e85bdb1f1d2efb0549d34861817061045d69f8b32b3932facafa898f700a1d2#rd)

添加`go test xx -gcfilags=all=- `选项，关闭Go内联，防止打桩失败。



#### 7、import

规范第三方库的版本:

> github.com/golang/mock v1.4.3 //升级版本 v1.4.4 以上
>
> github.com/urfave/di v1.22.4 //升级版本到v1.22.5 以上
>
> gopkgin/yaml2 v2.3.0 /升级版本到 v2.4.0 以上

按照go mod规范，导入包的版本号大于1的，需要在

import时标识版本号 `github.com/agiledragon/gomonkey/v2`，(注:后续代码中使用时还是用gomonkey.前缀)



#### 8、panic

一般的业务逻辑代码中禁用 panic 及内部 panic的函数。



#### 9、time

函数参数中统一使用time.Duration为单位。

常量声明时推荐: const timeout=3*time.Second

原因：更好理解



#### 10、unittest

单元测运行时，开启-race检测，用于发现代码中竞争异常。

不要在buid时开启，会影响生成可执行文件。



#### 11、string

大量的字符串拼接使用strings.Builder。标准库做了优化，性能更好。



#### 12、file

不推荐用 os.OpenFile 方式读写文件，而推荐使用os.WriteFile os.ReadFile 写和读。

解释：s.OpenFile 写的坑比较多，不设置O_TRUNC时，会出现写截断的问题。



#### 13、struct

当struct嵌套了字段，不同类型的字段分布会导致strut 内存大小不一样；

空struct(无任何属性)的优势：

内存地址都一样(程序启动的零地址)也就是说不占内存；

不管struct名称，不影响内存布局；

不允许nil值，加上不占内存，比如条件map；

结合channel + struct实现信号 比interfece 好些；

当slice大小0，分配出来就是空struct地址；