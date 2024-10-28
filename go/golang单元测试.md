本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 序

为什么要单独分一篇讲golang的单元测试？

在软件质量这一项中，这块的重视度是等价于代码开发，如果我们严格来讲，程序员代码开发完成这个动作是要包含功能代码开发、对应功能代码的单元测试，它们之间的比重是一样的。

单元测试区别于测试（提测给测试同学），测试是偏功能项上的（比如页面上交互是否OK），单元测试更多的站在代码程度上来更早识别出问题，有些问题不一定测试出来。

当然有些企业的测试又分为系统测试（用户功能性测试）、后台测试（api、链路测试），但大多数的测试资源都偏向于系统测试。



回到golang的测试是提供了两种：`Test`、 `Benchmark` ，从字面上来讲后者是基准测试。



### 命令说明

golang拥有一套单元测试和性能测试系统，仅需要添加很少的代码就可以快速测试一段需求代码。

go test 命令，会自动读取源码目录下面名为 *_test.go 的文件，生成并运行测试用的可执行文件。

 性能测试系统可以给出代码的性能数据，帮助测试者分析性能问题。



介绍`go test` 后面几个常用的参数：

- -bench regexp 执行相应的 benchmarks，例如 -bench=.
- -cover 开启测试覆盖率
- -run regexp 只运行 regexp 匹配的函数，例如 -run=Array 那么就执行包含有 Array 开头的函数
- -v 显示测试的详细命令



### 单测代码规则

规则是golang定义规范的。

单元测试通常放置在与被测试文件同目录下的`_test.go`文件中。测试函数必须以`Test`开头，后接被测试函数名，接受一个`t *testing.T`参数。

```
// example_test.go
package example

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3) // Add 在 example.go 文件中定义
    if result != 5 {
        t.Errorf("Add(2, 3) = %d; want 5", result)
    }
}

// 还有很多func TestXxx 测试用例
```



常用方法

- `t.Error`和`t.Fatal`：报告错误，后者还会终止测试。
- `t.Logf`：记录日志信息。
- `t.Errorf`：当条件不满足时，记录错误并继续执行后续测试。



#### 演示

进入当前文件目录下：

```
$ go test example_test.go
ok          command-line-arguments        0.003s
$ go test -v example_test.go
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
PASS
ok          command-line-arguments        0.004s
```



#### 运行指定单元测试用例

go test 指定文件时默认执行文件内的所有测试用例（包括Benchmark开头）。可以使用`-run`参数选择需要的测试用例单独执行，参考下面的代码。

```go
// example_test.go
package example
import "testing"

func TestAdd(t *testing.T) {
	t.Log("A")
}

func TestAK(t *testing.T) {
	t.Log("AK")
}

func TestB(t *testing.T) {
	t.Log("B")
}

func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}
```



#### 标记单元测试结果

当需要终止当前测试用例时，可以使用 FailNow，参考下面的代码。  测试结果标记

```go
func TestFailNow(t *testing.T) {
	t.FailNow()
}
```

还有一种只标记错误不终止测试的方法，代码如下：

```go
func TestFail(t *testing.T) {
	fmt.Println("before fail")
	t.Fail()
	fmt.Println("after fail")
}
```

测试结果如下：

```
== RUN   TestFail
before fail
after fail
--- FAIL: TestFail (0.00s)
FAIL
exit status 1
FAIL        command-line-arguments        0.002s
```



#### 子测试

这里就不得不提到子测试，即所谓的Subtests，该功能是go语言内置的支持的功能，可以在一个测试用例中，根据测试场景使用`t.Run`创建不同的子测试用例：

```go
func TestMul(t *testing.T) {

    cases := []struct {
        Name           string
        A, B, Expected int
    }{
        {"pos", 2, 3, 6},
        {"neg", 2, -3, -6},
        {"zero", 0, 2, 0},
    }
    
    for _, c := range cases {
        t.Run(c.Name, func(t *testing.T) {
            if ans := Add(c.A, c.B); ans != c.Expected {
                t.Fatalf("%d * %d expected %d, but %d got", c.A, c.B, c.Expected, ans)
            }
        })
    }
}
```





### 基准测试

基准测试用于评估代码性能，函数名以`Benchmark`开头，同样接受一个`*testing.B`参数。

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}
```

`b.N`会自动调整以获得稳定的运行时间。

用如下命令行开启基准测试：

```
$ go test -v -bench=. example_test.go
goos: linux
goarch: amd64
Benchmark_Add-4           20000000         0.33 ns/op
PASS
ok          command-line-arguments        0.700s
```

代码说明如下：

- 第 1 行的`-bench=.`表示运行` example_test.go` 文件里的所有基准测试，和单元测试中的`-run`类似。
- 第 4 行中显示基准测试名称，2000000000 表示测试的次数，也就是 testing.B 结构中提供给程序使用的 N。“0.33 ns/op”表示每一个操作耗费多少时间（纳秒）。

注意：Windows 下使用 go test 命令行时，`-bench=.`应写为`-bench="."`。



#### 基准测试原理

基准测试框架对一个测试用例的默认测试时间是 1 秒。开始测试时，当以 Benchmark 开头的基准测试用例函数返回时还不到 1 秒，那么 testing.B 中的 N 值将按 1、2、5、10、20、50……递增，同时以递增后的值重新调用基准测试用例函数。



#### 自定义测试时间

通过`-benchtime`参数可以自定义测试时间，例如：

```
$ go test -v -bench=. -benchtime=5s example_test.go
goos: linux
goarch: amd64
Benchmark_Add-4           10000000000                 0.33 ns/op
PASS
ok          command-line-arguments        3.380s
```



#### 测试内存

基准测试可以对一段代码可能存在的内存分配进行统计，下面是一段使用字符串格式化的函数，内部会进行一些分配操作。

```go
func Benchmark_Alloc(b *testing.B) {
	for i := 0; i < b.N; i++ {
		fmt.Sprintf("%d", i)
	}
}
```

在命令行中添加`-benchmem`参数以显示内存分配情况，参见下面的指令：

```
$ go test -v -bench=Alloc -benchmem example_test.go
goos: linux
goarch: amd64
Benchmark_Alloc-4 20000000 109 ns/op 16 B/op 2 allocs/op
PASS
ok          command-line-arguments        2.311s
```

代码说明如下：

- 第 1 行的代码中`-bench`后添加了 Alloc，指定只测试 Benchmark_Alloc() 函数。
- 第 4 行代码的“16 B/op”表示每一次调用需要分配 16 个字节，“2 allocs/op”表示每一次调用有两次分配。



#### 控制计时器

有些测试需要一定的启动和初始化时间，如果从 Benchmark() 函数开始计时会很大程度上影响测试结果的精准性。testing.B 提供了一系列的方法可以方便地控制计时器，从而让计时器只在需要的区间进行测试。我们通过下面的代码来了解计时器的控制。  

基准测试中的计时器控制

```go
func Benchmark_Add_TimerControl(b *testing.B) {
	// 重置计时器
	b.ResetTimer()
	// 停止计时器
	b.StopTimer()
	// 开始计时器
	b.StartTimer()
	var n int
	for i := 0; i < b.N; i++ {
		n++
	}
}
```

从 Benchmark() 函数开始，Timer 就开始计数。StopTimer() 可以停止这个计数过程，做一些耗时的操作，通过 StartTimer() 重新开始计时。ResetTimer() 可以重置计数器的数据。  计数器内部不仅包含耗时数据，还包括内存分配的数据。









### 高阶使用

#### 忽视初始化与清理

**问题**：测试之间状态可能相互影响，因为默认情况下每个测试函数共享同一个测试环境。

**解决**：使用`setup`和`teardown`逻辑。可以利用`t.Cleanup`函数注册一个或多个函数，在每次测试结束时执行。

```go
func TestExample(t *testing.T) {
    db := setupDB()
    t.Cleanup(func() { db.Close() })
    // ... 测试逻辑 ...
}
```



#### 忽略并发测试的同步

**问题**：并发测试时，如果没有正确同步，可能会导致竞态条件或测试结果不可预测。

**解决**：使用`t.Parallel()`标记并发安全的测试，并确保并发访问资源时有适当的锁或其他同步机制。

```go
func TestConcurrent(t *testing.T) {
    t.Parallel()
    // 并发安全的测试逻辑
}
```



#### 过度依赖外部服务

**问题**：直接依赖外部服务可能导致测试不稳定或缓慢。

**解决**：采用模拟（mock）或存根（stub）技术隔离外部依赖，或使用测试替身（test doubles）。

```go
type MockService struct{}

func (m *MockService) GetData() []byte {
    return []byte("mocked data")
}

func TestFunctionWithExternalDependency(t *testing.T) {
    mockSvc := &MockService{}
    // 使用mock对象进行测试
}
```



#### 忽视测试覆盖率

**问题**：只关注测试的存在，而不关心覆盖范围，可能导致未测试到的代码路径存在bug。

**解决**：定期检查测试覆盖率，使用`go test -coverprofile=coverage.out`生成覆盖率报告，并分析改进。

然后可以用`go tool` 命令进行分析该文件。



#### setUp和tearDown函数

1. setUpAll在所有case启动前运行一次
2. tearDownAll在所有case结束后运行一次
3. setUp在每个case启动前运行一次
4. tearDown在每个case结束后运行一次

```go
package main

import (
    "log"
    "os"

    "testing"
)

func TestMain(m *testing.M) {
    tearDownAll := setUpAll()

    code := m.Run()

    tearDownAll()   // you cannot use defer tearDownAll()
    os.Exit(code)
}

func setUpAll() func() {
    log.Printf("LLLLLLLL: setUpAll")
    return func() {
        log.Printf("LLLLLLLL: tearDownAll")
    }
}

func setUp(t *testing.T) func(t *testing.T) {
    log.Printf("LLLLLLLL: setUp");

    return func(t *testing.T) {
        log.Printf("LLLLLLLL: tearDown")
    }
}


func TestBasic1(t *testing.T) {
    tearDown := setUp(t)
    defer tearDown(t)

    log.Printf("LLLLLLLL: TestBasic1")
}
func TestBasic2(t *testing.T) {
    tearDown := setUp(t)
    defer tearDown(t)

    log.Printf("LLLLLLLL: TestBasic2")
}
func TestBasic3(t *testing.T) {
    tearDown := setUp(t)
    defer tearDown(t)

    log.Printf("LLLLLLLL: TestBasic3")
}
```

运行:

```go
$ go test -v -run Basic
2021/08/30 06:08:20 LLLLLLLL: setUpAll
=== RUN   TestBasic1
2021/08/30 06:08:20 LLLLLLLL: setUp
2021/08/30 06:08:20 LLLLLLLL: TestBasic1
2021/08/30 06:08:20 LLLLLLLL: tearDown
--- PASS: TestBasic1 (0.00s)
=== RUN   TestBasic2
2021/08/30 06:08:20 LLLLLLLL: setUp
2021/08/30 06:08:20 LLLLLLLL: TestBasic2
2021/08/30 06:08:20 LLLLLLLL: tearDown
--- PASS: TestBasic2 (0.00s)
=== RUN   TestBasic3
2021/08/30 06:08:20 LLLLLLLL: setUp
2021/08/30 06:08:20 LLLLLLLL: TestBasic3
2021/08/30 06:08:20 LLLLLLLL: tearDown
--- PASS: TestBasic3 (0.00s)
PASS
2021/08/30 06:08:20 LLLLLLLL: tearDownAll
ok      _/<pwd> 0.003s
```

