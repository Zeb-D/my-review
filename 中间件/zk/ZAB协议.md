本文章来源于：https://github.com/Zeb-D/my-review ，请star 强力支持，你的支持，就是我的动力。

<br>

##ZAB协议介绍

ZAB ( ZooKeeper Atomic Broadcast , ZooKeeper 原子消息广播协议）是zookeeper数据一致性的核心算法。

ZAB 协议并不像 Paxos 算法那样，是一种通用的分布式一致性算法，它是一种特别为 ZooKeeper 设计的崩溃可恢复的原子消息广播算法。

阅读本文前，需要了解[zk基本常识](./zk——你知道的zk是这样的吗.md)。

## ZAB协议特性

1. 使用一个单一的主进程来接收并处理客户端的所有事务请求，并采用 ZAB 的原子广播协议，将服务器数据的状态变更以事务 Proposal 的形式广播到所有的副本进程上去。
2. 保证一个全局的变更序列被顺序应用：ZooKeeper是一个树形结构，很多操作都要先检查才能确定能不能执行，比如P1的事务t1可能是创建节点“/a”，t2可能是创建节点“/a/aa”，只有先创建了父节点“/a”，才能创建子节点“/a/aa”。为了保证这一点，ZAB要保证同一个leader的发起的事务要按顺序被apply，同时还要保证只有先前的leader的所有事务都被apply之后，新选的leader才能在发起事务。
3. 当前主进程出现异常情况的时候，依旧能够正常工作。

## ZAB 协议的核心

所有事务请求必须由一个全局唯一的服务器来协调处理，这样的服务器被称为 Leader服务器，而余下的其他服务器则成为 Follower 服务器。 Leader 服务器负责将一个客户端事务请求转换成一个事务proposal（提议），并将该 Proposal分发给集群中所有的Follower服务器。之后 Leader 服务器需要等待所有Follower 服务器的反馈,一旦超过半数的Follower服务器进行了正确的反馈后，那么 Leader 就会再次向所有的 Follower服务器分发Commit消息，要求其将前一个proposal进行提交。在这里有详细介绍[zk主节点故障恢复和广播消息](./zk——你知道的zk是这样的吗.md)。

这种事务处理方式与[2PC](../../分布式/分布式系统深入理解2PC和3PC.md)（[两阶段提交协议](../../分布式/分布式事务中的两阶段、三阶段提交协议.md)）区别在于，两阶段提交协议的第二阶段中，需要等到所有参与者的"YES"回复才会提交事务，只要有一个参与者反馈为"NO"或者超时无反馈，都需要中断和回滚事务。

## ZAB处理废弃的事务

​	ZAB 协议是如何处理那些需要被丢弃的事务 Proposal 的。在 ZAB 协议的事务编号 ZXID 设计中， ZXID 是一个 64 位的数字，低 32 位可以看作是一个简单的单调递增的计数器，针对客户端的每一个事务请求， Leader 服务器在产生一个新的事务 Proposal 的时候，都会对该计数器进行加1操作；高 32 位代表了 Leader 周期 epoch 的编号，每当选举产生一个新的 Leader 服务器，就会从这个 Leader 服务器上取出其本地日志中最大事务 Proposal 的 ZXID ,并从该 ZXID 中解析出对应的 epoch 值，然后再对其进行加1操作，之后就会以此编号作为新的 epoch, 并将低 32 位置0来开始生成新的 ZXID 。

　　基于这样的策略，当一个包含了上一个 Leader 周期中尚未提交过的事务 Proposal的服务器启动加入到集群中，发现此时集群中已经存在leader，将自身以Follower 角色连接上 Leader 服务器之后， Leader 服务器会根据自己服务器上最后被提交的 Proposal来和 Follower 服务器的 Proposal进行比对，发现follower中有上一个leader周期的事务Proposal时，Leader 会要求 Follower 进行一个回退操作——回退到一个确实已经被集群中过半机器提交的最新的事务 Proposal 。

 ## ZAB源码实现

我们了解了ZAB的故障恢复和消息广播后，我们将其抽象成如下：

![zab实现](https://static.oschina.net/uploads/img/201610/31160806_blP5.png)

###名词解释

- long lastProcessedZxid：最后一次commit的事务请求的zxid

- LinkedList committedLog、long maxCommittedLog、long minCommittedLog：

  ZooKeeper会保存最近一段时间内执行的事务请求议案，个数限制默认为500个议案。上述committedLog就是用来保存议案的列表，上述maxCommittedLog表示最大议案的zxid，minCommittedLog表示committedLog中最小议案的zxid。

- ConcurrentMap outstandingProposals

  Leader拥有的属性，每当提出一个议案，都会将该议案存放至outstandingProposals，一旦议案被过半认同了，就要提交该议案，则从outstandingProposals中删除该议案

- ConcurrentLinkedQueue toBeApplied

  Leader拥有的属性，每当准备提交一个议案，就会将该议案存放至该列表中，一旦议案应用到ZooKeeper的内存树中了，然后就可以将该议案从toBeApplied中删除

整个[消息广播](./zk——你知道的zk是这样的吗.md)的处理过程可以描述为：

- leader针对客户端的事务请求（leader为该请求分配了zxid），创建出一个议案，并将zxid和该议案存放至leader的outstandingProposals中
- leader开始向所有的follower发送该议案，如果过半的follower回复OK的话，则leader认为可以提交该议案，则将该议案从outstandingProposals中删除，然后存放到toBeApplied中
- leader对该议案进行提交，会向所有的follower发送提交该议案的命令，leader自己也开始执行提交过程，会将该请求的内容应用到ZooKeeper的内存树中，然后更新lastProcessedZxid为该请求的zxid，同时将该请求的议案存放到上述committedLog，同时更新maxCommittedLog和minCommittedLog
- leader就开始向客户端进行回复，然后就会将该议案从toBeApplied中删除

这里讲从ZAB从新讲述**主节点故障恢复**。

### zk快速选举阶段

leader选举过程要关注的要点：

- 所有机器刚启动时进行leader选举过程
- 如果leader选举完成，刚启动起来的server怎么识别到leader选举已完成

投票过程有3个重要的数据:

- ServerState

  目前ZooKeeper机器所处的状态有4种，分别是

  - LOOKING：进入leader选举状态
  - FOLLOWING：leader选举结束，进入follower状态
  - LEADING：leader选举结束，进入leader状态
  - OBSERVING：处于观察者状态

- HashMap recvset

  用于收集LOOKING、FOLLOWING、LEADING状态下的server的投票

- HashMap outofelection

  用于收集FOLLOWING、LEADING状态下的server的投票（能够收集到这种状态下的投票，说明leader选举已经完成）

下面就来详细说明这个过程：

1、 serverA首先将electionEpoch自增，然后为自己投票

serverA会首先从快照日志和事务日志中加载数据，就可以得到本机器的内存树数据，以及lastProcessedZxid（这一部分后面再详细说明）

初始投票Vote的内容：

- proposedLeader：ZooKeeper Server中的myid值，初始为本机器的id
- proposedZxid：最大事务zxid，初始为本机器的lastProcessedZxid
- proposedEpoch:peerEpoch值，由上述的lastProcessedZxid的高32得到

然后该serverA向其他所有server发送通知，通知内容就是上述投票信息和electionEpoch信息

2、 serverB接收到上述通知，然后进行投票PK

如果serverB收到的通知中的electionEpoch比自己的大，则serverB更新自己的electionEpoch为serverA的electionEpoch

如果该serverB收到的通知中的electionEpoch比自己的小，则serverB向serverA发送一个通知，将serverB自己的投票以及electionEpoch发送给serverA，serverA收到后就会更新自己的electionEpoch

在electionEpoch达成一致后，就开始进行投票之间的pk，规则如下：

```java
/*
 * We return true if one of the following three cases hold:
 * 1- New epoch is higher
 * 2- New epoch is the same as current epoch, but new zxid is higher
 * 3- New epoch is the same as current epoch, new zxid is the same
 *  as current zxid, but server id is higher.
 */
 
return ((newEpoch > curEpoch) || 
        ((newEpoch == curEpoch) &&
        ((newZxid > curZxid) || ((newZxid == curZxid) && (newId > curId)))));

```

就是优先比较proposedEpoch，然后优先比较proposedZxid，最后优先比较proposedLeader

pk完毕后，如果本机器投票被pk掉，则更新投票信息为对方投票信息，同时重新发送该投票信息给所有的server。

如果本机器投票没有被pk掉，则看下面的过半判断过程：

3、 根据server的状态来判定leader

如果当前发来的投票的server的状态是LOOKING状态，则只需要判断本机器的投票是否在recvset中过半了，如果过半了则说明leader选举就算成功了，如果当前server的id等于上述过半投票的proposedLeader,则说明自己将成为了leader，否则自己将成为了follower

如果当前发来的投票的server的状态是FOLLOWING、LEADING状态，则说明leader选举过程已经完成了，则发过来的投票就是leader的信息，这里就需要判断发过来的投票是否在recvset或者outofelection中过半了

同时还要检查leader是否给自己发送过投票信息，从投票信息中确认该leader是不是LEADING状态（这一部分还需要仔细推敲下）

###发现阶段

一旦leader选举完成，就开始进入恢复阶段，就是follower要同步leader上的数据信息。

1.  通信初始化

   leader会创建一个ServerSocket，接收follower的连接，leader会为每一个连接会用一个LeaderHandler线程来进行服务

2.  重新为peerEpoch选举出一个新的peerEpoch

   follower会向leader发送一个Leader.FOLLOWERINFO信息，包含自己的peerEpoch信息

   leader的LearnerHandler会获取到上述peerEpoch信息，leader从中选出一个最大的peerEpoch，然后加1作为新的peerEpoch。

   然后leader的所有LearnerHandler会向各自的follower发送一个Leader.LEADERINFO信息，包含上述新的peerEpoch

   follower会使用上述peerEpoch来更新自己的peerEpoch，同时将自己的lastProcessedZxid发给leader

   leader的所有LearnerHandler会记录上述各自follower的lastProcessedZxid，然后根据这个lastProcessedZxid和leader的lastProcessedZxid之间的差异进行同步

3.  已经处理的事务议案的同步

   判断LearnerHandler中的lastProcessedZxid是否在minCommittedLog和maxCommittedLog之间

   - LearnerHandler中的lastProcessedZxid和leader的lastProcessedZxid一致，则说明已经保持同步了

   - 如果lastProcessedZxid在minCommittedLog和maxCommittedLog之间

     从lastProcessedZxid开始到maxCommittedLog结束的这部分议案，重新发送给该LearnerHandler对应的follower，同时发送对应议案的commit命令

     上述可能存在一个问题：即lastProcessedZxid虽然在他们之间，但是并没有找到lastProcessedZxid对应的议案，即这个zxid是leader所没有的，此时的策略就是完全按照leader来同步，删除该follower这一部分的事务日志，然后重新发送这一部分的议案，并提交这些议案

   - 如果lastProcessedZxid大于maxCommittedLog

     则删除该follower大于部分的事务日志

   - 如果lastProcessedZxid小于minCommittedLog

     则直接采用快照的方式来恢复

4.  未处理的事务议案的同步

   LearnerHandler还会从leader的toBeApplied数据中将大于该LearnerHandler中的lastProcessedZxid的议案进行发送和提交（toBeApplied是已经被确认为提交的）

   LearnerHandler还会从leader的outstandingProposals中大于该LearnerHandler中的lastProcessedZxid的议案进行发送，但是不提交（outstandingProposals是还没被被确认为提交的）

5.  将LearnerHandler加入到正式follower列表中

   意味着该LearnerHandler正式接受请求。即此时leader可能正在处理客户端请求，leader针对该请求发出一个议案，然后对该正式follower列表才会进行执行发送工作。这里有一个地方就是：

   上述我们在比较lastProcessedZxid和minCommittedLog和maxCommittedLog差异的时候，必须要获取leader内存数据的读锁，即在此期间不能执行修改操作，当欠缺的数据包已经补上之后（先放置在一个队列中，异步发送），才能加入到正式的follower列表，否则就会出现顺序错乱的问题

   同时也说明了，一旦一个follower在和leader进行同步的过程（这个同步过程仅仅是确认要发送的议案，先放置到队列中即可等待异步发送，并不是说必须要发送过去），该leader是暂时阻塞一切写操作的。

   对于快照方式的同步，则是直接同步写入的，写入期间对数据的改动会放在上述队列中的，然后当同步写入完成之后，再启动对该队列的异步写入。

   上述的要理解的关键点就是：既要不能漏掉，又要保证顺序

6.  LearnerHandler发送Leader.NEWLEADER以及Leader.UPTODATE命令

   该命令是在同步结束之后发的，follower收到该命令之后会执行一次版本快照等初始化操作，如果收到该命令的ACK则说明follower都已经完成同步了并完成了初始化

   leader开始进入心跳检测过程，不断向follower发送心跳命令，不断检是否有过半机器进行了心跳回复，如果没有过半，则执行关闭操作，开始进入leader选举状态

   LearnerHandler向对应的follower发送Leader.UPTODATE，follower接收到之后，开始和leader进入Broadcast处理过程

<br>

## 特殊情况

### 事务日志和快照日志的持久化和恢复

先来看看持久化过程：

- Broadcast过程的持久化

  leader针对每次事务请求都会生成一个议案，然后向所有的follower发送该议案

  follower接收到该议案后，所做的操作就是将该议案记录到事务日志中，每当记满100000个（默认），则事务日志执行flush操作，同时开启一个新的文件来记录事务日志

  同时会执行内存树的快照，snapshot.[lastProcessedZxid]作为文件名创建一个新文件，快照内容保存到该文件中

- leader shutdown过程的持久化

  一旦leader过半的心跳检测失败，则执行shutdown方法，在该shutdown中会对事务日志进行flush操作

再来说说恢复：

- 事务快照的恢复

  第一：会在事务快照文件目录下找到最近的100个快照文件，并排序，最新的在前

  第二：对上述快照文件依次进行恢复和验证，一旦验证成功则退出，否则利用下一个快照文件进行恢复。恢复完成更新最新的lastProcessedZxid

- 事务日志的恢复

  第一：从事务日志文件目录下找到zxid大于等于上述lastProcessedZxid的事务日志

  第二：然后对上述事务日志进行遍历，应用到ZooKeeper的内存树中，同时更新lastProcessedZxid

  第三：同时将上述事务日志存储到committedLog中，并更新maxCommittedLog、minCommittedLog

由此我们可以看到，在初始化恢复的时候，是会将所有最新的事务日志作为已经commit的事务来处理的

也就是说这里面可能会有部分事务日志还没真实提交，而这里全部当做已提交来处理。这个处理简单粗暴了一些，而raft对老数据的恢复则控制的更加严谨一些。

### follower恢复

follower挂了之后又重启的恢复过程：

一旦leader挂了，上述leader的2个集合

- ConcurrentMap outstandingProposals
- ConcurrentLinkedQueue toBeApplied

就无效了。他们并不在leader恢复的时候起作用，而是在系统正常执行，而某个follower挂了又恢复的时候起作用。

我们可以看到在上述2.3的恢复过程中，会首先进行快照日志和事务日志的恢复，然后再补充leader的上述2个数据中的内容。

如果在同步follower时：

目前leader和follower之间的同步是通过BIO方式来进行的，一旦该链路出现异常则会关闭该链路，重新与leader建立连接，重新同步最新的数据。

### client端消息是否一致

- 客户端收到OK回复，会不会丢失数据？
- 客户端没有收到OK回复，会不会多存储数据？

客户端如果收到OK回复，说明已经过半复制了，则在leader选举中肯定会包含该请求对应的事务日志，则不会丢失该数据

客户端连接的leader或者follower挂了，客户端没有收到OK回复，目前是可能丢失也可能没丢失，因为服务器端的处理也很简单粗暴，对于未来leader上的事务日志都会当做提交来处理的，即都会被应用到内存树中。

同时目前ZooKeeper的原生客户端也没有进行重试，服务器端也没有对重试进行检查。这一部分到下一篇再详细探讨与raft的区别

