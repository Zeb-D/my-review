本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 序

本篇文章只普及一些基础语法——如何学会最基础的编程。

涵盖范围为：变量、常量、条件与switch、循环、指针、Range、函数、闭包、方法、接口、范型、错误、Goroutines、Select 与channel、defer、panic与recover。

注意：本代码需要大家自己搭建好环境，或者 随便找个网页运行`https://toolin.cn/run-go` `http://go.jsrun.net/`



### 变量

声明局部变量、全局变量方式：

1、使用 关键字 var 定义变量，自动初始化为 0 值；

2、除了基本数据类型外，可以使用new 声明；

```golang
# 方式一 ：
func variable() {
    var a int
    var s string
}

# 方式二 ：
func variableInitialValue() {
    var a, b int = 3, 4
    var s string = "abc"
}

# 方式三 ： 推荐 
func variableTypeDeduction() {     
    a, b, c, s , s1 := 3, 4.12, "abc", true, [2]int{1, 2}
}

# 方式四 ：
var s , a = "abc" , 1

# 全局变量
var (
    a = 3
    b = 4
)
```



变量类型：

```
# bool , string
# (u)int, (u)int8  (u)int16 (u)int32 (u)int32  (u)int64  uintptr
# byte , rune
# float32, float64, complex64, complex128


# 字符类型是 uint8


# 复合类型
    array, slice, map, function

# 数据类型的分类：
            值类型：
                int，float，bool，string，array，struct
            引用类型
                slice，map，function
```



### 常量

```
# const
    const a, b = 3, 4
    const filename = "abc.txt"
    
    或者
    const (
        a = 3
        b = 4
        str = "name"
    )  
    
# 在常量组中，如不提供类型和初始化值，那么视为与上一常量相同
    const (
        s = "abc"
        x             // x = "abc"
    )
```



### 条件

```
# if
func bounded(v int) int {
    if v > 100 {
        return 100
    } else if v < 0 {
        return 0
    }
    return v
}

# switch
    switch 会自动 break, 除非使用 fallthrough
    
    func eval(a, b int, op string) int {
      var result int
      switch op {
      case "+":
        result = a + b
      case "-":
        result = a - b
      case "*":
        result = a * b
      case "/":
        result = a / b
      }
      return result
    }
    // 调用
    fmt.Printf("%d", eval(1, 3, "+"))
    
 
 func mainSwitch() {
	i := new(int)
	fmt.Fscan(os.Stdin, i)
	//基础模式，可以没有default
	switch *i {
	case 1:
		fmt.Println("one")
	case 2:
		fmt.Println("two")
	case 3:
		fmt.Println("three")
	}
	//同一个case里可以有多个表达式，用逗号隔开
	switch time.Now().Weekday() {
	case time.Sunday, time.Saturday:
		fmt.Println("enjoy weekend")
	default:
		fmt.Println("weekday")
	}
	//Switch不带表达式，类似与if-else
	t := time.Now()
	switch {
	case t.Hour() < 12:
		fmt.Println("上午", t.Hour(), t.Minute())
	case t.Hour() > 12:
		fmt.Println("下午", t.Hour(), t.Minute())
	}
	//小技巧，用于判断接口的类型
	whoami := func(i interface{}) {
		switch t := i.(type) {
		case int:
			fmt.Println("i am int")
		case string:
			fmt.Println("i am string")
		default:
			fmt.Println("i don't know", t)
		}
	}
	whoami(true)
}
 
```



### Range

```
func mainRange() {
	//遍历数组和切片
	nums := []int{1, 2, 3}
	for i, v := range nums {
		fmt.Println("i, v: ", i, v)
	}
	//还可以不要索引值
	for _, v := range nums {
		fmt.Println("v: ", v)
	}
	//遍历map
	maps := map[string]int{"yuyang": 33, "mingze": 7, "mingyue": 1}
	for k, v := range maps {
		fmt.Println("k, v: ", k, v)
	}
	//返回值只写一个时，为key的值
	for v := range maps {
		fmt.Println("v maps[v]: ", v, maps[v])
	}
	//遍历string，返回值为索引值和rune值
	for i, v := range "yuyang" {
		fmt.Printf("i, v: %v\t%-3v\t%v\n", i, v, string(v))
	}
}
```



### 循环

```
# for

func loopAdd(n int) int {
    sum := 0
    for i := 0; i < 100; i++ {
        sum += i
    }
    return sum
}

// 调用
fmt.Printf("%d", loopAdd(100))

// 无限循环 //相当于for true
for {
}

// for 循环退出
i := 0
for {
    i++
    if i == 5 {
        break
    } else {
        fmt.Println("i = ", i)
        contiune
    }
}
    
# 没有 while
```



### 函数

```
# 可以返回多个值，包括0个值
func div(a, b int) (q, r int) {
    return a / b, a % b
}

// 调用
q, r := div(19, 5)
fmt.Printf("q = %d, r = %d", q, r)


// 在函数中传递任意多个数字
func add(merbers ...int) int {
    sum := 0
    for i := range merbers {
        sum += merbers[i]
    }
    return sum
}

fmt.Printf("q = %d", add(1, 2, 3))
```



### 指针

```
# pointer，指针
            概念：存储了另一个变量的内存地址的变量。
# 指针的类型：*int,*float,*string,*array,*struct
        指针中存储的数据的类型:int,float,string,array,struct

# %T 指针的类型
# %p 地址

a := 10
fmt.Println("a的数值是：", a)     //10
fmt.Printf("a的类型是：%T\n", a)  //int
fmt.Printf("a的地址是：%p\n", &a) //0xc42001c0c8

func main() {
   a := 10
   var p1 *int
   p1 = &a
   var p2 **int
   fmt.Println(p2)
   p2 = &p1
   fmt.Println("p2的数值：", p2)              //0xc0000ac018
   fmt.Println("p2的数值，p1的地址，对应的数据：", *p2) //p2的数值，p1的地址，对应的数据： 0xc0000b2008
   fmt.Println("p2中存储的地址对应的数值，在对应的数值：", **p2) //p2中存储的地址对应的数值，在对应的数值： 10
}


# 深浅拷贝
    深拷贝：拷贝数据的副本。对原始数据没有影响
                值类型的数据，默认都是深拷贝
                    int，float，string，bool，array，struct
    浅拷贝：拷贝的是数据的地址。
                引用类型的数据，默认都是浅拷贝
                    slice，map，function

# 数组的深拷贝
    arr1 := [4]int{1, 2, 3, 4}
    arr2 := arr1
    
# 数组的浅拷贝
    arr3 := &arr1
    (*arr3)[0] = 100
    

func fun4(p2 *[4]int) {
    fmt.Println("函数中数组：", p2)
    p2[0] = 200
    fmt.Println("函数中数据：", p2)
}

func fun3(arr2 [4]int) { //arr2 = arr1,数组是值数据
    fmt.Println("函数中数组：", arr2) //[1,2,3,4]
    arr2[0] = 100
    fmt.Println("函数中数组：", arr2) //[100,2,3,4]
}


# 数组指针：首先是一个指针，一个数组的地址。
    存储的是数组的地址
    *[4]int
    
# 指针数组：首先是一个数组，存储的数据类型是指针
    [4]*Type
    *[5]float64,指针，一个存储了5个浮点数据的数组的指针
    *[3]string，指针，一个数组的指针，数组中存储了3个字符串
    [5]*float64，数组，存储5个float的数据的地址
    
    *[5]*float64，指针，一个数组的指针，数组中存储5个float的数据的地址
    *[3]*string，指针，数组的指针，3个字符串的地址
    **[4]string，指针，存储了4个字符串数据的数组的指针的指针
    **[4]*string，指针，存储了4个字符串的数据的地址的数组，的指针的指针。。
# eg : 数组指针
    a := 1
    b := 2
    c := 3
    d := 4
    arr2 := [4]int{a, b, c, d}
    arr3 := [4]*int{&a, &b, &c, &d}
    
    fmt.Println(*arr3[0])

# eg : 指针数组
    arr1 := [4]int{1, 2, 3, 4}
    var p1 *[4]int
    p1 = &arr1
    (*p1)[0] = 100 //简写
    p1[0] = 200

# 指针和函数
    函数指针： 一个指针，指向了一个函数的指针。

    因为go语言中，function，默认看作一个指针，没有*。
            slice，map，function是引用类型的数据，存储的就是数据的地址。看作一个默认的指针，没有*

    指针函数：一个函数，该函数的返回值是一个指针。
    
# eg
func fun2() [4]int { //普通函数
    arr := [4]int{1, 2, 3, 4}
    return arr
}

func fun3() *[4]int { //指针函数
    arr := [4]int{1, 2, 3, 4}
    return &arr
}

  arr1 := fun2()
  arr2 := fun2()
  fmt.Printf("arr1的类型：%T,地址：%p,数值：%v\n", arr1, &arr1, arr1)
  fmt.Printf("arr2的类型：%T,地址：%p,数值：%v\n", arr2, &arr2, arr2)
  p3 := fun3()
  p4 := fun3()
  fmt.Printf("p3：%T,地址：%p,数值：%v\n", p3, p3, p3)
  fmt.Printf("p4：%T,地址：%p,数值：%v\n", p4, p4, p4)
```



### 闭包

闭包（Closure）是一种引用了外部变量的函数。闭包可以访问定义在函数体外部的变量和参数，并且可以将这些变量和参数永久保存在自己的函数体内，不受外部影响。

如果一个函数返回的是一个闭包（函数），那么该函数内部定义的变量会保存在闭包中，即使该函数已经返回。闭包中的变量也可以在多个闭包之间共享，即可被多个闭包访问和修改。

```
package main

import "fmt"

func intSeq() func() int {
	i := 0
	return func() int {
		fmt.Println("i: ", i)
		i++
		return i
	}
}
func mainClosure() {
	nextInt := intSeq()
	fmt.Printf("%T\n", nextInt)
	fmt.Printf("%v\n", nextInt)
	fmt.Printf("%v\n", nextInt())
	fmt.Printf("%v\n", nextInt())
	fmt.Printf("%v\n", nextInt())
	nextInt2 := intSeq()
	fmt.Printf("%v\n", nextInt2())
	fmt.Printf("%v\n", nextInt2())
	fmt.Printf("%v\n", nextInt())
}
```



### 递归

递归函数时不断地调用自己，直到达到最基本的情况为止。在每此调用的过程中需要处理相应的逻辑以匹配目标。

```
package main

import "fmt"

// 基本玩法
func rec(i int) int {
	if i == 0 {
		return 0
	}
	return i + rec(i-1)
}

func mainRecursion() {
	sum := rec(5)
	fmt.Println("sum: ", sum)
	//用闭包做递归，需要先在闭包之外用var声明闭包
	var fib func(n int) int
	fib = func(n int) int {
		if n < 2 {
			return n
		}
		return fib(n-1) + fib(n-2)
	}
	fmt.Println("fib(5): ", fib(5))
}
```



### 方法

方法和函数、闭包的区别在于方法定义需要一个结构体。

另外方法可以支持普通结构体，也支持指针型结构体，即方法前面是否有带型号。

问个问题：这两者有哪些区别呢？

```go
type rect struct {
	width, height int
}

func (r rect) area() int {
	return r.width * r.height
}

func (r *rect) perim() int {
	return (r.height + r.width) * 2
}

func (r *rect) change() {
	r.height *= 2
	r.width *= 2
}

func (r rect) vchange() {
	r.width += 2
	r.height += 2
}

func mainMethod() {
	r := rect{1, 2}
	fmt.Println("面积：", r.area())
	fmt.Println("周长：", r.perim())
	//指针自动转换
	rp := &r
	fmt.Println("面积：", rp.area())
	fmt.Println("周长：", rp.perim())
	//应用传递
	r.change()
	fmt.Println("宽：", r.width)
	fmt.Println("长：", r.height)
	//值传递
	r.vchange()
	fmt.Println("宽：", r.width)
	fmt.Println("长：", r.height)

}
```



### 接口

**接口的定义**

```text
type my_interface interface {
     getValue(a type) type
}
```

所有**type开头**的，都是一种类型。后面是**接口名**和**关键字interface**，然后大括号里是接口的**方法签名**，每个接口都可以有0到N个方法。0个方法的就是空接口。



**接口的理解**

接口可以理解为是一种抽象的类型。这个类型有点特殊，即所有实现了接口约定的方法的对象，都可以成为这种类型（同时各个对象还保持有自己的类型）。某个对象成为接口的类型后，这个对象就可以赋值给这个接口，同样可以作为参数传递给以这种接口为入参的函数。



**关于方法接受者的理解**

方法的接受者分两种：值接收者（v type）、指针接收者（v *type）。方法的描述以什么样的方式给到他的对象，值接收可以理解为以类似变量名一样的方式给到对象，指针接收可以理解为把保存方法描述的地址给到对象（而根据地址是可以推导出值的）。

所以，在给接口赋值的时候，指针接收者的对象不能赋值给接口，必须是指针接收者的对象取地址才能赋值给接口。

```text
package main

import "fmt"

type sayer interface {
	say()
}

type dog struct{}

func (d dog) say() {
	fmt.Println("汪汪汪----")
}

type cat struct{}

func (c *cat) say() {
	fmt.Println("喵喵喵----")
}

func main() {
	var s sayer
        // 注意这里要给对象取地址
	c1 := &cat{}
	s = c1
	s.say()
	d1 := dog{}
	s = d1
	s.say()
}
```



接口的实现

**函数的参数：**创建一个函数，入参是接口类型，然后所有实现了接口的对象都可以作为入参传入这个函数，就实现了通用的能力。

- 实现接口，需要对象实现了接口里的所有方法（方法签名一样）
- 方法签名和函数签名是一个意思：函数名、入参、返回值一致的表示签名一致。
- 下面用两个对象都来实现一下上面的接口

```go
package main

import (
	"fmt"
	"math"
)

// 定义一个接口，有两个方法
type geometry interface {
	area() float64
	perim() float64
}


type rect1 struct {
	width, height float64
}

func (r rect1) area() float64 {
	return r.width * r.height
}
func (r rect1) perim() float64 {
	return 2 * (r.width + r.height)
}

type circle struct {
	radius float64
}

func (c circle) area() float64 {
	return math.Pi * c.radius * c.radius
}
func (c circle) perim() float64 {
	return 2 * math.Pi * c.radius
}

// 创建一个求所有几何形状面积和周长的行数，入参用的是接口类型
func measure(g geometry) {
	fmt.Println(g)
	fmt.Println(g.area())
	fmt.Println(g.perim())
}

func mainInterface() {
	r1 := rect1{1, 2}
	measure(r1)
	c1 := circle{3}
	measure(c1)
}
```



**空接口用法**

注意`interface` 和 `interface{}` 是有区别的，`interface`是名词上的接口，而`interface{}` 是个空接口（类似java的Object 万物的基类，或者rust 的any 对象）。

1、作为函数的入参，可以接收任意类型

2、作为字典的值

值用空接口，则可以是任意类型。

```text
package main

import "fmt"

func main() {
	m := map[string]interface{}{
		"name":   "yy",
		"Age":    33,
		"worker": true,
	}
	fmt.Println(m)
}
```



### 错误

```
import (
	"errors"
	"fmt"
)

func f1(arg int) (int, error) {
	if arg == 42 {
		return -1, errors.New("can't work wtih 42")
	}
	return arg + 3, nil
}

// 创建一个自己的error对象，定义自己关心的成员
type myerror struct {
	arg  int
	prob string
}

// 实现error接口的Error()方法即可以实现error接口了，在方法逻辑里包装自己想要的逻辑
func (m *myerror) Error() string {
	return fmt.Sprintf("%v -- %v", m.arg, m.prob)
}

// 用自定义的myerror作为error接口返回
func f2(arg int) (int, error) {
	if arg == 42 {
		return -1, &myerror{arg: arg, prob: "myerror"}
	}
	return arg + 3, nil
}
func mainError() {
	res, err := f1(42)
	fmt.Printf("f1-res: %v\n", res)
	fmt.Printf("f1-err: %v\n", err)
	//断言一下看看类型
	if b, ok := err.(error); ok {
		fmt.Println("f1-b: ", b)
		fmt.Println("f1-ok: ", ok)
	}

	res, err = f2(42)
	fmt.Printf("res: %v\n", res)
	fmt.Printf("err: %v\n", err)
	//断言一下看看类型
	b, ok := err.(*myerror)
	fmt.Println("b: ", b)
	fmt.Println("ok: ", ok)
}
```



### Goroutines

```go
func f(from string) {
	for i := 0; i < 3; i++ {
		fmt.Println(from, ":", i)
	}
}

// waitGroup在waitgroup里练习
func mainGoroutine() {
	//正常调用函数的方式
	f("yuyang")
	//并发调用，注意：主进程（main）不会主动等待并发的携程结束后再结束的，可以手动sleep等一下携程。
	go f("zhangsan")
	time.Sleep(time.Second * 1)
	//还可以用匿名函数
	go func(from string) {
		for i := 0; i < 3; i++ {
			fmt.Println(from, ":", i)
		}
	}("lisi")
	time.Sleep(time.Second * 1)
	fmt.Println("main over")
}
```



**waitgroup 代码说明**

- 少数携程之间保持同步，可以使用channel的阻塞能力实现，
- 但是携程多了以后，这个关系就不太好维护了。
- 多携程直接的同步可以用wg sync.WaitGroup实现

*注意：wg是指针类型，否则值传递就不对了，主函数的wg的值copy过来，就没有给主函数原本的wg做减法了*

```go
import (
	"fmt"
	"sync"
	"time"
)

// 注意wg是指针类型，否则值传递就不对了，主函数的wg的值copy过来，就没有给主函数原本的wg做减法了
func work(wg *sync.WaitGroup) {
	//没有参数，表示wg计数器减1,如果要减2用【-2】参数
	defer wg.Done()
	fmt.Println("work sleep 1s")
	time.Sleep(time.Second * 1)
}
func mainWaitGroup() {
	//初始化一个wg计数器
	wg := sync.WaitGroup{}
	//表示wg计数加1（参数是几，就表示加几）
	wg.Add(1)
	go work(&wg)
	//一定要保证每起一个携程就要有一个wg.Add，这是成对存在的。
	wg.Add(1)
	go func(wg *sync.WaitGroup) {
		//wg.Done()
		defer wg.Done()
		fmt.Println("anonymous sleep 2s")
		time.Sleep(time.Second * 2)
	}(&wg)
	//wg计数器等待归零
	wg.Wait()
	fmt.Println("main over")
}
```



**goroute的原子操作**

如果多个 goroutine 同时调用 atomic.AddInt64() 函数，它会确保这些 goroutine 操作的整数值是原子性的。

这意味着任何时刻只有一个 goroutine 可以操作该变量。这个函数非常适合在多个 goroutine 中共享整数值时使用，比如计数器。

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func mainAtomic() {
	var ops int64
	var wg sync.WaitGroup
	for i := 0; i < 50; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for c := 0; c < 1000; c++ {
				atomic.AddInt64(&ops, 1)
			}
		}()
	}
	wg.Wait()
	fmt.Println("ops: ", ops)
}
```



### Select 与channel

select case专门用于管道和goroutine

```go
package main

import (
	"fmt"
	"time"
)

func main1() {
	c1 := make(chan string, 1)
	c2 := make(chan string, 1)
	go func() {
		fmt.Println("c1 sleep ...")
		time.Sleep(time.Second * 9)
		c1 <- "yu"
	}()
	go func() {
		fmt.Println("c2 sleep ...")
		time.Sleep(time.Second * 10)
		c2 <- "yang" //放进去
	}()
	ticker := time.After(time.Second * 3)
	//for i := 0; i < 12; i++ {  //如果有多个case要等待，可以用循环
	select {
	//每个case必须是一个对管道的操作
	case res := <-c1: //阻塞式取出
		fmt.Println("c1 wake up", res)
	case res := <-c2:
		fmt.Println("c2 wake up", res)
	case <-ticker:
		//case <-time.After(time.Second * 1):
		fmt.Println("time out")
		//带上default就表示不阻塞了，上面的情况都没有触发的时候，就立即执行default的逻辑，而不是阻塞等待
		//default:
		//	fmt.Println("default")
	}
	//}
	jobs := make(chan int, 5)
	done := make(chan bool)
	for i := 0; i < 3; i++ {
		jobs <- i
	}
	go func() {
		for {
			fmt.Println("go func job: ", <-jobs)
		}
		done <- true
	}()
	<-done
}
```



### panic 与 recover

注意:通过 panic() 函数引发的运行时恐慌只能在 defer 函数中使用 recover() 函数进行捕获。
因此，在使用 panic() 函数时，我们需要结合使用 defer 和 recover() 函数来处理运行时恐慌并确保程序的安全性。
程序在运行到 panic() 函数时立即终止，它后面的语句都不会执行，包括第三个 fmt.Println() 语句。

```text
func div(i, j int) int {
	if j == 0 {
		panic("分母不能为0")
	}
	return i / j
}

func mainPanic() {
	defer func() {
		if res := recover(); res != nil {
			fmt.Println("frome recover ", res)
		}
	}()
	fmt.Println(div(4, 2))
	fmt.Println(div(4, 0))
	fmt.Println("main over")
}
```



### 范型

范型（也称为参数化类型）是一个复杂的特性，它允许我们创建可以用不同类型实例化的模板函数或数据结构。

以下是一个简单的使用泛型库实现的范型示例，它定义了一个函数，可以接受任何类型的切片，只要这些类型都实现了`constraints.Ordered`接口（这意味着它们可以与数字进行比较）。

```
package main
 
import (
    "fmt"
    "golang.org/x/exp/constraints"
)
 
// 使用constraints.Ordered范型实现一个简单的排序函数
func Sorted[T constraints.Ordered](a []T) []T {
    result := append([]T(nil), a...) // 复制切片以避免修改原始切片
    sort.Slice(result, func(i, j int) bool {
        return result[i] < result[j]
    })
    return result
}
 
func main() {
    // 使用int类型的切片
    intSlice := []int{5, 1, 3, 2, 4}
    sortedIntSlice := Sorted(intSlice)
    fmt.Println(sortedIntSlice) // 输出: [1 2 3 4 5]
 
    // 使用string类型的切片
    stringSlice := []string{"e", "a", "c", "b", "d"}
    sortedStringSlice := Sorted(stringSlice)
    fmt.Println(sortedStringSlice) // 输出: [a b c d e]
}
```

