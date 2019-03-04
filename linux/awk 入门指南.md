本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

awk是处理文本文件的一个应用程序，几乎所有 Linux 系统都自带这个程序。

它依次处理文件的每一行，并读取里面的每一个字段。对于日志、CSV 那样的每行格式相同的文本文件，awk可能是最方便的工具。

awk其实不仅仅是工具软件，还是一种编程语言。不过，本文只介绍它的命令行用法，对于大多数场合，应该足够用了。



### **一、基本用法**

awk的基本用法就是下面的形式。

> \# 格式
>
> $ awk 动作 文件名
>
>  
>
> \# 示例
>
> $ awk '{print $0}' demo.txt

上面示例中，demo.txt是awk所要处理的文本文件。前面单引号内部有一个大括号，里面就是每一行的处理动作print $0。其中，print是打印命令，$0代表当前行，因此上面命令的执行结果，就是把每一行原样打印出来。

awk也可以处理标准输入（stdin）。

> $ echo 'this is a test' | awk '{print $0}'
>
> this is a test

上面代码中，print $0就是把标准输入this is a test，重新打印了一遍。

awk会根据空格和制表符，将每一行分成若干字段，依次用$1、$2、$3代表第一个字段、第二个字段、第三个字段等等。

> $ echo 'this is a test' | awk '{print $3}'
>
> a

上面代码中，$3代表this is a test的第三个字段a。

下面，为了便于举例，我们把/etc/passwd文件保存成demo.txt。

![img](https://mmbiz.qpic.cn/mmbiz_png/9aPYe0E1fb3ueIFmEgu11rPEtPgPRNiaKiakdQuA3P9AFtgOSPEwUgamibpj6vllQEeLp5ZeEad6bYiaCAJQY6KRlA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这个文件的字段分隔符是冒号（:），所以要用-F参数指定分隔符为冒号。然后，才能提取到它的第一个字段。

> $ awk -F ':' '{ print $1 }' demo.txt
>
> root
>
> daemon
>
> bin
>
> sys
>
> sync



### 二、变量

除了$ + 数字表示某个字段，awk还提供其他一些变量。

变量NF表示当前行有多少个字段，因此$NF就代表最后一个字段。

> $ echo 'this is a test' | awk '{print $NF}'
>
> test

$(NF-1)代表倒数第二个字段。

> $ awk -F ':' '{print $1, $(NF-1)}' demo.txt
>
> root /root
>
> daemon /usr/sbin
>
> bin /bin
>
> sys /dev
>
> sync /bin

上面代码中，print命令里面的逗号，表示输出的时候，两个部分之间使用空格分隔。

变量NR表示当前处理的是第几行。

> $ awk -F ':' '{print NR ") " $1}' demo.txt
>
> 1) root
>
> 2) daemon
>
> 3) bin
>
> 4) sys
>
> 5) sync

上面代码中，print命令里面，如果原样输出字符，要放在双引号里面。

awk的其他内置变量如下。

- FILENAME：当前文件名
- FS：字段分隔符，默认是空格和制表符。
- RS：行分隔符，用于分割每一行，默认是换行符。
- OFS：输出字段的分隔符，用于打印时分隔字段，默认为空格。
- ORS：输出记录的分隔符，用于打印时分隔记录，默认为换行符。
- OFMT：数字输出的格式，默认为％.6g。

## 

### **三、函数**

awk还提供了一些内置函数，方便对原始数据的处理。

函数toupper()用于将字符转为大写。

> $ awk -F ':' '{ print toupper($1) }' demo.txt
>
> ROOT
>
> DAEMON
>
> BIN
>
> SYS
>
> SYNC

上面代码中，第一个字段输出时都变成了大写。

其他常用函数如下。

- tolower()：字符转为小写。
- length()：返回字符串长度。
- substr()：返回子字符串。
- sin()：正弦。
- cos()：余弦。
- sqrt()：平方根。
- rand()：随机数。

awk内置函数的完整列表，可以查看手册。



### 四、条件

awk允许指定输出条件，只输出符合条件的行。

输出条件要写在动作的前面。

> $ awk '条件 动作' 文件名

 

> $ awk -F ':' '/usr/ {print $1}' demo.txt
>
> root
>
> daemon
>
> bin
>
> sys

上面代码中，print命令前面是一个正则表达式，只输出包含usr的行。

下面的例子只输出奇数行，以及输出第三行以后的行。

> \# 输出奇数行
>
> $ awk -F ':' 'NR % 2 == 1 {print $1}' demo.txt
>
> root
>
> bin
>
> sync
>
>  
>
> \# 输出第三行以后的行
>
> $ awk -F ':' 'NR >3 {print $1}' demo.txt
>
> sys
>
> sync

下面的例子输出第一个字段等于指定值的行。

> $ awk -F ':' '$1 == "root" {print $1}' demo.txt
>
> root
>
>  
>
> $ awk -F ':' '$1 == "root" || $1 == "bin" {print $1}' demo.txt
>
> root
>
> bin



### 五、if 语句

awk提供了if结构，用于编写复杂的条件。

> $ awk -F ':' '{if ($1 > "m") print $1}' demo.txt
>
> root
>
> sys
>
> sync

上面代码输出第一个字段的第一个字符大于m的行。

if结构还可以指定else部分。

> $ awk -F ':' '{if ($1 > "m") print $1; else print "---"}' demo.txt
>
> root
>
> \---
>
> \---
>
> sys
>
> sync



##### 参考

www.ruanyifeng.com/blog/2018/11/awk.html