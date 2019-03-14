本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

对于Kafka这方面的集群了解越发迷茫，结合[Kafka 高并发写入数据](./kafka之高并发写入分析.md)，自己心里有个疑问：kafka的集群是怎么架构的，分布式存储是什么样的？等等

自己会默默的比对一下其它中间件在 集群方面的一致性方面的处理，如ZK的[ZAB协议](../../zk/ZAB协议.md)集群内数据一致性的处理；

### 主要特点

1. 同时为发布和订阅提供高吞吐量。据了解，Kafka每秒可以生产约25万消息（50 MB），每秒处理55万消息（110 MB）。
2. 可进行持久化操作。将消息持久化到磁盘，因此可用于批量消费，例如ETL，以及实时应用程序。通过将数据持久化到硬盘以及replication防止数据丢失。
3. 分布式系统，易于向外扩展。所有的producer、broker和consumer都会有多个，均为分布式的。无需停机即可扩展机器。
4. 消息被处理的状态是在consumer端维护，而不是由server端维护。当失败时能自动平衡。
5. 支持online和offline的场景。



### Kafka分布式存储

Kafka的整体架构非常简单，是显式分布式架构，producer、broker（kafka）和consumer都可以有多个。producer、consumer实现Kafka注册的接口；

数据从producer发送到broker，broker承担一个中间缓存和分发的作用。

broker分发注册到系统中的consumer。

broker的作用类似于缓存，即活跃的数据和离线处理系统之间的缓存。客户端和服务器端的通信，是基于简单，高性能，且与编程语言无关的TCP协议。

#### 基本概念

##### Topic

> 特指Kafka处理的消息源（feeds of messages）的不同分类。
>
> Kafka 由多个 broker 组成，每个 broker 是一个**节点**；你创建一个 topic【姑且认为是一个数据集合】

举个例子，如果你现在有一份网站的用户行为数据要写入Kafka，你可以搞一个topic叫做“user_access_log_topic”，这里写入的都是用户行为数据。

然后如果你要把电商网站的订单数据的增删改变更记录写Kafka，那可以搞一个topic叫做“order_tb_topic”，这里写入的都是订单表的变更记录。

比如有个topic里面如果每天写入几十TB的数据，你觉得都放一台机器上靠谱吗？

##### Partition

> Topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列。partition中的每条消息都会被分配一个有序的id（offset）。

topic 可以划分为多个 partition，每个 partition 可以存在于不同的 broker 上，每个 partition 就放一部分数据。

这就是**天然的分布式消息队列**，就是说一个 topic 的数据，是**分散放在多个机器上的，每个机器就放一部分数据**。

##### Message

> 消息，是通信的基本单位，每个producer可以向一个topic（主题）发布一些消息。

##### Producers

> 消息和数据生产者，向Kafka的一个topic发布消息的过程叫做producers。

##### Consumers

> 消息和数据消费者，订阅topics并处理其发布的消息的过程叫做consumers。

##### Broker

> 缓存代理，Kafka集群中的一台或多台服务器统称为broker。
>
> 里面一般包括多个topic、多个Partition



#### 高可用架构

实际上 RabbmitMQ 之类的，并不是分布式消息队列，它就是传统的消息队列，只不过提供了一些集群、HA(High Availability, 高可用性) 的机制而已，因为无论怎么玩儿，RabbitMQ 一个 queue 的数据都是放在一个节点里的，镜像集群下，也是每个节点都放这个 queue 的完整数据。

Kafka 0.8 以前，是没有 HA 机制的，就是任何一个 broker 宕机了，那个 broker 上的 partition 就废了，没法写也没法读，没有什么高可用性可言。

比如说，我们假设创建了一个 topic，指定其 partition 数量是 3 个，分别在三台机器上。但是，如果第二台机器宕机了，会导致这个 topic 的 1/3 的数据就丢了，因此这个是做不到高可用的。

[![kafka-before](../../../image/kafka-before.png)](../../../image/kafka-before.png)

Kafka 0.8 以后，提供了 HA 机制，就是 replica（复制品） 副本机制。每个 partition 的数据都会同步到其它机器上，形成自己的多个 replica 副本。所有 replica 会选举一个 leader 出来，那么生产和消费都跟这个 leader 打交道，然后其他 replica 就是 follower。写的时候，leader 会负责把数据同步到所有 follower 上去，读的时候就直接读 leader 上的数据即可。只能读写 leader？很简单，**要是你可以随意读写每个 follower，那么就要 care 数据一致性的问题**，系统复杂度太高，很容易出问题。Kafka 会均匀地将一个 partition 的所有 replica 分布在不同的机器上，这样才可以提高容错性。

[![kafka-after](../../../image/kafka-after.png)](../../../image/kafka-after.png)

这么搞，就有所谓的**高可用性**了，因为如果某个 broker 宕机了，没事儿，那个 broker上面的 partition 在其他机器上都有副本的，如果这上面有某个 partition 的 leader，那么此时会从 follower 中**重新选举**一个新的 leader 出来，大家继续读写那个新的 leader 即可。这就有所谓的高可用性了。

**写数据**的时候，生产者就写 leader，然后 leader 将数据落地写本地磁盘，接着其他 follower 自己主动从 leader 来 pull 数据。一旦所有 follower 同步好数据了，就会发送 ack 给 leader，leader 收到所有 follower 的 ack 之后，就会返回写成功的消息给生产者。（当然，这只是其中一种模式，还可以适当调整这个行为）

**消费**的时候，只会从 leader 去读，但是只有当一个消息已经被所有 follower 都同步成功返回 ack 的时候，这个消息才会被消费者读到。

简单理解

> Leader 节点平时**对外提供读写** 且 把自身的**数据同步到Follower** 列表机器，一旦宕机，Follower就开始某种选举变成Leader



### 写入一致性分析

#### 提问

写入数据都是往某个Partition的Leader写入的，然后那个Partition的Follower会从Leader同步数据。

有一条数据是没同步到Partition0的Follower上去的，然后Partition0的Leader所在机器宕机了。

此时就会选举Partition0的Follower作为新的Leader对外提供服务，然后用户是不是就读不到刚才写入的那条数据了？因为Partition0的Follower上是没有同步到最新的一条数据的。

#### ISR机制

然后发现kafka 对于这个问题的处理方式引入了ISR机制：

> 自动给每个Partition维护一个ISR列表，这个列表里一定会有Leader，然后还会包含跟Leader保持同步的Follower。
>
> 也就是说，只要Leader的某个Follower一直跟他保持数据同步，那么就会存在于ISR列表里。
>
> 但是如果Follower因为自身发生一些问题，导致不能及时的从Leader同步数据过去，那么这个Follower就会被认为是“out-of-sync”，从ISR列表里踢出去。

#### 数据不丢失

写入Kafka的**数据不丢失**要点

1. **每个Partition都至少得有1个Follower在ISR列表里，跟上了Leader的数据同步**
2. **每次写入数据的时候，都要求至少写入Partition Leader成功，同时还有至少一个ISR里的Follower也写入成功，才算这个写入是成功了**
3. **如果不满足上述两个条件，那就一直写入失败，让生产系统不停的尝试重试，直到满足上述两个条件，然后才能认为写入成功**
4. **按照上述思路去配置相应的参数，才能保证写入Kafka的数据不会丢失**

第一条，必须要求至少一个Follower在ISR列表里。

那必须的啊，要是Leader没有Follower了，或者是Follower都没法及时同步Leader数据，那么这个事儿肯定就没法弄下去了。

第二条，每次写入数据的时候，要求leader写入成功以外，至少一个ISR里的Follower也写成功。

大家看下面的图，这个要求就是保证说，每次写数据，必须是leader和follower都写成功了，才能算是写成功，保证一条数据必须有两个以上的副本。

这个时候万一leader宕机，就可以切换到那个follower上去，那么Follower上是有刚写入的数据的，此时数据就不会丢失了。

假如现在leader没有follower了，或者是刚写入leader，leader立马就宕机，还没来得及同步给follower。

在这种情况下，写入就会失败，然后你就让生产者不停的重试，直到kafka恢复正常满足上述条件，才能继续写入。

这样就可以让写入kafka的数据不丢失。



### 总结

其实kafka的数据丢失问题，涉及到方方面面。

譬如生产端的缓存问题，包括消费端的问题，同时kafka自己内部的底层算法和机制也可能导致数据丢失。

但是平时写入数据遇到比较大的一个问题，就是leader切换时可能导致数据丢失。





