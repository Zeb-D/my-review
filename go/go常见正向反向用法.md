本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 一、背景

用golang已有5年左右，平时多语言开发，加上平时review别人代码有一些总结，因此本文算是对golang平常用法做一些总结（正向、反向）。



### 二、细则

#### 1、slice

长度确定的切片，必须提前定义好长度。

```
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

不要再引入 bou.ke/monkey，要用 gomonkey 替代原因是monkey的功能单一，gomonkey是monkey的超集。

另外推荐使用 github.com/stretchr/testify/assert 的系列方法(如 assertEqual等)，当断言失败时，这个库能打印更有用的输出(帮助快速找出差异)

github.com/smartystreets/goconvey/convey 测试用例比较多时，用convey可以方便的组织用例，看起来更整洁

添加go test xx -gcfilags=all=- 选项，关闭Go内联，防止打桩失败。



#### 7、import

规范第三方库的版本:

github.com/golang/mock v1.4.3 //升级版本 v1.4.4

github.com/urfave/di v1.22.4 //升级版本到v1.22.5

gopkgin/yaml2 v2.3.0 /升级版本到 v2.4.0

按照go mod规范，导入包的版本号大于1的，需要在

import时标识版本号 github.com/agiledragon/gomonkey/v2，(注:后续代码中使用时还是用gomonkey.前缀)



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

内存地址都一样(程序启动的零地址)也就是说不占内存，不管struct名称；

不影响内存布局；不允许nil值，加上不占内存，比如条件map；

结合channel + struct实现信号 比interfece 好些；

当slice大小0，分配出来就是空struct地址；
