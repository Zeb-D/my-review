本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 什么是sync.Cond

> sync.Cond实现了一个[条件变量]，它是一个或者一组goroutine等待被唤醒的一个条件判断点。每个Cond都有一个关联的Locker L（通常是* Mutex或* RWMutex），与互斥锁和读写锁不同的是，简单的声明无法创建出来一个可用的条件变量，为了得到一个这样的条件变量我们需要使用到NewCond方法。具体实例如下：

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var locker = new(sync.Mutex)
var cond = sync.NewCond(locker)

func main() {
	for i := 0; i < 10; i++ {
		go func(x int) {
			cond.L.Lock()         //获取锁
			defer cond.L.Unlock() //释放锁
			cond.Wait()           //等待通知,阻塞当前goroutine
			fmt.Println(x)
			time.Sleep(time.Second * 1)

		}(i)
	}
	time.Sleep(time.Second * 1)
	fmt.Println("Signal...")
	cond.Signal() // 下发一个通知给已经获取锁的goroutine
	time.Sleep(time.Second * 1)
	cond.Signal() // 3秒之后 下发一个通知给已经获取锁的goroutine
	time.Sleep(time.Second * 3)
	cond.Broadcast() //3秒之后 下发广播给所有等待的goroutine
	fmt.Println("Broadcast...")
	time.Sleep(time.Second * 60)
}

// 结果输出
Signal...
0
3
Broadcast...
1
4
9
2
6
5
7
8
```



### sync.Cond 结构体说明

```go
type Cond struct {
	noCopy noCopy
	L Locker
	notify  notifyList
	checker copyChecker
}
```



#### noCopy

```go
type noCopy struct{}
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

> noCopy可以嵌入到结构中，在第一次使用后不可复制,使用go vet作为检测使用。



#### L

```go
type Locker interface {
	Lock()
	Unlock()
}
```

> L是匿名接口类型，用来继承实现了Locker里面方法的类型，也可以重写里面的方法。在sync.Cond中可以传入一个读写锁或[互斥锁]，当修改条件或者调用wait 方法时需要加锁。



#### notify

```go
type notifyList struct {
	wait uint32
	notify uint32
	lock mutex
	head *sudog
	tail *sudog
}
```

> notify：对应notifyList类型，其数据结构映射到 src/runtime/sema.go 中的notifyList。
> wait：下一个等待goroutine的ticket，是原子的，在锁之外递增。
> notify：下一个通知的goroutine的ticket，在锁之外读取，但只能在持有锁的情况下写入。
> lock：需要传入的锁标记。
> head：基于 *sudog的双向链表的前驱指针。
> tail：基于 *sudog的[双向链表](https://zhida.zhihu.com/search?content_id=169421273&content_type=Article&match_order=2&q=双向链表&zhida_source=entity)的后继指针。



#### checker

```go
type copyChecker uintptr 
```

> copyChecker保留指向自身的指针以检测对象的复制。



### 四种notify处理函数



#### runtime_notifyListAdd

> 在src/runtime/sema.go中被实现 notifyListAdd ；表示将调用者添加到 notify 列表中，以便它可以接收通知。



#### runtime_notifyListWait

> 在src/runtime/sema.go中被实现 notifyListWait ；表示将当前goroutine休眠，等待通知。接到通知以后才会被唤醒。



#### runtime_notifyListNotifyOne

> 在src/runtime/sema.go中被实现 notifyListNotifyOne ；表示发送通知，唤醒 notify 列表中的一个[协程](https://zhida.zhihu.com/search?content_id=169421273&content_type=Article&match_order=1&q=协程&zhida_source=entity)。



#### runtime_notifyListNotifyAll

> 在src/runtime/sema.go中被实现 notifyListNotifyAll ；表示发送通知，唤醒 notify 列表中的所有协程。



### 源码实现

#### NewCond

```go
// 初始化条件变量，l可以是读写锁或者是互斥锁
func NewCond(l Locker) *Cond {
	return &Cond{L: l}
}
```



#### Wait

```go
// 阻塞当前goroutine，并等待条件触发，必须获得锁之后才能调用wait方法
func (c *Cond) Wait() {
        // 检查c是否被复制，如果是则panic
	c.checker.check()
        // 将当前goroutine加入等待队列
	t := runtime_notifyListAdd(&c.notify)
        // 注意这里，必须先解锁，因为 runtime_notifyListWait 要切走 goroutine
	// 所以这里要解锁，要不然其他 goroutine 没法获取到锁了
	c.L.Unlock() 
        // 等待通知队列中的所有的goroutine执行等待唤醒操作
	runtime_notifyListWait(&c.notify, t)
        // 等待唤醒，因此需要再度锁上
	c.L.Lock() 
}
func (c *copyChecker) check() {
	if uintptr(*c) != uintptr(unsafe.Pointer(c)) &&
		!atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c))) &&
		uintptr(*c) != uintptr(unsafe.Pointer(c)) {
		panic("sync.Cond is copied")
	}
}
```



#### Signal

```go
// 唤起一个等待的goroutine
func (c *Cond) Signal() {
	c.checker.check()//检测c是否被复制，如果是则panic
        // 通知等待列表中的一个
	runtime_notifyListNotifyOne(&c.notify)
}
```



#### Broadcast

```go
// 唤醒等待通知队列中的goroutine
func (c *Cond) Broadcast() {
	c.checker.check() //检测c是否被复制，如果是则panic
        // 唤醒等待通知队列中的goroutine
	runtime_notifyListNotifyAll(&c.notify)
}
```