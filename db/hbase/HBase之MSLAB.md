本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 一、背景

HBase作为NoSQL的一族，也支撑了很多的业务场景，HBase支撑的数据量也是能达T级别的，虽然最终保存是在HDFS中，但也做了很多的优化如不同维度的缓存；换而言之，也是比较吃JVM内存，特别是高并发场景下GC压力特别高，故在优化道路越走越惊心；

目前作者对一些IOT设备日志(它包括下行日志、上行日志)、一些其他方面的日志业务会采用HBase存储，因为普通的RDMS是承受不住的：

```
它的QPS是非常大的，目前HBase集群是要支撑住亿级设备的日志，平均每个设备会定时5s上报自己的状态，下发控制日志一般需要人为触发(下发控制成功也会立马一条状态上报)，一般高峰占用也不过20%资源；
真正的使用这方面的日志请求却不到5%，包括大数据分析、ToB企业通过OpenAPI load 备份；
当然，这方面的存储不是长久有效的，当前最多支持7天有效期；
```

面对如此多的存储，Hbase采用多级索引方式+cache方式进行支持HDFS的crud；HBase的regionServer的内存配置往往是32G以上，是个IO的存储；

加上之前采用JVM GC是CMS，也出现了一些CMS自身的问题；



### 二、CMS GC分析

CMS是在jkd7之前就有了，GC这种玩意不止java有这，只不过Java就业人多；

#### GC的一些概念

##### GC回收算法：

1、引用计数法；如python

2、可达性分析；如大多数的jvm、go

##### GC实现：

1、复制

2、标记清除

3、标记压缩

##### 回收方式：

1、串行

2、并行

3、并发

##### 内存管理：

1、代管理；

2、非代管理；如jvm的zgc



那么CMS对应是什么？

首先它肯定是可达性分析，也肯定是代管理，如eden、from servior to，old 、Metaspace..

GC实现的话，新生代是采用复制，from to两个空间进行拷贝，然后其中一个空间在young gc(也叫 minor gc)进行清除；

回收方式大多数情况是并行+并发，cms作用在老年代，当然也有STW 情况是串行；

```
1.初始标记：为了收集应用程序的对象引用需要暂停应用程序线程，该阶段完成后，应用程序线程再次启动。
2.并发标记：从第一阶段收集到的对象引用开始，遍历所有其他的对象引用。
3.并发预清理：改变当运行第二阶段时，由应用程序线程产生的对象引用，以更新第二阶段的结果。
4.重标记：由于第三阶段是并发的，对象引用可能会发生进一步改变。因此，应用程序线程会再一次
被暂停以更新这些变化，并且在进行实际的清理之前确保一个正确的对象引用视图。
这一阶段十分重要，因为必须避免收集到仍被引用的对象。
5.并发清理：所有不再被应用的对象将从堆里清除掉。
6.并发重置：收集器做一些收尾的工作，以便下一次GC周期能有一个干净的状态。

其中4个阶段(名字以Concurrent开始的)与实际的应用程序是并发执行的，
而其他2个阶段需要暂停应用程序线程(STW).
```



#### CMS GC有哪些缺点

虽然加了并发+并行GC，但GC耗时随着堆内存大小是线性增长的，越大的堆耗时增加8~10秒/G（官网报告的数据），但有人会问，既然是Concurrent GC，为什么还会出现STW，影响业务请求？

一般和老年代的Full GC有关系；

##### 1、同步模式失败（concurrent mode failure）

在CMS还没有把垃圾收集完的时候空间还没有完全释放，而这个时候如果新生代的对象过快地转化为老生代的对象时发现老生代的可用空间不够了。

此时收 集器会停止并发收集过程，转为单线程的STW（Stop The World）暂停，这就又回到了Full GC的过程了。 不过这个过程可以通过设置XX:CMSInitiatingOccupancyFraction=N来缓解。

N代表了当JVM启动垃 圾回收时的堆内存占用百分比。你设置的越小，JVM越早启动垃圾回收进程，一般设置为70。



##### 2、由于碎片化造成的失败（Promotion Failure due to Fragmentation）

当前要从新生代提升到老年代的对象比老年代的所 有可以使用的连续的内存空间都大。

比如你当前的老年代里面有500MB 的空间是可以用的，但是都是1KB大小的碎片空间，现在有一个2KB的对 象要提升为老年代却发现没有一个空间可以插入。

这时也会触发STW暂停，进行Full GC。 这个问题无论你把-XX:CMSInitiatingOccupancyFraction=N调多小都是无法解决的，因为CMS只做回收不做合并；



##### 3、`concurrent mode failure` 与`Promotion Failure due to Fragmentation`的区别：

- 一个是一边收集一边分配，内存不够导致的；所以要暂停分配内存，全心回收；
- 一个是内存够了，但是连续空间内存却没找到；所以要停下来重新排列；



### HBase是如何GC优化

#### HBase Full GC行为

避免不了Full GC，那就尽量减少Full GC的频率，那为什么HBase除了会发生同步模式失败（concurrent mode failure）问题，也会发生由于碎片化造成的失败（Promotion Failure due to Fragmentation）？

这个问题无论你把-XX:CMSInitiatingOccupancyFraction=N调多小都是无法解决的，

因为CMS只做回收不做合并，所以只要你的 RegionServer启动得够久一定会遇上Full GC。 

很多人可能会疑惑为什么会出现碎片内存空间？ 

我们知道Memstore是会定期刷写成为一个HFile的，在刷写的同时 这个Memstore所占用的内存空间就会被标记为待回收，一旦被回收了， 这部分内存就可以再次被使用，但是由于JVM分配对象都是按顺序分配下去的；

 JVM也没有办法，为了不让情况继续地恶化下去，只好停止接收一 切请求，然后启用一个单独的进程来进行内存空间的重新排列。

这个排 列的时间随着内存空间的增大而增大，当内存足够大的时候，暂停的时 间足以让ZooKeeper认为我们的RegionServer已死。  



#### Full GC优化

##### MSLAB（Memstore-Local Allocation Buffers）

有一个基于线程的解决方案，叫 TLAB（Thread-Local allocation buffer）。当你使用TLAB的时候，每 一个线程都会分配一个固定大小的内存空间，专门给这个线程使用，当 线程用完这个空间后再新申请的空间还是这么大，这样下来就不会出现 特别小的碎片空间，基本所有的对象都可以有地方放。

缺点就是无论你 的线程里面有没有对象都需要占用这么大的内存，其中有很大一部分空 间是闲置的，内存空间利用率会降低。不过能避免Full GC，这些都是 值得的。 

 但是HBase不能直接使用这个方案，因为在HBase中多个Region是被 一个线程管理的，多个Memstore占用的空间还是无法合理地分开。



 **MSLAB的引入**

于是 HBase就自己实现了一套以Memstore为最小单元的内存管理机制，称为 MSLAB（Memstore-Local Allocation Buffers）。这套机制完全沿袭了 TLAB的实现思路，只不过内存空间是由Memstore来分配的。



#####  **MSLAB的实现**

MSLAB，全称是 MemStore-Local Allocation Buffer，是Cloudera在HBase 0.90.1时提交的一个patch里包含的特性。它基于Arena Allocation解决了HBase因Region flush导致的内存碎片问题。

**Arena Allocation**

Arena Allocation是一种非传统的内存管理方法。它通过顺序化分配内存（池化技术），内存数据分块等特性使内存碎片粗化，有效改善了内存碎片导致的Full GC问题。

它的原理：

- 创建一个大小固定的bytes数组和一个偏移量，默认值为0。
- 分配对象时，将新对象的data bytes复制到数组中，数组的起始位置是偏移量，复制完成后为偏移量自增data.length的长度，这样做是防止下次复制数据时不会覆盖掉老数据（append）。
- 当一个数组被充满时，创建一个新的数组。
- 清理时，只需要释放掉这些数组，即可得到固定的大块连续内存。



**MSLAB的实现原理**：

- MemstoreLAB为Memstore提供Allocator。
- 创建一个2M（默认）的Chunk数组和一个chunk偏移量，默认值为0。
- 当Memstore有新的KeyValue被插入时，通过KeyValue.getBuffer()取得data bytes数组。将data复制到Chunk数组起始位置为chunk偏移量处，并增加偏移量=偏移量+data.length。
- 当一个chunk满了以后，再创建一个chunk。
- 所有操作lock free，基于CMS原语。

优势：

- KeyValue原始数据在minor gc时被销毁。
- 数据存放在2m大小的chunk中，chunk归属于memstore。
- flush时，只需要释放多个2m的chunks，chunk未满也强制释放，从而为Heap腾出了多个2M大小的内存区间，减少碎片密集程度。



由此可以看出堆内存被chunk区分为规则的空间，这样就消除了小 碎片引起的无法插入数据问题，但是会降低内存利用率，因为就算你的 chunk里面只放1KB的数据，这个chunk也要占2MB的大小。不过，为了不发生Full GC，这些都可以忍（空间换时间，只不过这也算时间）。



##### MSLAB相关的参数

- **hbase.hregion.memstore.mslab.enabled**：设置为true，即打开 MSLAB，默认为true。      
-  **hbase.hregion.memstore.mslab.chunksize**：每个chunk的大 小，默认为2048 * 1024 即2MB。    
- **hbase.hregion.memstore.mslab.max.allocation**：能放入chunk 的最大单元格大小，默认为256KB，已经很大了。  
-  **hbase.hregion.memstore.chunkpool.maxsize**：在整个memstore 可以占用的堆内存中，chunkPool占用的比例。该值为一个百分 比，取值范围为0.0~1.0。默认值为0.0。
- **hbase.hregion.memstore.chunkpool.initialsize**：在 RegionServer启动的时候可以预分配一些空的chunk出来放到 chunkPool里面待使用。该值就代表了预分配的chunk占总的 chunkPool的比例。该值为一个百分比，取值范围为0.0~1.0，默 认值为0.0。



引入池化技术，避免了堆内存走高低峰路线，会维持一个稳定的内存占用；因为所有的空间已经被MSLAB的chunk所瓜分，所以可用空间的大小非常均匀。



#### G1GC和MSLAB

G1的出现是为了支撑更大的堆内存回收，对整个堆分成不同的区，新生代与老年代占用只不过区的数量不一样，

可以动态调整新生代大小（可以设置最小比例），老年代回收只会回收部分老年代分区（每个区垃圾占用情况排名），增量部分最后与新生代进行混合回收；

所以G1比CMS更能支持这种高内存的应用，也是更低停顿；



你可能会觉得G1GC跟MSLAB的实现思路非常接近，那为什么还要发明MSLAB策略呢？因为G1GC是MSLAB发明后才出现的策略。

HBase在JVM G1打开了MSLAB，会有哪些效果呢？

> HBase的PMC成员 ramkrishna.s.vasudevan曾做过一次测试：
>
> 测试条件：
>
> - RegionServer堆内存：64GB。
> - JVM回收策略：G1GC。
> - 工具：HBase的 org.apache.hadoop.hbase.PerformanceEvaluation，简称PE。
> - 批量插入150GB的数据，每行数据拥有10个列。 采用50、100线程分别测试插入时间和GC次数。

- 写入速度，提升了10%~12%，速度的提升 主要是因为减少了GC的次数，JVM的工作更有效率了。
- GC的次数，减少了16%左右；
- GC的时间综合，减少了21%；

