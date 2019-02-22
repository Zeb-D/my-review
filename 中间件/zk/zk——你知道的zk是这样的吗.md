本文章已同步到个人学习仓库中：https://github.com/Zeb-D/my-review，欢迎品读及star。

[TOC]

------



### 提问

你平常使用zookeeper做什么？是分布式协调服务、共享变量、协调锁资源、还是提供命名空间？

好了，接下来我们以提问的形式来打开话题：

你知道zk能用来做什么？

你知道zk的数据模型吗？

你知道zk的数据结构吗？

你会zk操作基本命令吗？

这些命令是如何事件通知？

zk是如何保证一致性的？

你会用zk做什么？

<br>

### zk数据模型

了解一门技术，先知道它大致长得啥样，这才好的去进一步认识。

zk的数据模型：

很像数据结构中的树，也像文件系统的目录；

zk的数据存储同样基于节点，叫Znode；

但引用方式是：路径引用，类似于文件路径，让每一个节点拥有唯一的路径

这里讲到了Znode （zk的节点）：

<br>

### Znode数据结构

它的主要属性有data、ACL、child、stat

data：Znode存储的数据信息。

ACL：记录Znode的访问权限，即哪些人或哪些IP可以访问本节点。

stat：包含Znode的各种元数据，比如事务ID、版本号、时间戳、大小等等。

child：当前节点的子节点引用，类似于二叉树的左孩子右孩子。

**注意**：Zookeeper是为读多写少的场景所设计。Znode并不是用来存储大规模业务数据，而是用于存储少量的状态和配置信息，每个节点的数据最大不能超过1MB。

<br>

### 基本操作

Zookeeper包含了哪些基本操作呢？这里列举出比较常用的API：

create：创建节点

delete：删除节点

exists：判断节点是否存在

getData：获得一个节点的数据

setData：设置一个节点的数据

getChildren：获取节点下的所有子节点

exists，getData，getChildren属于读操作。Zookeeper客户端在请求读操作的时候，可以选择是否设置Watch。

讲到这，大家可能会联想到我们平常使用的zkclient 这插件，其实这里面的命令都是通过if-else这样判断单独调用远程server的命令，目前博主对这个jar通信方式还在研究，会单独抽出时间整理一下。<br>

zk客户端的数据是如何与server数据保持一致，其中是离不开Watch动作的？

###事件通知

Watch是什么意思呢？

 我们可以理解成是注册在特定Znode上的触发器。当这个Znode发生改变，也就是调用了create，delete，setData方法的时候，将会触发Znode上注册的对应事件，请求Watch的客户端会接收到异步通知。 

具体交互过程如下：

1.客户端调用getData方法【getData(nodePath,isWatch)】，watch参数是true。服务端接到请求，返回节点数据，并且在对应的哈希表里插入被Watch的Znode路径，以及Watcher列表。

2.当被Watch的Znode已删除，服务端会查找哈希表，找到该Znode对应的所有Watcher，异步通知客户端，并且删除哈希表中对应的Key-Value。

<br>

### zk的应用

讲到zk的应用，肯定得想到zk有什么作用，才能知道它能做什么？

zk可以用来 **协调**分布式服务、共享变量、协调锁资源、提供命名空间，其实这些作用是依赖zk的数据模型以及ZNode的数据结构，也可以从侧面反映出，先有功能需求 才能想出 好的数据模型及数据结构。

好了，我们平常会将zk用来做什么？

1.分布式锁

这是雅虎研究员设计Zookeeper的初衷。利用Zookeeper的临时顺序节点，可以轻松实现分布式锁。

2.服务注册和发现

利用Znode和Watcher，可以实现分布式服务的注册和发现。最著名的应用就是阿里的分布式RPC框架Dubbo。

3.共享配置和状态信息

Redis的分布式解决方案Codis，就利用了Zookeeper来存放数据路由表和codis-proxy 节点的元信息。同时 codis-config 发起的命令都会通过 ZooKeeper 同步到各个存活的 codis-proxy。

此外，Kafka、HBase、Hadoop，也都依靠Zookeeper同步节点信息，实现高可用。

4.其实它还可以用来解决分布式Id生成，只不过zk的zNode支持的数量不够多，因为zk主要使用场景是那种读多写少的，

好了，插个小广告，我自己 基于netty4+twitter-snowFlake 写了个分布式Id生成之服务：https://github.com/Zeb-D/distributed-id ；关于这开源项目，我后续会单独写个系列的博客。

<br>

### zk的集群

集群单纯地防止单个zk挂了，导致所有依赖zk的服务都不可用（或者被影响到），其实zkClient 这会在调用方 会缓存下来一下zk Server的数据，具体是怎么个样子，可能需要单独研究分析下，博主目前不知道怎么去分析，希望各位老道们给点建议，谢谢！

ZookeeperService集群是一主多从结构。

![https://github.com/Zeb-D/my-review/blob/master/image/zk-distributed.png](zk集群图)

在更新数据时，首先更新到主节点（这里的节点是指服务器，不是Znode），再同步到从节点。

在读取数据时，直接读取任意从节点。

说到这一主多从集群方式，大家可能会联想到Mysql集群方式，这种集群方式保证服务高可用，但又是如何保证各个主从节点任意时间的数据一致性呢？

<br>

### zk的一致性

为了保证主从节点的数据一致性，Zookeeper采用了ZAB协议，这种协议非常类似于一致性算法Paxos和Raft。

### ZAB协议

我们需要首先了解ZAB协议所定义的三种节点状态：

Looking ：选举状态。

Following：Follower节点（从节点）所处的状态。

Leading：Leader节点（主节点）所处状态。 

我们还需要知道**最大ZXID的概念**：

最大ZXID也就是节点本地的最新事务编号，包含epoch和计数两部分。epoch是纪元的意思，相当于Raft算法选主时候的term。 

###zk主节点故障恢复

假如Zookeeper当前的主节点挂掉了，集群会进行崩溃恢复。ZAB的崩溃恢复分成三个阶段：

1.Leader election

**选举阶段**，此时集群中的节点处于Looking状态。它们会各自向其他节点发起投票，投票当中包含自己的服务器ID和最新事务ID（ZXID）。

接下来，节点会用自身的ZXID和从其他节点接收到的ZXID做比较，如果发现别人家的ZXID比自己大，也就是数据比自己新，那么就重新发起投票，投票给目前已知最大的ZXID所属节点。

每次投票后，服务器都会统计投票数量，判断是否有某个节点得到半数以上的投票。如果存在这样的节点，该节点将会成为准Leader，状态变为Leading。其他节点的状态变为Following。 

2.Discovery

**发现阶段**，用于在从节点中发现最新的ZXID和事务日志。或许有人会问：既然Leader被选为主节点，已经是集群里数据最新的了，为什么还要从节点中寻找最新事务呢？

这是为了防止某些意外情况，比如因网络原因在上一阶段产生多个Leader的情况。

所以这一阶段，Leader集思广益，接收所有Follower发来各自的最新epoch值。Leader从中选出最大的epoch，基于此值加1，生成新的epoch分发给各个Follower。

各个Follower收到全新的epoch后，返回ACK给Leader，带上各自最大的ZXID和历史事务日志。Leader选出最大的ZXID，并更新自身历史日志。

3.Synchronization

**同步阶段**，把Leader刚才收集得到的最新历史事务日志，同步给集群中所有的Follower。只有当半数Follower同步成功，这个准Leader才能成为正式的Leader。

自此，故障恢复正式完成。

<br>

###ZAB消息广播

在上面的zk集群图，可用看出zk客户端是轮询到某个zk Server的，那这是如何工作及保证数据一致性的，这主要用的了广播 Broadcast，简单来说，就是Zookeeper常规情况下更新数据的时候，由Leader广播到所有的Follower。其过程如下：

1.客户端发出写入数据请求给任意Follower。 

2.Follower把写入数据请求转发给Leader。 

3.Leader采用二阶段提交方式，先发送Propose广播给Follower。

4.Follower接到Propose消息，写入日志成功后，返回ACK消息给Leader。 

5.Leader接到半数以上ACK消息，返回成功给客户端，并且广播Commit请求给Follower。

ZAB协议既不是强一致性，也不是弱一致性，而是处于两者之间的单调一致性。它依靠事务ID和版本号，保证了数据的更新和读取是有序的。

想深入理解请阅读[ZAB协议](./ZAB协议.md)。

<br>

### 配置中心Vs：zk?eureka

延伸阅读，zk 可以作为一个配置中心，那么zk符合BASE理论或CAP理论 哪些特点，或者直接想看[Eureka和Zookeeper比较](/java/spring-boot/Eureka和Zookeeper比较.md#总结)

<br>













