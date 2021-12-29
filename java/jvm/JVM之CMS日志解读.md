本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 一、背景

趁着最近有时间，总结手上业务线发生一些问题，特别是HBase在前期使用的时候经常报与CMS有关的问题，

虽然理论学了好几轮次，但真正实操起来，也难了然于胸，按部就班代价太高了；

HBase前期使用JVM CMS，发生了碎片过多导致的Full GC；

对此，特定研究下CMS大体复现、日志解读；



### 二、CMS Full GC 复现

CMS GC 的 6个过程:

- 1.初始标记：为了收集应用程序的对象引用需要暂停应用程序线程，该阶段完成后，应用程序线程再次启动。
- 2.并发标记：从第一阶段收集到的对象引用开始，遍历所有其他的对象引用。
- 3.并发预清理：改变当运行第二阶段时，由应用程序线程产生的对象引用，以更新第二阶段的结果。
- 4.重标记：由于第三阶段是并发的，对象引用可能会发生进一步改变。因此，应用程序线程会再一次被暂停以更新这些变化，并且在进行实际的清理之前确保一个正确的对象引用视图。这一阶段十分重要，因为必须避免收集到仍被引用的对象。
- 5.并发清理：所有不再被应用的对象将从堆里清除掉。
- 6.并发重置：收集器做一些收尾的工作，以便下一次GC周期能有一个干净的状态。

其中4个阶段(名字以Concurrent开始的)与实际的应用程序是并发执行的，

而其他2个阶段需要暂停应用程序线程(STW).



核心[代码](https://github.com/Zeb-D/my-java-api/blob/master/learn-java/src/main/java/com/yd/java/java8/CMSGcTest.java)如下：

```java
    /**
     * VM arg： -Xms100m(设置最大堆内存) -Xmx100m(设置初始堆内存)
     * -Xmn50m(设置新生代大小)
     * -XX:+PrintGCDetails(打印GC日志详细信息)
     * -XX:+UseConcMarkSweepGC (采用 cms gc算法)
     * -XX:+UseParNewGC (新生代采用并行GC方式,
     * 高版本的jdk使用了UseConcMarkSweepGC参数时 这个参数会自动开启)
     * -XX:SurvivorRatio=8 (新生代eden区与survivor区空间比例8:1,
     * eden:fromsurvivor:tosurvivor -->8:1:1)
     * -XX:MaxTenuringThreshold=1 (用于控制对象能经历多少次
     * Minor GC(young gc)才晋升到老年代,默认15次)
     * -XX:+PrintTenuringDistribution(输出survivor区幸存对象的年龄分布)
     * -XX:CMSInitiatingOccupancyFraction=68 *(设置老年代空间使用率多少时触发第一次cms *gc,默认68%)
     *
     * @throws InterruptedException
     */
    public static void test() throws InterruptedException {
        List<byte[]> list = new ArrayList<>();
        for (int n = 1; n < 8; n++) {
            byte[] alloc = new byte[_10MB];
            list.add(alloc);
        }
        Thread.sleep(Integer.MAX_VALUE);
    }
```

开启方式：

> 串行收集器：
> DefNew：是使用-XX:+UseSerialGC（新生代，老年代都使用串行回收收集器）。
> 
> 并行收集器：ParNew：是使用-XX:+UseParNewGC（新生代使用并行收集器，老年代使用串行回收收集器）或者
>
> -XX:+UseConcMarkSweepGC(新生代使用并行收集器，老年代使用CMS)。
>
> PSYoungGen：是使用-XX:+UseParallelOldGC（新生代，老年代都使用并行回收收集器）或者
>
> -XX:+UseParallelGC（新生代使用并行回收收集器，老年代使用串行收集器）
>
> garbage-first heap：是使用-XX:+UseG1GC（G1收集器）





### CMS GC 日志

完整日志如下：

```java
➜  learn-java git:(master) ✗ java -Xms100m -Xmx100m -Xmn50m -XX:+PrintGCDetails 
-XX:+UseConcMarkSweepGC -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=1 
-XX:+PrintTenuringDistribution GCTest
[GC (Allocation Failure) 
[ParNew Desired survivor size 2621440 bytes, new threshold 1 (max 1)
- age   1:     636784 bytes,     636784 total: 33997K->632K(46080K), 0.0105313 secs] 
33997K->31354K(97280K), 0.0107013 secs]
[Times: user=0.01 sys=0.00, real=0.01 secs]
//step 1,young gc,新生代没有空间存入新对象,则发生young gc(也叫 minor gc),
//33997K->632K(46080K), 0.0105313 secs]新生代由33997K gc后变成了632K,
//整个heap区情况: 33997K->31354K(97280K)
//新生代的对象copy(新生代采用的是复制算法收集内存)到survivor区,
//但此处survivor容纳不下,则直接copy到了老年代中大小(可以设置多大的对象直接升级到老年代中,
//参数XX:PretenureSizeThreshold=<value>)
[GC (Allocation Failure) [ParNew: 32961K->32961K(46080K), 0.0001067 secs]
[CMS: 30722K->40960K(51200K), 0.0111639 secs] 63683K->62060K(97280K), 
[Metaspace: 2509K->2509K(1056768K)], 0.0118014 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
//step 2,老年代空间占用大小占比达到阈值,触发第一次老年代GC,即CMS GC
//CMS: 30722K->40960K(51200K): 老年代由30722K变为40960K,总大小51200K
[GC (CMS Initial Mark) [1 CMS-initial-mark: 40960K(51200K)] 72995K(97280K), 0.0004270 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
//step 3 初始标记
[CMS-concurrent-mark-start]
[CMS-concurrent-mark: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
//step 4 并发标记
[CMS-concurrent-preclean-start]
[CMS-concurrent-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
//step 5 并发预清理
[CMS-concurrent-abortable-preclean-start]
[CMS-concurrent-abortable-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
[GC (CMS Final Remark) [YG occupancy: 32035 K (46080 K)][Rescan (parallel) , 0.0003930 secs]
//step 6重标记 
[weak refs processing, 0.0000436 secs][class unloading, 0.0001861 secs]
[scrub symbol table, 0.0003188 secs]
[scrub string table, 0.0000946 secs]
//weak refs processing 处理old区的弱引用，用于回收native memory
class unloading 回收SystemDictionary
[1 CMS-remark:40960K(51200K)] 72995K(97280K), 0.0013492 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
[CMS-concurrent-sweep-start]
[CMS-concurrent-sweep: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
//step 7 并发清理
[CMS-concurrent-reset-start]
[CMS-concurrent-reset: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
//step 8 并发重置
Heap
 par new generation   total 46080K, used 32445K [0x00000000f9c00000, 0x00000000fce00000, 0x00000000fce00000)
  eden space 40960K,  79% used [0x00000000f9c00000, 0x00000000fbbaf5b0, 0x00000000fc400000)
  from space 5120K,   0% used [0x00000000fc900000, 0x00000000fc900000, 0x00000000fce00000)
  to   space 5120K,   0% used [0x00000000fc400000, 0x00000000fc400000, 0x00000000fc900000)
 concurrent mark-sweep generation total 51200K, used 40960K [0x00000000fce00000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 2515K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 269K, capacity 386K, committed 512K, reserved 1048576K
```



#### 日志解读

基本上都是这种格式：回收前区域占用的大小->回收后区域占用的大小（区域设置的大小）、耗时.

现在取step2 这部分日志做一个全面的分析作为解读GC日志的典型:

```
[GC (Allocation Failure) [ParNew: 32961K->32961K(46080K), 0.0001067 secs]
[CMS: 30722K->40960K(51200K), 0.0111639 secs] 63683K->62060K(97280K), 
[Metaspace: 2509K->2509K(1056768K)], 0.0118014 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
```

```
[ParNew: 32961K->32961K(46080K), 0.0001067 secs]
```

ParNew 表明新生代是并行方式进行GC,

32961K->32961K(46080K) 新生代大小由32961K变成32961K,总大小为46080K,young gc 耗时 0.0001067 secs.



```
[CMS: 30722K->40960K(51200K), 0.0111639 secs]
```

CMS:表明是老年代堆区(因为CMS只作用于老年代),



```
 63683K->62060K(97280K)
```

heap 区情况，



```json
[Metaspace: 2509K->2509K(1056768K)]
```

Metaspace :元空间是jdk 8 的新特性,可简单理解为其替换了8 之前的永久代PermGen Space(方法区).

方法区是JVM的规范,而永久代则是JVM规范的一种实现,元空间也是方法区的一种实现.



```
[Times: user=0.02 sys=0.00, real=0.02 secs]
```

这里的user,sys,real 与 linux的time 命令所输出的时间含义一致,

分别代表用户太消耗的CPU时间、内核态消耗的CPU时间、操作从开始到结束所经过的墙钟时间；





发现了一个有趣的东西,就是`concurrent mode failure`情况,当我new 出的对象超过了堆内存的最大内存后会发生OOM时,就出现`concurrent mode failure`,GC日志如下:

```
[GC [ParNew: 31539K->496K(46080K), 0.0137601 secs] 31539K->31218K(97280K), 0.0152885 secs] 
[Times: user=0.02 sys=0.00, real=0.02 secs]
[GC [ParNew: 33275K->33275K(46080K), 0.0003866 secs][CMS: 30722K->40960K(51200K), 0.0217084 secs]
63997K->61912K(97280K), [CMS Perm : 2534K->2532K(21248K)], 0.0231226 secs] 
[Times: user=0.02 sys=0.00, real=0.02 secs]
[GC [1 CMS-initial-mark: 40960K(51200K)] 72152K(97280K), 0.0004554 secs]
 [Times: user=0.00 sys=0.00, real=0.00 secs]
[Full GC [CMS[CMS-concurrent-mark: 0.010/0.011 secs] 
[Times: user=0.00 sys=0.00, real=0.01 secs]
 (concurrent mode failure): 40960K->40960K(51200K), 0.0159043 secs] 72152K->72152K(97280K), [CMS Perm : 2532K->2532K(21248K)], 0.0165056 secs] [Times: user=0.01 sys=0.00, real=0.02 secs]
[Full GC [CMS: 40960K->40960K(51200K), 0.0075621 secs] 
72152K->72138K(97280K), [CMS Perm : 2532K->2532K(21248K)], 0.0084089 secs]
[Times: user=0.00 sys=0.00, real=0.01 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
        at GCTest.test(GCTest.java:54)
        at GCTest.main(GCTest.java:32)
```

造成这种情况的原因是:

在执行CMS GC的过程中同时有对象要放入旧生代，而此时老年代空间不足造成的。

同样的情况还有promotion failed这种情形,这是由于在进行Minor GC(young gc)时，survivor space放不下、对象只能放入老年代，而此时老年代也放不下造成的.
