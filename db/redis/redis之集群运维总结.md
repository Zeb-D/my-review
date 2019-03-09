本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------



### 序

本篇文章只在总结平时线上关于redis发生一些有趣的事情总结，不单单是当前公司，也可是个人与别人交流过权且记录；

阅读本篇文章前，你可能需要了解一些关于[ redis 集群原理](./Redis之集群原理分析.md)，一些[redis备份方式](./redis持久化方式.md)等一些知识；

### 背景

比较流行的 Nosql 技术之一的redis，在集群方面实战方面的坑还是有比较多的，特别是 Redis Cluster 集群节点达到破500 近1000的情况下，这期间随着业务的发展，不管是业务开发师，还是devOps，都记录着这期间的汗与泪；

------

### **主从重同步问题**

#### **问题描述**

服务器宕机并恢复后，需要重启 Redis 实例，因为集群采用主从结构并且宕机时间比较长，此时宕机上的节点对应的节点都是主节点，宕掉的节点重启后都应该是从节点。启动 Redis 实例，我们通过日志发现节点一直从不断的进行主从同步。我们称这种现象为**主从重同步**。

#### **主从同步机制**

为了分析以上问题，我们首先应该搞清楚 Redis 的主从同步机制。以下是从节点正常的主从同步流程日志：

```
17:22:49.763 * MASTER <-> SLAVE sync started17:22:49.764 * Non blocking connect for SYNC fired the event.
17:22:49.764 * Master replied to PING, replication can continue...
17:22:49.764 * Partial resynchronization not possible (no cached master)
17:22:49.765 * Full resync from master: c9fabd3812295cc1436af69c73256011673406b9:174522475324717:23:42.223 * MASTER <-> SLAVE sync: receiving 1811656499 bytes from master17:24:04.484 * MASTER <-> SLAVE sync: Flushing old data17:24:25.646 * MASTER <-> SLAVE sync: Loading DB in memory17:27:21.541 * MASTER <-> SLAVE sync: Finished with success17:28:22.818 # MASTER timeout: no data nor PING received...
17:28:22.818 # Connection with master lost.
17:28:22.818 * Caching the disconnected master state.
17:28:22.818 * Connecting to MASTER xxx.xxx.xxx.xxx:xxxx17:28:22.818 * MASTER <-> SLAVE sync started17:28:22.819 * Non blocking connect for SYNC fired the event.
17:28:22.824 * Master replied to PING, replication can continue...
17:28:22.824 * Trying a partial resynchronization (request c9fabd3812295cc1436af69c73256011673406b9:1745240101942).
17:28:22.825 * Successful partial resynchronization with master.
```

**“Connection with master lost”**的字面意思是从节点与主节点的连接超时。在 Redis 中主从节点需要互相感知彼此的状态，这种感知是通过从节点定时 PING 主节点并且主节点返回 PONG 消息来实现的。那么当主节点或者从节点因为其他原因不能**及时**收到 PING 或者 PONG 消息时，则认为主从连接已经断开。

何为及时，Redis 通过参数 repl-timeout 来设定，它的默认值是 60s。Redis 配置文件（redis.conf）中详细解释了 repl-timeout 的含义：

> The following option sets the replication timeout for:
>
> #
>
> 1) Bulk transfer I/O during SYNC, from the point of view of slave.
>
> 2) Master timeout from the point of view of slaves (data, pings).# 
>
> 3) Slave timeout from the point of view of masters (REPLCONF ACK pings).## 
>
> It is important to make sure that this value is greater than the value# specified for repl-ping-slave-period otherwise a timeout will be detected# 
>
> every time there is low traffic between the master and the slave.## 
>
> repl-timeout 60

上边的同步日志，从节点加载 RDB 文件花费将近三分钟的时间，超过了 repl-timeout，所以从节点认为与主节点的连接断开，所以它尝试重新连接并进行主从同步。

#### **部分同步**

这里补充一点当进行主从同步的时候 Redis 都会先尝试进行部分同步，部分同步失败才会尝试进行全量同步。

Redis 中主节点接收到的每个写请求，都会写入到一个被称为 repl_backlog 的缓存空间中，这样当进行主从同步的时候，首先检查 repl_backlog 中的缓存是否能满足同步需求，这个过程就是部分同步。

考虑到全量同步是一个很重量级别并且耗时很长的操作，部分同步机制能在很多情况下极大的减小同步的时间与开销。

Redis 中主节点接收到的每个写请求，都会写入到一个被称为 repl_backlog 的缓存空间中，这样当进行主从同步的时候，首先检查 repl_backlog 中的缓存是否能满足同步需求，这个过程就是部分同步。

考虑到全量同步是一个很重量级别并且耗时很长的操作，部分同步机制能在很多情况下极大的减小同步的时间与开销。

#### **重同步问题**

通过上面的介绍大概了解了主从同步原理，我们在将注意力放在加载 RDB 文件所花费的三分钟时间上。在这段时间内，主节点不断接收前端的请求，这些请求不断的被加入到 repl_backlog 中，但是因为 Redis 的单线程特性，从节点是不能接收主节点的同步写请求的。所以不断有数据写入到 repl_backlog 的同时却没有消费。

当 repl_backlog 满的时候就不能满足部分同步的要求了，所以部分同步失败，需要又一次进行全量同步，如此形成无限循环，导致了主从重同步现象的出现。不仅侵占了带宽，而且影响主节点的服务。

#### **解决方案**

至此解决方案就很明显了，调大 repl_backlog。

Redis 中默认的 repl_backlog 大小为 1M，这是一个比较小的值，我们的集群中曾经设置为 10M，有时候还是会出现主从重同步现象，后来改为 100M，一切太平。可以通过以下命令修改 repl_backlog 的大小：

//200Mredis-cli -h xxx -p xxx config set repl-backlog-size 209715200



------

### **内存碎片**

#### 问题描述

首先对于绝大部分系统内存碎片是一定存在的。试想内存是一整块连续的区域，而数据的长度可以是任意的，并且会随时发生变化，随着时间的推移，在各个数据块中间一定会夹杂着小块的难以利用的内存，所以在 Redis 中内存碎片是存在的。

在 Redis 中通过 info memory 命令能查看内存及碎片情况：

```
# Memory
used_memory:4221671264          /* 内存分配器为数据分配出去的内存大小，可以认为是数据的大小 */
used_memory_human:3.93G         /* used_memoryd 的阅读友好形式 */
used_memory_rss:4508459008      /* 操作系统角度上 Redis 占用的物理内存空间大小，注意不包含 swap*/
used_memory_peak:4251487304     /* used_memory 的峰值大小 */
used_memory_peak_human:3.96G    /* used_memory_peak 的阅读友好形式 */
used_memory_lua:34816
mem_fragmentation_ratio:1.07    /* 碎片率 */
mem_allocator:jemalloc-3.6.0    /* 使用的内存分配器 */
```

对于每一项的意义请注意查看注释部分，也可以参考官网上 info 命令 memory 部分。Redis 中内存碎片计算公式为：

```
mem_fragmentation_ratio = used_memory_rss / used_memory
```

可以看出上边的 Redis 实例的内存碎片率为 1.07，是一个较小的值，这也是正常的情况，有正常情况就有不正常的情况。发生数据迁移之后的 Redis 碎片率会很高，以下是迁移数据后的 Redis 的碎片情况：

```
used_memory:4854837632
used_memory_human:4.52G
used_memory_rss:7362924544
used_memory_peak:7061034784
used_memory_peak_human:6.58G
used_memory_lua:39936
mem_fragmentation_ratio:1.52
mem_allocator:jemalloc-3.6.0
```

#### 解决方案

可以看到碎片率是 1.52，也就是说有三分之一的内存被浪费掉了。针对以上两种情况，对于碎片简单的分为两种：

- 常规碎片
- 迁移碎片

常规碎片数量较小，而且一定会存在，可以不用理会。那么如何去掉迁移碎片呢？

其实方案很简单，只需要先 BGSAVE 再重新启动节点，重新加载 RDB 文件会去除绝大部分碎片。

但是这种方案有较长的服务不可用窗口期，所以需要另一种较好的方案。

这种方案需要 Redis 采用主从结构为前提，主要思路是先通过重启的方式处理掉从节点的碎片，之后进行主从切换，最后处理老的主节点的碎。这样通过极小的服务不可用时间窗口为代价消除了绝大大部分碎片。

------

### **Redis Cluster 剔除节点失败**

#### 问题描述

Redis Cluster 采用无中心的集群模式，集群中所有节点通过互相交换消息来维持一致性。当有新节点需要加入集群时，只需要将它与集群中的一个节点建立联系即可，通过集群间节点互相交换消息所有节点都会互相认识。所以当需要剔除节点的时候，需要向所有节点发送 cluster forget 命令。

而向集群所有节点发送命令需要一段时间，在这段时间内已经接收到 cluster forget 命令的节点与没有接收的节点会发生信息交换，从而导致 cluster forget 命令失效。

#### 问题分析

为了应对这个问题 Redis 设计了一个黑名单机制。当节点接收到 cluster forget 命令后，不仅会将被踢节点从自身的节点列表中移除，还会将被剔除的节点添加入到自身的黑名单中。当与其它节点进行消息交换的时候，节点会忽略掉黑名单内的节点。所以通过向所有节点发送 cluster forget 命令就能顺利地剔除节点。

但是黑名单内的节点不应该永远存在于黑名单中，那样会导致被踢掉的节点不能再次加入到集群中，同时也可能导致不可预期的内存膨胀问题。所以黑名单是需要有时效性的，Redis 设置的时间为一分钟。

所以当剔除节点的时候，在一分钟内没能向所有节点发出 cluster forget 命令，会导致剔除失败，尤其在集群规模较大的时候会经常发生。

#### 解决方案

多个进程发送 cluster forget 命令，是不是很简单。



### **迁移数据时的 JedisAskDataException 异常**

#### **问题描述**

Redis Cluster 集群扩容，需要将一部分数据从老节点迁移到新节点。在迁移数据过程中会出现较多的 JedisAskDataException 异常。

#### **迁移流程**

由于官方提供迁移工具 redis-trib 在大规模数据迁移上的一些限制，我们自己开发了迁移工具，Redis Cluster 中数据迁移是以 Slot 为单位的，迁移一个 Slot 主要流程如下：

```
目标节点 cluster setslot <slot> importing <source_id>

源节点   cluster setslot <slot> migrating <target_id>

源节点   cluster getkeysinslot <slot> <count>  ==> keys 
源节点   migrate <target_ip> <target_port> <key> 0 <timeout>

重复 3&4 直到迁移完成 
任一节点 cluster setslot <slot> node <target_id>
```

我们使用 Redis 中的 MIGRATE 命令来把数据从一个节点迁移到另外一个节点。MIGRATE 命令实现机制是先在源节点上 DUMP 数据，再在目标节点上 RESTORE 它。

但是 DUMP 命令并不会包含过期信息，又因为集群中所有的数据都有过期时间，所以我们需要额外的设置过期时间。所以迁移一个 SLOT 有点类似如下：

```java
while (from.clusterCountKeysInSlot(slot) != 0) {   

    keys = from.clusterGetKeysInSlot(slot, 100);    
    for (String key : keys) {        
    // 获取 key 的 ttl        
    Long ttl = from.pttl(key);        
    if (ttl > 0) {    
            from.migrate(host, port, key, 0, 2000);            
            to.asking();            
            to.pexpire(key, ttl);        
            }    
     }
}
```

但是上边的迁移工具在运行过程中报了较多的 JedisAskDataException 异常，通过堆栈发现是“Long ttl = from.pttl(key)”这一行导致的。为了解释上述异常，我们需要先了解 Redis 的一些内部机制。

#### **Redis 数据过期机制**

我们继续复习一下以前了解[redis过期机制](./Redis基础、高级使用、调优.md#内存管理与数据淘汰机制)，其中关键如下：

edis 数据过期混合使用两种策略

- 主动过期策略：定时扫描过期表，并删除过期数据，注意这里并不会扫描整个过期表，为了减小任务引起的主线程停顿，每次只扫描一部分数据，这样的机制导致数据集中可能存在较多已经过期但是并没有删除的数据。
- 被动过期策略：当客户端访问数据的时候，首先检查它是否已经过期，如果过期则删掉它，并返回数据不存在标识。

这样的过期机制兼顾了每次任务的停顿时间与已经过期数据不被访问的功能性，充分体现了作者优秀的设计能力，详细参考官网数据过期机制。

#### **Open 状态 Slot 访问机制**

在迁移 Slot 的过程中，需要先在目标节点将 Slot 设置为 importing 状态，然后在源节点中将 Slot 设置为 migrating 状态，我们称这种 Slot 为 Open 状态的 Slot。

因为处于 Open 状态的 Slot 中的数据分散在源与目标两个节点上，所以如果需要访问 Slot 中的数据或者添加数据到 Slot 中，需要特殊的访问规则。Redis 推荐规则是首先访问源节点再去访问目标节点。如果源节点不存在，Redis 会返回 ASK 标记给客户端，详细参考官网。

#### **问题分析**

经过阅读 Redis 代码发现 clusterCountKeysInSlot 函数不会触发被动过期策略，所以它返回的数据包含已经过期但是没有被删除的数据。当程序执行到“Long ttl = from.pttl(key);”这一行时，首先 Redis 会触发触发被动过期策略删掉已经过期的数据，此时该数据已经不存在，又因为该节点处于 migrating 状态，所以 ASK 标记会被返回。而 ASK 标记被 Jedis 转化为 JedisAskDataException 异常。

#### 解决方案

这种异常只需要捕获并跳过即可。

------



### **Redis Cluster flush 失败**

flush 是一个极少用到的操作，不过既然碰到过诡异的现象，也记录在此。

#### 问题场景

在 Reids Cluster 中使用主从模式，向主节点发送 flush 命令，预期主从节点都会清空数据库。但是诡异的现象出现了，我们得到的结果是主从节点发生了切换，并且数据并没有被清空。

#### 分析问题

分析以上 case，Redis 采用单线程模型，flush 操作执行的时候会阻塞所有其它操作，包括集群间心跳包。当 Redis 中有大量数据的时候，flush 操作会消耗较长时间。所以该节点较长时间不能跟集群通信，当达到一定阈值的时候，集群会判定该节点为 fail，并且会切换主从状态。

Redis 采用异步的方式进行主从同步，flush 操作在主节点执行完成之后，才会将命令同步到从节点。此时老的从节点变为了主节点，它不会再接受来自老的主节点的删除数据的操作。

当老的主节点 flush 完成的时候，它恢复与集群中其它节点的通讯，得知自己被变成了从节点，所又会把数据同步过来。最终造成了主从节点发生了切换，并且数据没有被清空的现象。

#### 解决方式

临时调大集群中所有节点的 cluster-node-timeout 参数。



### **Redis 启动异常问题**

#### 问题描述

我们集群中每个物理主机上启动多个 Redis 以利用多核主机的计算资源。

问题发生在一次主机宕机。恢复服务的过程中，当启动某一个 Redis 实例的时候，Redis 实例正常启动，但是集群将它标记为了 fail 状态。

#### 问题分析

众所周知 Redis Cluster 中的实例，需要监听两个端口，一个服务端口（默认 6379），另一个是集群间通讯端口（16379），它是服务端口加上 10000。

经过一番调查发现该节点的服务通讯端口，已经被集群中其它节点占用了，导致它不能与集群中其它节点通讯，被标记为 fail 状态。

#### 解决方案

找到占用该端口的 Redis 进程并重启。



### 总结

本篇文章偏运维，不过在开发人员进行业务重构，数据也跟着迁移，是有一定的参考性的；

参考：https://www.infoq.cn/article/2016%2F08%2Fyouku-Redis-nosql



