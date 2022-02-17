本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

学了几门语言，无论是高并发入门，还是高深压榨机器，说白了，高级编程语言封装越厉害，越容易入门（卷），对底层性能不能得到充分代码性能率；

golang在2019年就开始火了，此时是否能在一些方面进行研究，或者结合其他语言进行侧向对比；

本文章从汇编代码来各项对比函数调用的函数；



### 汇编代码

#### C++

我们准备一段简单的函数调用代码。

[func_main.cpp](https://github.com/Zeb-D/my-cpp/blob/master/test/func_main.cpp)

```
int a(int p) {
    return 2*p;
}

int main() {
    int i;
    for (i = 0; i < 100000000; i++) {
        a(i);
    }
    return 0;
}
```

用 gcc 来查看下汇编代码。

```
gcc -S func_main.cpp 
```

汇编源码如下：

```
	.section	__TEXT,__text,regular,pure_instructions
	.build_version macos, 10, 14	sdk_version 10, 14
	.globl	__Z1ai                  ## -- Begin function _Z1ai
	.p2align	4, 0x90
__Z1ai:                                 ## @_Z1ai
	.cfi_startproc
## %bb.0:
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register %rbp
	movl	%edi, -4(%rbp)
	movl	-4(%rbp), %edi
	shll	$1, %edi
	movl	%edi, %eax
	popq	%rbp
	retq
	.cfi_endproc
                                        ## -- End function
	.globl	_main                   ## -- Begin function main
	.p2align	4, 0x90
_main:                                  ## @main
	.cfi_startproc
## %bb.0:
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register %rbp
	subq	$16, %rsp
	movl	$0, -4(%rbp)
	movl	$0, -8(%rbp)
LBB1_1:                                 ## =>This Inner Loop Header: Depth=1
	cmpl	$100000000, -8(%rbp)    ## imm = 0x5F5E100
	jge	LBB1_4
## %bb.2:                               ##   in Loop: Header=BB1_1 Depth=1
	movl	-8(%rbp), %edi
	callq	__Z1ai
	movl	%eax, -12(%rbp)         ## 4-byte Spill
## %bb.3:                               ##   in Loop: Header=BB1_1 Depth=1
	movl	-8(%rbp), %eax
	addl	$1, %eax
	movl	%eax, -8(%rbp)
	jmp	LBB1_1
LBB1_4:
	xorl	%eax, %eax
	addq	$16, %rsp
	popq	%rbp
	retq
	.cfi_endproc
                                        ## -- End function

.subsections_via_symbols
```

可以看到，在C++语言中：

**主要通过寄存器传递参数**
所以，C 语言函数的性能杠杠的。寄存器是整个计算机体系结构中访问最最快的存储了。只有当参数数量大于 6 的时候，才开始使用栈。

**固定 eax 寄存器返回数据**
因为固定使用 eax 寄存器做返回数据之用，所以在 C 语言中无法支持多个返回值。我们接下来看看 Golang 是如何支持多个返回值的。



#### go

同样先写一段最简单的函数调用代码[func_cost.go](https://github.com/Zeb-D/go-util/blob/master/todo/main/fun/main/func_cost.go)。

```
package main

//统计下golang 方法调用的耗时
//go build -gcflags="-m -l" func_cost.go
func main() {
	for i := 0; i < 100000000; i++ {
		a(i)
	}
}

func a(a int) int {
	return 2 * a
}
```

然后查看其汇编代码。

```
//为了方便查看，使用-N -l 参数，能阻止编译器对汇编代码的优化
#go tool compile -S -N -l main.go > main.s
```

结果是这样的：

```
"".main STEXT size=104 args=0x0 locals=0x20 funcid=0x0
	0x0000 00000 (func_cost.go:5)	TEXT	"".main(SB), ABIInternal, $32-0
	0x0000 00000 (func_cost.go:5)	MOVQ	(TLS), CX
	0x0009 00009 (func_cost.go:5)	CMPQ	SP, 16(CX)
	0x000d 00013 (func_cost.go:5)	PCDATA	$0, $-2
	0x000d 00013 (func_cost.go:5)	JLS	97
	0x000f 00015 (func_cost.go:5)	PCDATA	$0, $-1
	0x000f 00015 (func_cost.go:5)	SUBQ	$32, SP	//在栈上分配32字节
	0x0013 00019 (func_cost.go:5)	MOVQ	BP, 24(SP)	//保存BP
	0x0018 00024 (func_cost.go:5)	LEAQ	24(SP), BP
	0x001d 00029 (func_cost.go:5)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x001d 00029 (func_cost.go:5)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x001d 00029 (func_cost.go:6)	MOVQ	$0, "".i+16(SP)
	0x0026 00038 (func_cost.go:6)	JMP	40
	0x0028 00040 (func_cost.go:6)	CMPQ	"".i+16(SP), $100000000
	0x0031 00049 (func_cost.go:6)	JLT	53
	0x0033 00051 (func_cost.go:6)	JMP	86
	0x0035 00053 (func_cost.go:7)	MOVQ	"".i+16(SP), AX
	0x003a 00058 (func_cost.go:7)	MOVQ	AX, (SP)
	0x003e 00062 (func_cost.go:7)	PCDATA	$1, $0
	0x003e 00062 (func_cost.go:7)	NOP
	0x0040 00064 (func_cost.go:7)	CALL	"".a(SB)
	0x0045 00069 (func_cost.go:7)	JMP	71
	0x0047 00071 (func_cost.go:6)	MOVQ	"".i+16(SP), AX
	0x004c 00076 (func_cost.go:6)	INCQ	AX
	0x004f 00079 (func_cost.go:6)	MOVQ	AX, "".i+16(SP)
	0x0054 00084 (func_cost.go:6)	JMP	40
	0x0056 00086 (func_cost.go:6)	PCDATA	$1, $-1
	0x0056 00086 (func_cost.go:6)	MOVQ	24(SP), BP
	0x005b 00091 (func_cost.go:6)	ADDQ	$32, SP
	0x005f 00095 (func_cost.go:6)	NOP
	0x0060 00096 (func_cost.go:6)	RET
	0x0061 00097 (func_cost.go:6)	NOP
	0x0061 00097 (func_cost.go:5)	PCDATA	$1, $-1
	0x0061 00097 (func_cost.go:5)	PCDATA	$0, $-2
	0x0061 00097 (func_cost.go:5)	CALL	runtime.morestack_noctxt(SB)
	0x0066 00102 (func_cost.go:5)	PCDATA	$0, $-1
	0x0066 00102 (func_cost.go:5)	JMP	0
	0x0000 65 48 8b 0c 25 00 00 00 00 48 3b 61 10 76 52 48  eH..%....H;a.vRH
	0x0010 83 ec 20 48 89 6c 24 18 48 8d 6c 24 18 48 c7 44  .. H.l$.H.l$.H.D
	0x0020 24 10 00 00 00 00 eb 00 48 81 7c 24 10 00 e1 f5  $.......H.|$....
	0x0030 05 7c 02 eb 21 48 8b 44 24 10 48 89 04 24 66 90  .|..!H.D$.H..$f.
	0x0040 e8 00 00 00 00 eb 00 48 8b 44 24 10 48 ff c0 48  .......H.D$.H..H
	0x0050 89 44 24 10 eb d2 48 8b 6c 24 18 48 83 c4 20 90  .D$...H.l$.H.. .
	0x0060 c3 e8 00 00 00 00 eb 98                          ........
	rel 5+4 t=17 TLS+0
	rel 65+4 t=8 "".a+0
	rel 98+4 t=8 runtime.morestack_noctxt+0
"".a STEXT nosplit size=23 args=0x10 locals=0x0 funcid=0x0
	0x0000 00000 (func_cost.go:11)	TEXT	"".a(SB), NOSPLIT|ABIInternal, $0-16
	0x0000 00000 (func_cost.go:11)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (func_cost.go:11)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (func_cost.go:11)	MOVQ	$0, "".~r1+16(SP)
	0x0009 00009 (func_cost.go:12)	MOVQ	"".a+8(SP), AX
	0x000e 00014 (func_cost.go:12)	SHLQ	$1, AX
	0x0011 00017 (func_cost.go:12)	MOVQ	AX, "".~r1+16(SP)
	0x0016 00022 (func_cost.go:12)	RET
	0x0000 48 c7 44 24 10 00 00 00 00 48 8b 44 24 08 48 d1  H.D$.....H.D$.H.
	0x0010 e0 48 89 44 24 10 c3                             .H.D$..
go.cuinfo.packagename. SDWARFCUINFO dupok size=0
	0x0000 6d 61 69 6e                                      main
""..inittask SNOPTRDATA size=24
	0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 00 00 00 00 00 00 00 00                          ........
gclocals·33cdeccccebe80329f1fdbee7f5874cb SRODATA dupok size=8
	0x0000 01 00 00 00 00 00 00 00                          ........
```

可以看到，在Golang中：

**使用栈来传递参数**
栈是位于内存之中的，虽然有 CPU 中 L1、L2、L3的帮助，但平均每次访问性能仍然和寄存器没法比。所以 Golang 的函数调用开销肯定会比 C 语言要高。后面我们将用一个实验来进行量化的比较。

**使用栈来返回数据**
不像 C 语言那样固定使用一个 eax 寄存器，Golang 是使用栈来返回值的。这就是为啥 Golang 可以返回多个值的根本原因。



### 性能对比

我们的测试方法简单粗暴，直接调用空函数 1 亿次，再统计计算平均耗时。

#### **C函数编译运行测试：**

```
# gcc func_main.s -o main
# time ./main
```

第一次执行耗时大约是 0.339 s。

但这个耗时中包含了两块。一块是函数调用开销，另外一块是 for 循环的开销(其它的代码调用因为只有 1 次，而函数调用和 for 循环都有 1 亿次，所以直接就可以忽略了)。

所以我们得减去 for 循环的开销。接着我手工注释掉对函数的调用，只是空循环 100000000 次。

```
int a(int p) {
    return 2*p;
}

int main() {
    int i;
    for (i = 0; i < 100000000; i++) {
//        a(i);
    }
    return 0;
}
```

这次总耗时是 0.314 s。

这样就计算出平均每次函数调用耗时 = (0.339s - 0.314s) / 100000000 = 0.25ns



#### **Golang函数编译运行**

```
# go build -gcflags="-m -l" func_cost.go
```

同样采用上述方法测出平均每次函数调用耗时 = (0.302s - 0.056 s) / 100000000 = 2.46ns

可见 Golang 的函数调用性能还是比 C 要差一些。

> 如果禁止 -N: 禁止编译优化 就会更慢些
