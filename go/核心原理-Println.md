本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 序

在平时开发，在日志这块打印碰到最多的输出，包括日志输出、控制输出，但最终还是调用了Println函数，只不过底层用了不同的输出端。

那么作为平时经常碰到的这个函数，底层到底有什么？它的设计又是什么，此时我们是有义务`爱它就应该了解它`。



### 初识

先看看它的有哪些函数，以及我们要研究的是哪些函数。



#### 函数分类

> **print.go包函数**

```go
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error)
func Printf(format string, a ...interface{}) (n int, err error)
func Sprintf(format string, a ...interface{}) string

func Fprint(w io.Writer, a ...interface{}) (n int, err error)
func Print(a ...interface{}) (n int, err error)
func Sprint(a ...interface{}) string

func Fprintln(w io.Writer, a ...interface{}) (n int, err error)
func Println(a ...interface{}) (n int, err error)
func Sprintln(a ...interface{}) string
```

根据后缀关键字“f”、“ln”、“”进行归类划分：

1、后缀为 *f* 指定了格式化 format 输出；

2、*ln* 是输出结束换行显示；

3、""空是标准输出

```text
Println、Fprintln、Sprintln  输出内容时会加上换行符；
Print、Fprint、Sprint        输出内容时不加上换行符；
Printf、Fprintf、Sprintf     按照指定格式化文本输出内容。
```

根据前缀关键字“F”、“S”、“”进行归类划分：

1、前缀为 F 实现`io.Writer`接口类型 输出；

2、前缀为 S 返回一个字符串而没有任何输出；

3、""空是标准输出

```text
Print、Printf、Println      输出内容到标准输出os.Stdout；
Fprint、Fprintf、Fprintln   输出内容到指定的io.Writer；
Sprint、Sprintf、Sprintln   输出内容到字符串。
```



> **Print**
> func Print(a ...interface{})(n int, err error)
> **a …interface{}**:它包含一些代码中使用的字符串和常量变量。
> 返回写入的字节数和遇到的任何写入错误
> **Println**
> funcPrintln(a ...interface{})(n int, err error)
> 和Print的参数，返回值一样，唯一不同的是，两个输出之间会进行换行
> **Sprint**
> func Sprint(a ...interface{}) string
> 参数和Print一样，但是不会有任何输出，但是可以把返回的字符串复制给新的变量



> **Printf、Sprintf、Fprintf 格式化输出详解，对 format 进行分类【**占位符由'%'打头，动词结尾。占位符由五类元素组成： 标志位（flag），宽度，精度，参数索引，以及动词**】**



#### format分类

**一般类型**

| format 格式 | 解释                     |
| ----------- | ------------------------ |
| %v          | 值的默认格式，输出结构体 |
| %+v         | 输出结构体显示字段名     |
| %#v         | 输出结构体源代码片段     |
| %T          | 输出值的类型             |
| %%          | 输出值并添加百分号       |

**布尔类型**

| format 格式 | 解释          |
| ----------- | ------------- |
| %t          | 返回输出 true |

**整数类型**

| format 格式 | 解释                                                         |
| ----------- | ------------------------------------------------------------ |
| %b          | 输出标准的二进制格式化                                       |
| %c          | 输出对应的unicode码的一个字符                                |
| %d          | 输出标准的十进制格式化                                       |
| %o          | 输出标准的八进制格式化                                       |
| %q          | 要输出的值是双引号输出就是双引号字符串；另外一种就是 go 自转义的 unicode 单引号字符 |
| %x（小写x） | 输出十六进制编码，字母形式为小写 a-f                         |
| %X（大写X） | 输出十六进制编码，字母形式为大写 A-F                         |
| %U（大写U） | 输出Unicode格式                                              |

**浮点数与复数**

| format 格式 | 解释                                                   |
| ----------- | ------------------------------------------------------ |
| %b          | 无小数部分、二进制指数的科学计数法                     |
| %e          | 科学计数法（小写e）                                    |
| %E          | 科学计数法（大写E）                                    |
| %f          | 十进制小数                                             |
| %F          | 和%f一样                                               |
| %g          | 根据实际情况采用%e或%f格式（以获得更简洁、准确的输出） |
| %G          | 根据实际情况采用%E或%F格式（以获得更简洁、准确的输出） |

**字符串和 []byte**

| format 格式 | 解释                                      |
| ----------- | ----------------------------------------- |
| %s          | 输出字符串                                |
| %q          | 源代码中那样带有双引号的输出              |
| %x（小写x） | 每个字节用两字符十六进制数表示（使用a-f） |
| %X（大写X） | 每个字节用两字符十六进制数表示（使用A-F） |

**指针**

| format 格式 | 解释                         |
| ----------- | ---------------------------- |
| %p          | 输出十六进制，并加上前导的0x |

**%f 格式**

Go语言中对宽度和有效数字的限定：**%[宽度].[精度]f ；**默认的`%f`输出格式是小数点后要有6位有效数字，不够补后置零

宽度是指整个浮点数的宽度，是指整数位数+小数位数 + 1(小数点算一位)，指定的宽度小于实际宽度或不指定宽度时，以实际宽度展示；当指定宽度大于实际宽度时，按照对齐方式和填充字符来填充。



#### 小结

以上是一些fmt一些基础知识，但这些函数大多数最终会调用Println函数进行输出，比如格式化函数先拼接好再输出，其他函数大多数是不用输出（不再本次分析范围内）。



### Println

Println函数接受参数a，其类型为…interface{}。用过Java的对这个应该比较熟悉，Java中也有…的用法。其作用是传入可变的参数，而interface{}类似于Java中的Object，代表任何类型。

所以，…interface{}转换成Java的概念，就是`Object args ...`。

Println函数中没有什么实现，只是return了Fprintln函数。

```
func Println(a ...interface{}) (n int, err error) {
	return Fprintln(os.Stdout, a...)
}
```

而在此处的…放在了参数的后面。我们知道`...interface{}`是代表可变参数，即函数可接收任意数量的参数，而且参数参数分开写的。

当我们再调用这个函数的时候，我们就没有必要再将参数一个一个传给被调用函数了，直接使用a…就可以达到相同的效果。



#### Fprintln

该函数接收参数os.Stdout.write，和需要打印的数据作为参数。

```go
func Fprintln(w io.Writer, a ...interface{}) (n int, err error) {
	p := newPrinter()
	p.doPrintln(a)
	n, err = w.Write(p.buf)
	p.free()
	return
}
```

#### sync.Pool

从广义上看，newPrinter申请了一个临时对象池。我们逐行来看newPrinter函数做了什么。

```
var ppFree = sync.Pool{
	New: func() interface{} { return new(pp) },
}

// newPrinter allocates a new pp struct or grabs a cached one.
func newPrinter() *pp {
	p := ppFree.Get().(*pp)
	p.panicking = false
	p.erroring = false
	p.wrapErrs = false
	p.fmt.init(&p.buf)
	return p
}
```

sync.Pool是go的临时对象池，用于存储被分配了但是没有被使用，但是未来可能会使用的值。以此来减少 GC的压力。



#### ppFree.Get

ppFree.Get()上有大量的注释。

> Get selects an arbitrary item from the Pool, removes it from the Pool, and returns it to the caller.
>
> Get may choose to ignore the pool and treat it as empty. Callers should not assume any relation between values passed to Put and the values returned by Get.
>
> If Get would otherwise return nil and p.New is non-nil, Get returns the result of calling p.New.

麻瓜翻译一波。

> Get会从临时对象池中任意选一个printer返回给调用者，并且将此项从对象池中移除。
>
> Get也可以选择把临时对象池当成空的忽略。调用者不应该假设传递给Put方法的值和Get返回的值之间存在任何关系。
>
> 如果Get返回nil或者p，New就一定不为空。Get将返回调用p.New的结果。

上面提到的Put方法，作用是将对象加入到临时对象池中。

`p := ppFree.Get().(*pp)`下面的三个参数分别代表什么呢？

| 参数名      | 用途                                                   |
| :---------- | :----------------------------------------------------- |
| p.panicking | 由catchPanic设置，是为了避免在panic和recover中无限循环 |
| p.erroring  | 当打印错误的标识符的时候，防止调用handleMethods        |
| p.wrapErrs  | 当格式字符串包含了动词时的设置                         |
| fmt.init    | 初始化 fmt 配置，会设置 buf 并且清空 fmtFlags 标志位   |

然后就返回这个新建的printer给调用方。



#### doPrintln

接下来是doPrintln函数。

doPrintln就跟doPrint类似，但是doPrintln总是会在参数之间添加一个空格，并且在最后一个参数后面添加换行符。以下是两种输出方式的对比。

```
fmt.Println("test", "hello", "word") // test hello word
fmt.Print("test", "hello", "word")   // testhelloword%
```

看了样例，我们再具体看一下doPrintln的具体实现。

```go
func (p *pp) doPrintln(a []interface{}) {
	for argNum, arg := range a {
		if argNum > 0 {
			p.buf.writeByte(' ')
		}
		p.printArg(arg, 'v')
	}
	p.buf.writeByte('\n')
}
```

这个函数的思路很清晰。遍历所有传入的需要print的参数，在除了第一个 参数以外的所有参数的前面加上一个空格，写入buffer中。然后调用printArg函数，再将换行符写入buffer中。

writeByte的实现很简单，使用了append函数，将传入的参数，append到buffer中。

```go
func (b *buffer) writeByte(c byte) {
	*b = append(*b, c)
}
```



#### printArg

从上可以看出，调用printArg函数的时候，传入了两个参数。

第一个是需要打印的参数，第二个则是verb，在doPrintln中我们传的是单引号的v。那么在go中的单引号和双引号有什么区别呢？下面我们通过一个表格来对比一下在不同的语言中，单引号和双引号的区别。

| 语言       | 单引号 | 双引号 |
| :--------- | :----- | :----- |
| Java       | char   | String |
| JavaScript | string | string |
| go         | rune   | String |
| Python     | string | string |



### rune

那么rune到底是什么类型呢？rune是int32的别名，在任何方面等于int32相同，用于区分字符串和整形。其实现很简单，`type rune = int32`，rune常用来表示Unicode中的码点，其例子如下所示。

```
str := "hello 你好"
fmt.Println([]rune(str)) // [104 101 108 108 111 32 20320 22909]
```

说到了rune就不得不说一下byte。同样，我们通过例子来看一下byte和rune的区别。

```
str := "hello 你好"
fmt.Println([]rune(str)) // [104 101 108 108 111 32 20320 22909]
fmt.Println([]byte(str)) // [104 101 108 108 111 32 228 189 160 229 165 189]
```

没错，区别就在类型上。rune是`type rune = int32`，一个字节；而byte是`type byte = uint8`，四个字节。实际上，golang中的字符串的底层是靠byte数组实现的。如果我们处理的数据中出现了中文字符，都可用rune来处理。例如。

```go
str := "hello 你好"
fmt.Println(len(str))         // 12
fmt.Println(len([]rune(str))) // 8
```

#### printArg具体实现

```go
func (p *pp) printArg(arg interface{}, verb rune) {
	p.arg = arg
	p.value = reflect.Value{}

	if arg == nil {
		switch verb {
		case 'T', 'v':
			p.fmt.padString(nilAngleString)
		default:
			p.badVerb(verb)
		}
		return
	}

	switch verb {
	case 'T':
		p.fmt.fmtS(reflect.TypeOf(arg).String())
		return
	case 'p':
		p.fmtPointer(reflect.ValueOf(arg), 'p')
		return
	}

  switch f := arg.(type) {
	case bool:
		p.fmtBool(f, verb)
	case float32:
		p.fmtFloat(float64(f), 32, verb)
	case float64:
		p.fmtFloat(f, 64, verb)
	case complex64:
		p.fmtComplex(complex128(f), 64, verb)
	case complex128:
		p.fmtComplex(f, 128, verb)
	case int:
		p.fmtInteger(uint64(f), signed, verb)
	case int8:
		p.fmtInteger(uint64(f), signed, verb)
	case int16:
		p.fmtInteger(uint64(f), signed, verb)
	case int32:
		p.fmtInteger(uint64(f), signed, verb)
	case int64:
		p.fmtInteger(uint64(f), signed, verb)
	case uint:
		p.fmtInteger(uint64(f), unsigned, verb)
	case uint8:
		p.fmtInteger(uint64(f), unsigned, verb)
	case uint16:
		p.fmtInteger(uint64(f), unsigned, verb)
	case uint32:
		p.fmtInteger(uint64(f), unsigned, verb)
	case uint64:
		p.fmtInteger(f, unsigned, verb)
	case uintptr:
		p.fmtInteger(uint64(f), unsigned, verb)
	case string:
		p.fmtString(f, verb)
	case []byte:
		p.fmtBytes(f, verb, "[]byte")
	case reflect.Value:
		if f.IsValid() && f.CanInterface() {
			p.arg = f.Interface()
			if p.handleMethods(verb) {
				return
			}
		}
		p.printValue(f, verb, 0)
	default:
		if !p.handleMethods(verb) {
			p.printValue(reflect.ValueOf(f), verb, 0)
		}
	}
}
```

可以看到有一部分类型是通过反射获取到的，而大部分都是switch case出来的，并不是所有的类型都用的反射，相对的提高了效率。

例如，我们传入的是字符串。则接下来就会走到fmtString。

#### fmtString

从printArg中带来的参数有需要打印的字符串，以及rune类型的’v’。

```
func (p *pp) fmtString(v string, verb rune) {
	switch verb {
	case 'v':
		if p.fmt.sharpV {
			p.fmt.fmtQ(v)
		} else {
			p.fmt.fmtS(v)
		}
	case 's':
		p.fmt.fmtS(v)
	case 'x':
		p.fmt.fmtSx(v, ldigits)
	case 'X':
		p.fmt.fmtSx(v, udigits)
	case 'q':
		p.fmt.fmtQ(v)
	default:
		p.badVerb(verb)
	}
}
```

`p.fmt.sharpV`在过程中没有被重新赋值，初始化的零值为false。所以下一步会进入fmtS。

#### fmtS

```go
func (f *fmt) fmtS(s string) {
	s = f.truncateString(s)
	f.padString(s)
}
```

如果存在设定的精度，则truncate将字符串s截断为指定的精度。多用于需要输出数字时。

```go
func (f *fmt) truncateString(s string) string {
	if f.precPresent {
		n := f.prec
		for i := range s {
			n--
			if n < 0 {
				return s[:i]
			}
		}
	}
	return s
}
```

而padString则将字符串s写入buffer中，最后调用io的包输出就好了。



### free

```go
func (p *pp) free() {
	if cap(p.buf) > 64<<10 {
		return
	}

	p.buf = p.buf[:0]
	p.arg = nil
	p.value = reflect.Value{}
	p.wrappedErr = nil
	ppFree.Put(p)
}
```

在前面讲过，要打印的时候，需要从临时对象池中获取一个对象，避免重复创建。而在此处，用完之后就需要通过Put函数将其放回临时对象池中，已备下次调用。

当然，并不是无限的将用过的变量放入对象池。如果缓冲区的大小超过了设定的阙值也就是65535，就无法再执行后续的操作了。画外音：异步获取资源的协程数和这方面有关系的。