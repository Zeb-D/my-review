





### ZAB 和 Paxos

Zab 和 Paxos 协议在实现上其实有非常多的相似点，例如：

- 主节点会向所有的从节点发出提案；
- 主节点在接收到一组从节点中 50% 以上节点的确认后，才会认为当前提案被提交了；
- Zab 协议中的每一个提案都包含一个 epoch 值，与 Paxos 中的 Ballot 非常相似；

因为它们有一些相同的特点，所以有的观点会认为 Zab 是 Paxos 的一个简化版本，但是 Zab 和 Paxos 在设计理念上就有着比较大的不同，两者的主要区别就在于 Zab 主要是为构建高可用的主备系统设计的，而 Paxos 能够帮助工程师搭建具有一致性的状态机系统。

作为一个一致性状态机系统，它能够保证集群中任意一个状态机副本都按照客户端的请求执行了相同顺序的请求，即使来自客户端请求是异步的并且不同客户端的接收同一个请求的顺序不同，集群中的这些副本就是会使用 Paxos 或者它的变种对提案达成一致；在集群运行的过程中，如果主节点出现了错误导致宕机，其他的节点会重新开始进行选举并处理未提交的请求。

但是在类似 Zookeeper 的高可用主备系统中，所有的副本都需要对增量的状态更新顺序达成一致，这些状态更新的变量都是由主节点创建并发送给其他的从节点的，每一个从节点都会严格按照顺序逐一的执行主节点生成的状态更新请求，如果 Zookeeper 集群中的主节点发生了宕机，新的主节点也必须严格按照顺序对请求进行恢复。

总的来说，使用状态更新节点数据的主备系统相比根据客户端请求改变状态的状态机系统对于请求的执行顺序有着更严格的要求。



### Zab只是Paxos的一个特殊实现吗？

不，Zab是一种与Paxos不同的协议，尽管它与它共享一些关键方面，例如：

- 领导者向追随者提出价值观
- 在考虑提交的建议（学习）之前，领导者等待法定数量的追随者的确认
- 提案包括纪元数字，类似于Paxos中的选票数字

Zab和Paxos之间的主要概念差异在于它主要设计用于主备份系统，如Zookeeper，而不是用于状态机复制。



### 主备份和状态机复制有什么区别？

状态机是处理一系列请求的软件组件。对于每个已处理的请求，它可以修改其内部状态并生成回复。状态机在某种意义上是确定性的，给定两次运行，它接收相同的请求序列，它总是进行相同的内部状态转换并产生相同的回复。

状态机复制系统是客户机 - 服务器系统，确保每个状态机副本执行相同的**客户机请求**序列，即使这些请求由客户端同时提交并由副本以不同的顺序接收。副本使用像Paxos这样的一致性算法就客户端请求的执行顺序达成一致。可以按任何顺序执行同时发送并在时间上重叠的客户端请求。如果领导者失败，执行恢复的新领导者可以随意对任何未提交的请求进行任意重新排序，因为它尚未完成。

对于主备份系统（例如Zookeeper），副本同意**增量（增量）状态更新**的应用顺序，**增量（增量）状态更新**由主副本生成并发送给其关注者。与客户端请求不同，状态更新必须以主要的原始初始状态开始，以主要的确切原始生成顺序应用。如果主服务器发生故障，执行恢复的新主服务器不能任意重新排序未提交的状态更新，或者从不同的初始状态开始应用它们。

总之，关于状态更新（对于主备份系统）的协议要求比对客户端请求（对于状态机复制系统）的协议更严格的排序保证。



### 对协议算法有什么影响？

Paxos可以通过让主要成为领导者来用于主备份复制。Paxos的问题在于，如果主服务器*同时*提出多个状态更新并且失败，则新主服务器可能以不正确的顺序应用未提交的更新。可以看下[DSN 2011论文](https://pdfs.semanticscholar.org/fc11/031895c302dc52404d34de58af1a72f3b817.pdf)（图1）中提供了一个示例。在该示例中，副本应该仅在应用A之后应用状态更新B.该示例显示，使用Paxos，新的主要及其后续可以在C之后应用B，达到任何未达到的错误状态。以前的初选。

使用Paxos解决此问题的方法是*按顺序*就状态更新达成一致：主要仅在提交所有先前的状态更新后才建议状态更新。由于一次最多只有一个未提交的更新，因此新主服务器不能错误地重新排序更新。然而，这种方法导致性能不佳。

Zab不需要这种解决方法。Zab副本可以同时就多个状态更新的顺序达成一致，而不会损害正确性。与Paxos相比，通过在恢复期间再添加一个同步阶段，并通过使用基于zxids的不同实例数来实现这一点。

![Fig. 1. Paxos run](https://ai2-s2-public.s3.amazonaws.com/figures/2017-08-08/b02c6b00bd5dbdbd951fddb00b906c82fa80f0b3/2-Figure1-1.png)

![å¾2. ZooKeeperæ¦è¿°ã](https://ai2-s2-public.s3.amazonaws.com/figures/2017-08-08/b02c6b00bd5dbdbd951fddb00b906c82fa80f0b3/3-Figure2-1.png)![Fig. 4. Example of an execution satisfying causal order (and PO causal order), but not strict causality, epoch(z) < epoch(zâ²) < epoch(zâ²â²).](https://ai2-s2-public.s3.amazonaws.com/figures/2017-08-08/b02c6b00bd5dbdbd951fddb00b906c82fa80f0b3/5-Figure4-1.png)

![Fig. 3. Example of an execution satisfying PO causal order, but not causal order, epoch(z) < epoch(zâ²) < epoch(zâ²â²).](https://ai2-s2-public.s3.amazonaws.com/figures/2017-08-08/b02c6b00bd5dbdbd951fddb00b906c82fa80f0b3/4-Figure3-1.png)

![Fig. 5. Zab protocol summary](https://ai2-s2-public.s3.amazonaws.com/figures/2017-08-08/b02c6b00bd5dbdbd951fddb00b906c82fa80f0b3/7-Figure5-1.png)









##### 参考

https://cwiki.apache.org/confluence/display/ZOOKEEPER/Zab+vs.+Paxos

https://www.semanticscholar.org/paper/Zab%3A-High-performance-broadcast-for-primary-backup-Junqueira-Reed/b02c6b00bd5dbdbd951fddb00b906c82fa80f0b3