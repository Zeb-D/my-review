本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

`kafka`是一个非常优秀的消息中间件，本文更多的吐槽低级业务使用kafka导致的使用坑；

### 背景

IOT智能设备在未来肯定会是越来越流行，在写本文总结时，IOT商城已经迭代了好几年了，主要为B端客户提供各式功能，功能特性也是经过了风风雨雨得到沉沙式推广；

我们先不扯IOT商城和市面上商城的差异，毕竟每个业务领域不一样，就好比每种中间件自身的实现方式同时会成为它的缺点；

IOT商城主要核心功能有智能商品、活动、库存、订单、支付等模块；里面用到的MQ场景比较多，对此MQ中间件大多数是kafka；

在IOT商城各个组员、同事大佬们在使用kafka会出现使用水平不一样导致问题故障严重程度不一样；

> 作者所在的企业对线上出现了客户严重不能使用问题、资损问题会要求走对外的故障流程，搞不好辛辛苦苦地一年就白干了；

我先回忆了些有趣可对外暴露的使用坑；



### 顺序问题

#### Why

为什么要保证消息的顺序？商城业务普遍存在一个经典的订单状态的维护，`下单、支付、完成、撤销`，不可能`下单`的消息都没读取到，就先读取`支付`或`撤销`的消息吧，如果真的这样，数据不是会产生错乱？



#### How

##### partition分区

我们都知道`kafka`的`topic`是无序的，但是一个`topic`包含多个`partition`，每个`partition`内部是有序的。

只要保证生产者写消息时，按照一定的规则写到同一个`partition`，不同的消费者读不同的`partition`的消息，就能保证生产和消费者消息的顺序。

当时订单中心在初步设计的时候，对应topic分区数就3个（不要在挑刺了，历史的一把梭，后面重构得红红火火）；

接手之前是根据商城saas的一个店铺code分区，后面在一个日常走读代码发现这么一个坑，就改成了订单Id作为分区key；

> 根据二八定律，只有那2成的B端流程会使用到系统8成的资源，如果mallCode进行分区，99%概率会出现分区LB问题；
>
> 题外话，另外一个业务时隔不久就暴雷了；



##### 重试机制

你以为上面这样可以解决问题了吗？中低级员工可能一把梭啥都不考虑把代码敲完，之后就开始进入暗无天日的排查故障问题了；

众所周知，一些状态的维护，他是有生命周期的，如果前面状态节点处理失败了，必然会影响到后续状态流转，所以后面其他同事出现了`为啥B端企业下的订单列表丢失、详情细节确实等等`故障问题；

> 就当时没有做`失败重试机制`，使得这个问题被放大了。问题变成：一旦”下单“消息的数据入库失败，C用户就永远看不到这个订单了。



重试机制是异步？还是同步这在当时又是一个问题，因为我接手这个业务的时候，订单中心承担了整个IOT是有的订单业务，你可以理解为，这一个订单中台；



**同步重试机制必然会降低现有商城订单的消费速度，如果使用异步重试机制又得引入重试日志表；**

最后我们模拟很多突发的可能性，根据墨菲定律，最终采用一种思路：

> 先持久化一条重试记录到Redis中，再将消息重发到重试topic中，重试消费成功就把这条Redis记录状态字段修改，设置失效时间，不成功的话，记录某字段加1，继续投递；对此对重试消费我们采用敏感操作，即业务监控通知开发按梯队介入；
>
> 可能有人会问为啥不是MySQL？异构存储啥数据不好管理？
>
> MySQL插入更新幂等使用DUPLICATE KEY UPDATE会增加DB IO；一条消息的重试周期不会跨天；



### 消息挤压问题

随着公司成功上市，品牌效应进一步在销售及各个团队的推广，在某年上半年接入的B端企业数量达到了万级，

一个企业又可以开通不同的店铺，在我们的OEM APP又可以关联对应N个店铺；

红红火火的业务扩张式发展，必然的一些技术债也开始来了（99%公司业务就是这么一个发展周期）；



##### 消息体过大

某个夜晚kakfa监控堆积报警开始夺命连环催，为啥是夜晚？业务是面向全球的；

对堆积报警先入为主就知道了，生产速度变快了；

第二天排查发现我们的堆积数量没有破万，通过`Prometheus+Grafana`消费上限居然被IO限制住了，

众所周知，生产者发送消息到partion先到内存然后定时flush到磁盘，消费又是判断内存是否有这个offset，没有就会load到内存进行堆外内存映射DMR；

从消费者角度来看，拉取的消息是在内存的话，就减少了一次磁盘IO，加上一次网络IO，通过监控对比前几天发现消费者拉取速度从5ms变成了15ms；

有了目标就开始查看这几天的发布单，结果还真的发现上游服务迭代了好几个需求，都藕带了订单内容；

接着通过日志平台发现，额，这日志咋这么长？？？

> MQ在微服务中使用，基本上生产者的序列化器会把自己的对象使用JSON.toJSONString序列化byte数组；
>
> 在消费的时候，当成一个String进行反序列化JSON.parseObject 成一个JSONObject

又是一次对比发现，这个JSON又变大了许多，通过在线JSON页面得翻好两次滚动条；

因为下游进行消费的时候，我们处理的关键字字段不超过5个，而且还是根据策略模式维护状态机；

第二天果断让业务方对消息题进行裁剪，最后发现一个快1MB的对象，最后变成了不到1KB；

第三天一看监控，拉取速度变成了2ms了，可以说，优化得立竿见影。

> 题外话：后面反馈后在框架层对数据大小作了一定限制规范，比如一些中间件；
>
> 避免代码梭的一时爽，修复忍得直接骂人。



##### 批量操作

某个下午突然欧美BD找过来，说他们的订单消息推送有很大的延时；

可能有的同学会奇怪，怎么扯到了消息推送呢？我来讲下抽象后的业务链路：

> C端用户使用了B端企业OEM 商城，进行了商城购买行为，MQ->订单状态机处理，可能触发->对外MQ -> 消息网关规则引擎 -pulsar-> B端企业服务器；
>
> 有较多欧美客户很喜欢使用推送数据进行二次沉淀；有一些特殊的客户对消息的延时是比较在乎，毕竟这也是公司吸引客户的一个亮点；

消息网关投递的消息为啥会变得这么慢，还好公司内部业务的消息我们都有一个业务消息ID，通过工具排除了消息网关末端的嫌疑；

再次来到了监控系统，发现堆积MQ会比较高，但还没达到我们报警阈值（可能上几次狼来了后没及时调整），发现生产者在前面某个小时发送了快100w的消息，消费线程一直满负载跑着，各种DB IO、网络IO、日志IO、RPC IO 量都很大，及联影响着底层RPC Provider相互影响着IO，本来一条消息处理维持在40ms，结果此时就飙到快400ms；



线上问题优先处理，火急让底层加机器硬抗下，把RPC IO降低些，避免影响其他业务，当然还有个防止紧急发布的大招；



我们当时预估了按这个消费速度得好几个小时，因为之前的消费能力大概是之前生产速度的1.5倍，

这个1.5倍能力除了被partion分区数限制了，也有可能被一条消息处理耗时不稳定导致，在微服务架构中有个非常没办法的缺点增加了更多的RPC IO；



我们为什么采用了临时代码使用多线程消费？

> 1、如果直接调大`partition`数量是不行的，历史消息已经存储到3个固定的`partition`，只有新增的消息才会到新的`partition`。我们重点需要处理的是已有的partition。新增分区数下个发布日再操作；
>
> 2、直接加服务节点也不行，因为`kafka`允许同组的多个`partition`被一个`consumer`消费，但不允许一个`partition`被同组的多个`consumer`消费，可能会造成资源浪费。

为了紧急解决问题，改成了用`线程池`处理消息，核心线程和最大线程数配置成了`2(核数)、8`。

> 线程数并不是越大越好，要从监控中得出当前环境压力允许下的一个最快处理速度；

调整之后，果然，不到20分钟，消息积压数量不断减少至0。

也没有把底层的RPC IO比平常飙高到2倍，也避免了一次发布兄弟项目忙着救火；

> 故障的action：
>
> 1、批量操作必须TL审批，无论是界面上的；同时会发送一条批量操作的业务消息到工作群，
>
> 批量操作后会在半个小时后审批才执行。
>
> 2、底层能力引入了多级缓存，维持RPC IO 90%耗时在5ms；
>
> 3、对消息报警采用阶梯化报警策略；



##### 表过大

随着日子一天天过去，风暴总是在不经意中来临，不过这种惬意的日子维持了半年。

订单的消息堆积监控报警又来了，为啥又出现了kafka消息堆积？还有几百的堆积趋势；

> 上一次的紧急发布的多线程消费代码去除了，因为后面分区数扩大了4倍，也就是说比上一次故障多了快4倍的处理能力；
>
> 业务操作也更加规范化；

随便在日志平台通过链路id看了下各阶段耗时，最终定位到了卡在了自身DB IO这块，对比了前几天日志，也就是多了不到5ms，90%耗时有100ms，个别超过500ms；

通过分析生产者速度是上一次故障截图的速度2倍，这个是正常的，在我们允许业务扩张范围内；可惜当时忘记了截图各个阶段的处理耗时；

带着SQL操作变慢的思路，也走了索引，最后发现有个核心表的数量快3千万了，其他数量也是1千万水平；

这次我们可以各种方式紧急加代码发布吗？

这次还真的黔驴技穷了。

> 这个表加缓存？效果不大，因为它要读取实时数据，当然强加也是可以的，但有一半是更新操作，加缓存增加网络IO，还得维护缓存的生命周期，收益可能是负的；
>
> 1个主库，一个备份库（公司要求所有业务库必须要有备份库，分库分表除外）；
>
> 优化索引？从覆盖索引、最左索引规则、索引下推方面看了下，没有操作的空间

最后临时让伟大的DBA对库集群扩容，升了一个等级；

同时给组内增加了一个技术优化的紧急内部需求：分库分表+读库；



##### 其他问题

- 主从延时导致读写数据不一致问题，导致状态机处理失败，额外导致重试压力，导致堆积的可能性；
- 重复消费，kafka`默认的模式是`at least once，所以我们的DB SQL都要求具备一定的幂等能力；
- `kafka`的`consumer`使用自动确认机制，导致`cpu使用率100%`。
- `kafka`集群中的一个`broker`节点挂了，重启不起来；
