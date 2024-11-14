本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 序

在上节中我们聊了些golang底层库开发了一些并发安全下的工具类，[golang 底层基石unsafe](https://mp.weixin.qq.com/s/wUuPr3gcj8-AmFGfnk77Mg)、[golang 一文搞懂高性能从lock free到锁](https://mp.weixin.qq.com/s/-R4PxUKrbpTy7_ebai5Iuw)。

那么本节开始聊聊golang常用的并发安全sync.Map的演变到深入理解。



### 原生map

> map是一系列键值对的存储数据结构，提供快速的读写操作。map数据结构的核心是如何实现一个高效的hash寻址方式，以便程序能快速的读写数据。目前主流的寻址方式有两种，**开放寻址**和**拉链法**。
>
> 原生map在并发读写的时候，容易造成panic，这是因为原生map并不是线程安全的，对它进行并发读写操作的时候，会导致map里的数据发生错乱，因而导致panic。



#### 例子

```go
package main

import "time"

func main() {
	m := make(map[int]int)
	go func() {
		for {
			m[1] = 1
		}
	}()
	go func() {
		for {
			_, _ = m[1]
		}
	}()
	time.Sleep(2 * time.Second)
}
// 返回结果：fatal error: concurrent map read and map write
```



#### 为什么它是不安全的？



#### 源码解析

go底层map到底怎么存储呢?

接下来我们一探究竟。map的源码位于 src/runtime/map.go中 ，map同样也是数组存储的的，每个数组下标处存储的是一个bucket,这个bucket的类型见下面代码，每个bucket中可以存储8个kv键值对，当每个bucket存储的kv对到达8个之后，会通过overflow指针指向一个新的bucket，从而形成一个链表,看bmap的结构，我想大家应该很纳闷，没看见kv的结构和overflow指针啊，事实上，这两个结构体并没有显示定义，是通过指针运算进行访问的。



我们先看看数据结构

```go
//bucket结构体定义 b就是bucket
type bmap{
    // tophash generally contains the top byte of the hash value
    // for each key  in this bucket. If tophash[0] < minTopHash,
    // tophash[0] is a bucket               evacuation state instead.
    //翻译：top hash通常包含该bucket中每个键的hash值的高八位。
    如果tophash[0]小于mintophash，则tophash[0]为桶疏散状态    //bucketCnt 的初始值是8
    tophash [bucketCnt]uint8
    // Followed by bucketCnt keys and then bucketCnt values.
    // NOTE: packing all the keys together and then all the values together makes the    // code a bit more complicated than alternating key/value/key/value/... but it allows    // us to eliminate padding which would be needed for, e.g., map[int64]int8.// Followed by an overflow pointer.    //翻译：接下来是bucketcnt键，然后是bucketcnt值。
    注意：将所有键打包在一起，然后将所有值打包在一起，    使得代码比交替键/值/键/值/更复杂。但它允许//我们消除可能需要的填充，    例如map[int64]int8./后面跟一个溢出指针
}
```



https://zhuanlan.zhihu.com/p/706431180

https://www.cnblogs.com/songgj/p/16993883.html

https://www.cnblogs.com/qi66/p/16771543.html

https://zhuanlan.zhihu.com/p/599178236

https://zhuanlan.zhihu.com/p/366986038

