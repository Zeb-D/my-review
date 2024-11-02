本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 序

我们在前面的一篇讲了[golang 常见语法入门](https://mp.weixin.qq.com/s?__biz=Mzg5NDc0Mzc4MQ==&mid=2247483919&idx=1&sn=7cbde0c0d2f3336dad204e46b95fd46c&chksm=c01ba785f76c2e930846666c2e0f016e926e2474108326f69e94ee077e98dd413b772c84e134#rd)，这篇讲了比较多的语法，这是语法关键词的运用可以通过熟练实践就可以达到的，那么大家想不想知道背后golang 是底层是怎么实现的；



### make 和 new

- make 的作用是初始化内置的数据结构，也就是我们在前面提到的切片、哈希表和 Channel；
- new 的作用是根据传入的类型分配一片内存空间并返回指向这片内存空间的指针；



#### make关键字

在编译期间的类型检查阶段，Go 语言会将代表 make 关键字的 OMAKE 节点根据参数类型的不同转换成了 OMAKESLICE、OMAKEMAP 和 OMAKECHAN 三种不同类型的节点，这些节点会调用不同的运行时函数来初始化相应的数据结构。



#### new 关键字

new底层是通过runtime.newobject 实现的，runtime.newobject 函数会获取传入类型占用空间的大小，调用 runtime.mallocgc 在堆上申请一片内存空间并返回指向这片内存空间的指针：

```
func newobject(typ *_type) unsafe.Pointer {
        return mallocgc(typ.size, typ, true)
}
```



#### make和new的区别

1. make 只能用于创建 map、slice、channel；
2. make 会进行初始化，new 不会初始化，new只是申请一段内存空间，并返回指向这片内存的指针



### iota

```
const (
    bit0, mask0 = 1 << iota, 1<<iota - 1//const声明第0行，即iota==0
    bit1, mask1                            //const声明第1行，即iota==1, 表达式继承上面的语句
    _, _                                   //const声明第2行，即iota==2
    bit3, mask3                            //const声明第3行，即iota==3
)
```

- 第0行的表达式展开即bit0, mask0 = 1 << 0, 1<<0 - 1，所以bit0 == 1，mask0 == 0；
- 第1行没有指定表达式继承第一行，即bit1, mask1 = 1 << 1, 1<<1 - 1，所以bit1 == 2，mask1 == 1；
- 第2行没有定义常量
- 第3行没有指定表达式继承第一行，即bit3, mask3 = 1 << 3, 1<<3 - 1，所以bit0 == 8，mask0 == 7；



### defer

#### defer规则

#### 规则一：延迟函数的参数在defer语句出现时就已经确定下来了

官方给出一个例子，如下所示：

```
func a() {
    i := 0
    defer fmt.Println(i)
    i++
    return
}
```

defer语句中的fmt.Println()参数i值在defer出现时就已经确定下来，实际上是拷贝了一份。后面对变量i的修改不会影响fmt.Println()函数的执行，仍然打印"0"。

注意：对于指针类型参数，规则仍然适用，只不过延迟函数的参数是一个地址值，这种情况下，defer后面的语句对变量的修改可能会影响延迟函数。



#### 规则二：延迟函数执行按后进先出顺序执行，即先出现的defer最后执行

这个规则很好理解，定义defer类似于入栈操作，执行defer类似于出栈操作。

设计defer的初衷是简化函数返回时资源清理的动作，资源往往有依赖顺序，比如先申请A资源，再跟据A资源申请B资源，跟据B资源申请C资源，即申请顺序是:A-->B-->C，释放时往往又要反向进行。这就是把deffer设计成FIFO的原因。

每申请到一个用完需要释放的资源时，立即定义一个defer来释放资源是个很好的习惯。

```
func main() {
    defer fmt.Println(1)
    defer fmt.Println(2)
    defer fmt.Println(3)
}

输出
3
2
1
```



#### 规则三：延迟函数可能操作主函数的具名返回值

定义defer的函数，即主函数可能有返回值，返回值有没有名字没有关系，defer所作用的函数，即延迟函数可能会影响到返回值。

若要理解延迟函数是如何影响主函数返回值的，只要明白函数是如何返回的就足够了。



#### 函数返回过程

有一个事实必须要了解，关键字*return*不是一个原子操作，实际上*return*只代理汇编指令*ret*，即将跳转程序执行。比如语句return i，实际上分两步进行，即将i值存入栈中作为返回值，然后执行跳转，而defer的执行时机正是跳转前，所以说defer执行时还是有机会操作返回值的。

举个实际的例子进行说明这个过程：

```
func deferFuncReturn() (result int) {
    i := 1
    
    deferfunc() {
       result++
    }()
    
    return i
}
```

该函数的return语句可以拆分成下面两行：

```
result = i
return
```

而延迟函数的执行正是在return之前，即加入defer后的执行过程如下：

```
result = i
result++
return
```

所以上面函数实际返回i++值。

关于主函数有不同的返回方式，但返回机制就如上机介绍所说，只要把return语句拆开都可以很好的理解，下面分别举例说明



##### 主函数拥有匿名返回值，返回字面值

一个主函数拥有一个匿名的返回值，返回时使用字面值，比如返回"1"、"2"、"Hello"这样的值，这种情况下defer语句是无法操作返回值的。

一个返回字面值的函数，如下所示：

```
func foo() int {
    var i int
    deferfunc() {
        i++
    }()

    return 1
}
```

上面的return语句，直接把1写入栈中作为返回值，延迟函数无法操作该返回值，所以就无法影响返回值。



##### 主函数拥有匿名返回值，返回变量

一个主函数拥有一个匿名的返回值，返回使用本地或全局变量，这种情况下defer语句可以引用到返回值，但不会改变返回值。

一个返回本地变量的函数，如下所示：

```
func foo() int {
    var i int
    deferfunc() {
        i++
    }()
    
    return i
}
```

上面的函数，返回一个局部变量，同时defer函数也会操作这个局部变量。对于匿名返回值来说，可以假定仍然有一个变量存储返回值，假定返回值变量为"anony"，上面的返回语句可以拆分成以下过程：

```
anony = i
i++
return
```

由于i是整型，会将值拷贝给anony，所以defer语句中修改i值，对函数返回值不造成影响。



##### 主函数拥有具名返回值

主函声明语句中带名字的返回值，会被初始化成一个局部变量，函数内部可以像使用局部变量一样使用该返回值。如果defer语句操作该返回值，可能会改变返回结果。

一个影响函返回值的例子：

```
func foo() (ret int) {
    deferfunc() {
        ret++
    }()

    return0
}
```

上面的函数拆解出来，如下所示：

```
ret = 0
ret++
return
```

函数真正返回前，在defer中对返回值做了+1操作，所以函数最终返回1。



#### defer实现原理

```
func main() {

    defer greet("friend")

    println("welcome")

}

func greet(text string) {
    print("hello " + text)

}
```

上面代码翻译成汇编后代码如下：

```
"".main STEXT size=203 args=0x0 locals=0x68
        0x000000000 (main.go:5)        TEXT    "".main(SB), ABIInternal, $104-0
        // ...
        0x003400052 (main.go:6)        CALL    runtime.deferproc(SB)
        // ...
        0x00a200162 (main.go:11)       CALL    runtime.deferreturn(SB)
        // ...
        0x00b000176 (main.go:11)       RET
```

可以看到是通过调用 runtime.deferproc(SB) 和 runtime.deferreturn(SB) 函数来实现 defer 功能的。



##### runtime.deferproc

runtime.deferproc 会为 defer 创建一个新的 runtime._defer 结构体、设置它的函数指针 fn、程序计数器 pc 和栈指针 sp 并将相关的参数拷贝到相邻的内存空间中：

```
func deferproc(siz int32, fn *funcval) {
        sp := getcallersp()
        argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
        callerpc := getcallerpc()

        d := newdefer(siz)
        if d._panic != nil {
                throw("deferproc: d.panic != nil after newdefer")
        }
        d.fn = fnd.pc = callerpcd.sp = spswitch siz {
        case0:
        case sys.PtrSize:
                *(*uintptr)(deferArgs(d)) = *(*uintptr)(unsafe.Pointer(argp))
        default:
                memmove(deferArgs(d), unsafe.Pointer(argp), uintptr(siz))
        }

        return0()
}
```

runtime._defer 结构体定义如下：

```
type _defer struct {
        siz       int32// 是参数和结果的内存大小
        started   bool
        openDefer bool// 表示当前 defer 是否经过开放编码的优化
        sp        uintptr// 当前调用栈的栈顶指针
        pc        uintptr// 调用方的程序计数器
        fn        *funcval // defer 关键字中传入的函数
        _panic    *_panic
        link      *_defer
}
```

runtime._defer 结构体是延迟调用链表上的一个元素，所有的结构体都会通过 link 字段串联成链表。



![图片](https://mmbiz.qpic.cn/mmbiz_png/2kOTFMdShwtMaRhibm1nUEaQ5CglfkHdUe8FaiabFia7T0ppiaGrS1KQjuK08nlY6VadLU6fl7licltZ0tHhyUrbTBQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



##### runtime.deferreturn

runtime.deferreturn 会从 Goroutine 的 _defer 链表中取出最前面的 runtime._defer 并调用 runtime.jmpdefer 传入需要执行的函数和参数：

```
func deferreturn() {
    gp := getg()
    for {
       d := gp._defer
       if d == nil {
          return
       }
       sp := getcallersp()
       if d.sp != sp {
          return
       }
       if d.openDefer {
          done := runOpenDeferFrame(d)
          if !done {
             throw("unfinished open-coded defers in deferreturn")
          }
          gp._defer = d.link
          freedefer(d)
          // If this frame uses open defers, then this
          // must be the only defer record for the
          // frame, so we can just return.
          return
       }

       fn := d.fn
       d.fn = nil
       gp._defer = d.link
       freedefer(d)
       fn()
    }
}
```

在执行defer时都会判断caller栈顶指针是否 defer 结构体中sp的相等，如果相等说明defer是由这个caller创建的，所以可以执行。



#### defer的优化

1. **栈上分配**：Go 1.13 中对 defer 关键字进行了优化，当该关键字在函数体中最多执行一次时，编译期间的 cmd/compile/internal/gc.state.call 会将结构体分配到栈上，将 defer 关键字的额外开销降低~30%。

2. **开放编码：**Go 语言在 1.14 中通过开放编码（Open Coded）实现 defer 关键字，该设计使用代码内联优化 defer 关键的额外开销并引入函数数据 funcdata 管理 panic 的调用(3)，该优化可以将 defer 的调用开销从 1.13 版本的 ~35ns 降低至 ~6ns 左右，开放编码只会在满足以下的条件时启用：

3. 1. 函数的 defer 数量少于或者等于 8 个；
   2. 函数的 defer 关键字不能在循环中执行；
   3. 函数的 return 语句与 defer 语句的乘积小于或者等于 15 个；



### panic 和 recover

#### panic 和 recover的使用

1. panic 能够改变程序的控制流，调用 panic 后会立刻停止执行当前函数的剩余代码，并在当前 Goroutine 中递归执行调用方的 defer；
2. recover 可以中止 panic 造成的程序崩溃。它是一个只能在 defer 中发挥作用的函数，在其他作用域中调用不会发挥作用；



注意:

**跨协程无效**

panic 只会触发当前 Goroutine 的延迟函数调用，我们可以通过如下所示的代码了解该现象：

```
func main() {
        deferprintln("in main")
        gofunc() {
                deferprintln("in goroutine")
                panic("")
        }()

        time.Sleep(1 * time.Second)
}

$ go run main.go
in goroutine
panic:
...
```



**recover正确的使用姿势**

recover 只有在 defer 调用的函数中才有效，defer函数外或者defer函数和recover之间还存在调用

```
// 无效的
func main() {
        defer fmt.Println("in main")
        if err := recover(); err != nil {
                fmt.Println(err)
        }

        panic("unknown err")
}

$ go run main.go
in main
panic: unknown err

goroutine 1 [running]:
main.main()
        ...
exit status 2
// 无效的
func main() {
        deferfunc(){
            fmt.Println("in main")
            
            dorecover()
        }()

        panic("unknown err")
}

func dorecover() {
    if err := recover(); err != nil {
            fmt.Println(err)
    }
}

$ go run main.go
in main
panic: unknown err

goroutine 1 [running]:
main.main()
        ...
exit status 2
// 有效的
func main() {
        deferfunc(){
            fmt.Println("in main")
            if err := recover(); err != nil {
                    fmt.Println(err)
            }
        }()
        
        panic("unknown err")
}

$ go run main.go
in main
unknown err
// 有效的
func main() {
        defer dorecover()
        
        panic("unknown err")
}


func dorecover() {
    fmt.Println("in main")
    if err := recover(); err != nil {
            fmt.Println(err)
    }
}

$ go run main.go
in main
unknown err
```



**panic和defer可以嵌套**

```
func main() {
        defer fmt.Println("in main")
        deferfunc() {
                deferfunc() {
                        panic("panic again and again")
                }()
                panic("panic again")
        }()

        panic("panic once")
}

$ go run main.go
in main
panic: panic once
        panic: panic again
        panic: panic again and again

goroutine 1 [running]:
...
exit status 2
```



#### **panic 和 recover底层原理**

##### 程序崩溃

编译器会将关键字 panic 转换成 runtime.gopanic，该函数的执行过程包含以下几个步骤：

1. 创建新的 runtime._panic 并添加到所在 Goroutine 的 _panic 链表的最前面；
2. 在循环中不断从当前 Goroutine 的 _defer 中链表获取 runtime._defer 并运行延迟调用函数；
3. 判断 panic 是否recover
4. 调用 runtime.fatalpanic 中止整个程序；

```
func gopanic(e interface{}) {
        gp := getg()
        ...
        
        // 创建新的 runtime._panic 并添加到所在 Goroutine 的 _panic 链表的最前面
        var p _panic
        p.arg = e
        p.link = gp._panic
        gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

        // 在循环中不断从当前 Goroutine 的 _defer 中链表获取 runtime._defer 并运行延迟调用函数；
        for {
                d := gp._defer
                if d == nil {
                        break
                }

                d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))
                
                // 执行延迟函数
                done := true
                if d.openDefer {
                    done = runOpenDeferFrame(d)
                    if done && !d._panic.recovered {
                       addOneOpenDeferFrame(gp, 0, nil)
                    }
                } else {
                    p.argp = unsafe.Pointer(getargp())
                    d.fn()
                }

                d._panic = nil
                d.fn = nil
                gp._defer = d.link

                freedefer(d)
                
                // 判断 panic 是否recover
                if p.recovered {
                        ...
                }
        }

        // 中止整个程序
        fatalpanic(gp._panic)
        *(*int)(nil) = 0
}
```



##### 崩溃恢复

编译器会将关键字 recover 转换成 runtime.gorecover：

```
func gorecover(argp uintptr) interface{} {
        gp := getg()
        p := gp._panic
        if p != nil && !p.recovered && argp == uintptr(p.argp) {
                p.recovered = true
                return p.arg
        }
        returnnil
}
```

该函数的实现很简单，如果当前 Goroutine 没有调用 panic，那么该函数会直接返回 nil，这也是崩溃恢复在非 defer 中调用会失效的原因。在正常情况下，它会修改 runtime._panic 的 recovered 字段，runtime.gorecover 函数中并不包含恢复程序的逻辑，程序的恢复是由 runtime.gopanic 函数负责的：

```
func gopanic(e interface{}) {
        ...

        for {
                // 执行延迟调用函数，可能会设置 p.recovered = true
                ...

                pc := d.pc
                sp := unsafe.Pointer(d.sp)

                ...
                if p.recovered {
                        gp._panic = p.link
                        for gp._panic != nil && gp._panic.aborted {
                                gp._panic = gp._panic.link
                        }
                        if gp._panic == nil {
                                gp.sig = 0
                        }
                        gp.sigcode0 = uintptr(sp)
                        gp.sigcode1 = pc
                        mcall(recovery)
                        throw("recovery failed")
                }
        }
        ...
}
```

从 runtime._defer 中取出了程序计数器 pc 和栈指针 sp 并调用 runtime.recovery 函数触发 Goroutine 的调度，调度之前会准备好 sp、pc 以及函数的返回值：

```
func recovery(gp *g) {
        sp := gp.sigcode0
        pc := gp.sigcode1

        gp.sched.sp = sp
        gp.sched.pc = pc
        gp.sched.lr = 0
        gp.sched.ret = 1
        gogo(&gp.sched)
}
```



#### 小结

1. 编译器会负责做转换关键字的工作；

2. 1. 将 panic 和 recover 分别转换成 runtime.gopanic 和 runtime.gorecover；
   2. 将 defer 转换成 runtime.deferproc 函数；
   3. 在调用 defer 的函数末尾调用 runtime.deferreturn 函数；

3. 在运行过程中遇到 runtime.gopanic 方法时，会从 Goroutine 的链表依次取出 runtime._defer 结构体并执行；

4. 如果调用延迟执行函数时遇到了 runtime.gorecover 就会将 _panic.recovered 标记成 true 并返回 panic 的参数；

5. 1. 在这次调用结束之后，runtime.gopanic 会从 runtime._defer 结构体中取出程序计数器 pc 和栈指针 sp 并调用 runtime.recovery 函数进行恢复程序；
   2. runtime.recovery 会根据传入的 pc 和 sp 跳转回 runtime.deferproc；
   3. 编译器自动生成的代码会发现 runtime.deferproc 的返回值不为 0，这时会跳回 runtime.deferreturn 并恢复到正常的执行流程；

6. 如果没有遇到 runtime.gorecover 就会依次遍历所有的 runtime._defer，并在最后调用 runtime.fatalpanic 中止程序、打印 panic 的参数并返回错误码 2；



### for range

#### range for slice

下面的注释解释了遍历slice的过程：

```
// The loop we generate:
//   for_temp := range
//   len_temp := len(for_temp)
//   for index_temp = 0; index_temp < len_temp; index_temp++ {
//           value_temp = for_temp[index_temp]
//           index = index_temp
//           value = value_temp
//           original body
//   }
```

遍历slice前会先获以slice的长度len_temp作为循环次数，循环体中，每次循环会先获取元素值，如果for-range中接收index和value的话，则会对index和value进行一次赋值。

由于循环开始前循环次数就已经确定了，所以循环过程中新添加的元素是没办法遍历到的。

另外，数组与数组指针的遍历过程与slice基本一致，不再赘述。



#### range for map

下面的注释解释了遍历map的过程：

```
// The loop we generate:
//   var hiter map_iteration_struct
//   for mapiterinit(type, range, &hiter); hiter.key != nil; mapiternext(&hiter) {
//           index_temp = *hiter.key
//           value_temp = *hiter.val
//           index = index_temp
//           value = value_temp
//           original body
//   }
```

遍历map时没有指定循环次数，循环体与遍历slice类似。由于map底层实现与slice不同，map底层使用hash表实现，插入数据位置是随机的，所以遍历过程中新插入的数据不能保证遍历到。



#### range for channel

遍历channel是最特殊的，这是由channel的实现机制决定的：

```
// The loop we generate:
//   for {
//           index_temp, ok_temp = <-range
//           if !ok_temp {
//                   break
//           }
//           index = index_temp
//           original body
//   }
```

channel遍历是依次从channel中读取数据,读取前是不知道里面有多少个元素的。如果channel中没有元素，则会阻塞等待，如果channel已被关闭，则会解除阻塞并退出循环。

注：

- 上述注释中index_temp实际上描述是有误的，应该为value_temp，因为index对于channel是没有意义的。
- 使用for-range遍历channel时只能获取一个返回值。





### 参考

https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/

https://books.studygolang.com/GoExpertProgramming/chapter02/2.1-defer.html