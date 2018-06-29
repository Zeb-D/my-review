# JVM常用命令

<br>

jdk本身提供了很多方便的jvm性能调优工具，

除了集成式的visualVM和jconsole外，还有jps,jstack,jmap,jhat,jstat等工具。

<br>

**jps**（java virtual machine process status tool）

 jps主要用来输出jvm中运行的进程的状态信息。语法格式如下：

```java
 jps [options] [hostid]
```

如果不指定hostid就默认为当前主机或者服务器 命令行参数如下:

```java
  -q 不输出类名，jar名和传入main方法的参数
  -m 输出传入main方法的参数
  -l 输出main类或jar的全名
  -v 输出传入jvm的参数
```

例如:

```java
basis-study git:(vine_leraning) ✗ jps -m -l
84294 org.jetbrains.jps.cmdline.Launcher /Applications/IntelliJ IDEA 15 CE.app/Contents/lib/jna-platform.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/snappy-in-java-0.3.1.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/picocontainer.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/netty-all-4.1.0.Beta8.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/oromatcher.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/annotations.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/jdom.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/jgoodies-forms.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/javac2.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/idea_rt.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/resources_en.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/protobuf-2.5.0.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/rt/jps-plugin-system.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/asm-all.jar:/Applications/IntelliJ IDEA 15 CE.
12187 com.intellij.rt.execution.application.AppMain com.weibo.motan.benchmark.MotanBenchmarkServer
12188 org.jetbrains.jps.cmdline.Launcher /Applications/IntelliJ IDEA 15 CE.app/Contents/lib/jna-platform.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/snappy-in-java-0.3.1.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/picocontainer.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/netty-all-4.1.0.Beta8.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/oromatcher.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/annotations.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/jdom.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/jgoodies-forms.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/javac2.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/idea_rt.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/resources_en.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/protobuf-2.5.0.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/rt/jps-plugin-system.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/asm-all.jar:/Applications/IntelliJ IDEA 15 CE.
52781 sun.tools.jps.Jps -m -l
255

basis-study git:(vine_leraning) ✗ jps -m  
84294 Launcher /Applications/IntelliJ IDEA 15 CE.app/Contents/lib/jna-platform.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/snappy-in-java-0.3.1.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/picocontainer.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/netty-all-4.1.0.Beta8.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/oromatcher.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/annotations.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/jdom.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/jgoodies-forms.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/javac2.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/idea_rt.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/resources_en.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/protobuf-2.5.0.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/rt/jps-plugin-system.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/asm-all.jar:/Applications/IntelliJ IDEA 15 CE.
12187 AppMain com.weibo.motan.benchmark.MotanBenchmarkServer
12188 Launcher /Applications/IntelliJ IDEA 15 CE.app/Contents/lib/jna-platform.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/snappy-in-java-0.3.1.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/picocontainer.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/netty-all-4.1.0.Beta8.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/oromatcher.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/annotations.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/jdom.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/jgoodies-forms.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/javac2.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/idea_rt.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/resources_en.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/protobuf-2.5.0.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/rt/jps-plugin-system.jar:/Applications/IntelliJ IDEA 15 CE.app/Contents/lib/asm-all.jar:/Applications/IntelliJ IDEA 15 CE.
255 
54447 Jps -m 
```

<br>

<br>

## jstack 

jstack主要用来查看某个java进程内的线程堆栈信息，语法格式如下:

```java
jstack [option] pid
```

命令行参数如下:

```java
-l long listings，会显示出额外的锁信息，在发生死锁是可以使用jstack -l pid 来观察锁持有的情况
-m mixed mode,不仅会输出java堆栈信息，还会输出c/c++堆栈信息(比如Native方法)
```

jstack还可以用来定位到线程堆栈，根据堆栈信息我们可以定位到具体的代码，所以它在jvm性能调优中使用的非常多。

<br><br>

## jmap（java memory map）

## jhat(java heap analysis tool)

jmap用来查看堆内存使用情况，一般结合jhat使用 jmap的语法格式如下：

```java
jmap [option] pid
```

使用jmap -heap pid查看进程内的堆内存使用情况，包括使用的gc算法，堆配置参数和各代中堆内存的使用情况：

```java
basis-study git:(vine_leraning) ✗ jmap -heap 12187
Attaching to process ID 12187, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.73-b02

using thread-local object allocation.
Parallel GC with 4 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 2147483648 (2048.0MB)
   NewSize                  = 44564480 (42.5MB)
   MaxNewSize               = 715653120 (682.5MB)
   OldSize                  = 89653248 (85.5MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 34078720 (32.5MB)
   used     = 26546424 (25.31664276123047MB)
   free     = 7532296 (7.183357238769531MB)
   77.89736234224759% used
From Space:
   capacity = 5242880 (5.0MB)
   used     = 3910600 (3.7294387817382812MB)
   free     = 1332280 (1.2705612182617188MB)
   74.58877563476562% used
To Space:
   capacity = 5242880 (5.0MB)
   used     = 0 (0.0MB)
   free     = 5242880 (5.0MB)
   0.0% used
PS Old Generation
   capacity = 89653248 (85.5MB)
   used     = 81936 (0.0781402587890625MB)
   free     = 89571312 (85.42185974121094MB)
   0.09139211554276316% used

5611 interned Strings occupying 419096 bytes.
```

<br>

使用jmap -histo[:live] pid查看堆内存中的对象的数目，大小统计直方图，如果带上live则只统计活对象。

```java
basis-study git:(vine_leraning) ✗ jmap -histo:live 12187 | more

 num     #instances         #bytes  class name
----------------------------------------------
   1:          9644         732488  [C
   2:          2507         283720  java.lang.Class
   3:           533         248064  [B
   4:          9552         229248  java.lang.String
   5:          1692         106928  [Ljava.lang.Object;
   6:          2540          81280  java.util.concurrent.ConcurrentHashMap$Node
   7:          1177          55056  [I
   8:          1427          45664  java.util.HashMap$Node
   9:          2272          36352  java.lang.Object
  10:            81          35824  [Ljava.util.concurrent.ConcurrentHashMap$Node;
  11:          1083          34656  java.lang.ref.WeakReference
  12:           682          27280  java.lang.ref.SoftReference
  13:           226          24904  [Ljava.util.HashMap$Node;
  14:           615          23456  [Ljava.lang.String;
  15:            99          19800  com.sun.org.apache.xerces.internal.impl.dv.xs.XSSimpleTypeDecl
  16:           379          15160  java.util.LinkedHashMap$Entry
  17:           225          12600  java.beans.MethodDescriptor
  18:           366          11712  java.util.Hashtable$Entry
  19:            31          11656  java.lang.Thread
:
```

其中class name是对象类型:

```java
B byte
C char
D double
F float
I int
J long
Z boolean
[ 数组，如[I表示int []
[L+类名 其他对象
```

用jmap把进程内存使用情况dump到文件中，再用jhat分析查看**，jmap进行dump**的命令格式如下：

```java
jmap -dump:format=b,file=dumpFileName
```

例如

```java
 basis-study git:(vine_leraning) ✗ jmap -dump:format=b,file=/tmp/dump.dat 12187
Dumping heap to /private/tmp/dump.dat ...
Heap dump file created
```

dump出来的文件可以用MAT,VisualVM来分析，这里使用jhat来查看。