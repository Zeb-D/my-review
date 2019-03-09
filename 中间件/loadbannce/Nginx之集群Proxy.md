本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

如今分布式、微服务各种名词兴起与尝试，但这其中带来的各种治理是非常痛苦的；

特别是用户群体的暴增，导致服务端与各种中间件的IO链接数不够，虽然我们服务加了节点，但是存储方面如mongdb 经常会在某些时候，导致我们线上服务经常报数据源不够；

经过分析后，问题描述为：

线上购买了某云的mongdb 实例，但其中提供给最大链接数是1500，是1500，这个数字什么概念？意味着你业务datasource 链接数最多1500，一旦有其中业务处理过长阻塞，那平时正常的业务链接数就已经倒瓶颈，这带来的容错率太低了，经常各种告警；

当然购买这种服务决策不是我们说了定，说白了，我们只是填坑人；

该云的实例虽然有集群分片、副本，这时候看起来就是一台机器一样，只不过这台性能超好，烧钱也多；

> 其中业务代码解决方案：
>
> 使用到该mongdb 数据源 使用熔断降级机制：就是保证单位时间使用io数限制，其它的请求进行降级熔断处理，防止以前请求堆积，导致相关服务不可用；
>
> 这种处理方式，充满了无可奈何，短时候只能先这样

### 问题分析

为了避免这种类似的问题发生性，决定采用 代理+server 作集群，目前开发思路大致和曾经我公司进行单独架构代理，只不过那个时候数**Proxy+Redis Cluster**

对于mongdb 这种问题的 只能采用这种方式，寻找代理层，进行转发 扩容的mongdb 实例（虽然某云提供mongdb 实例也是集群，但链接数的体现就是超强单机）；

接下来我分析以前是怎么弄**Proxy+Redis Cluster**

### 代理架构

redis 官方提供那种集群优缺点如下：

**Redis Cluster：**

**优点**

- 无中心节点
- 数据按照Slot存储分布在多个Redis实例上
- 平滑的进行扩容/缩容节点
- 自动故障转移(节点之间通过Gossip协议交换状态信息,进行投票机制完成Slave到Master角色的提升)
- 降低运维成本，提高了系统的可扩展性和高可用性

**缺点**

- 严重依赖外部Redis-Trib
- 缺乏监控管理
- 需要依赖Smart Client(连接维护, 缓存路由表, MultiOp和Pipeline支持)
- Failover节点的检测过慢，不如“中心节点ZooKeeper”及时
- Gossip消息的开销
- 无法根据统计区分冷热数据
- Slave“冷备”，不能缓解读压力

接下来分析**Proxy+Redis Cluster**

![image-20190309220225633](/Users/yongdongzou/Library/Application Support/typora-user-images/image-20190309220225633.png)

**Smart Client vs Proxy：**

**优点**

Smart Client：

a. 相比于使用代理，减少了一层网络传输的消耗，效率较高。

b. 不依赖于第三方中间件，实现方法和代码自己掌控，可随时调整。

Proxy：

a. 提供一套HTTP Restful接口，隔离底层存储。对客户端完全透明，跨语言调用。

b. 升级维护较为容易，维护Redis Cluster，只需要平滑升级Proxy。

c. 层次化存储，底层存储做冷热异构存储。

d. 权限控制，Proxy可以通过秘钥控制白名单，把一些不合法的请求都过滤掉。并且也可以控制用户请求的超大Value进行控制，和过滤。

e. 安全性，可以屏蔽掉一些危险命令，比如Keys、Save、Flush All等。

f. 容量控制，根据不同用户容量申请进行容量限制。

g. 资源逻辑隔离，根据不同用户的Key加上前缀，来进行资源隔离。

h. 监控埋点，对于不同的接口进行埋点监控等信息。

**缺点**

Smart Client：

a. 客户端的不成熟，影响应用的稳定性，提高开发难度。

b. MultiOp和Pipeline支持有限。

c. 连接维护，Smart客户端对连接到集群中每个结点Socket的维护。

Proxy：

a.  代理层多了一次转发，性能有所损耗。

b．进行扩容/缩容时候对运维要求较高，而且难以做到平滑的扩缩容。



### **为什么选择Nginx开发Proxy**

1. 1.单Master多Work模式，每个Work跟Redis一样都是单进程单线程模式，并且都是基
2. 于Epoll事件驱动的模式。
3. Nginx采用了异步非阻塞的方式来处理请求，高效的异步框架。
4. 内存占用少，有自己的一套内存池管理方式,。将大量小内存的申请聚集到一块，能够比Malloc 更快。减少内存碎片，防止内存泄漏。减少内存管理复杂度。
5. 为了提高Nginx的访问速度，Nginx使用了自己的一套连接池。
6. 最重要的是支持自定义模块开发。
7. 业界内，对于Nginx，Redis的口碑可称得上两大神器。性能也就不用说了。



### Proxy+Redis Cluster介绍

#### **Proxy+Redis Cluster架构方案介绍**

1. 用户在ACL平台申请集群资源，如果申请成功返回秘钥信息。

2. 用户请求接口必须包含申请的秘钥信息，请求至LVS服务器。

3. LVS根据负载均衡策略将请求转发至Nginx Proxy。

4. Nginx Proxy首先会获取秘钥信息，然后根据秘钥信息去ACL服务上获取集群的种子信息。(种子信息是集群内任意几台IP:PORT节点)

然后把秘钥信息和对应的集群种子信息缓存起来。并且第一次访问会根据种子IP:PORT获取集群Slot对应节点的Mapping路由信息，进行缓存起来。最后根据Key计算SlotId，从缓存路由找到节点信息。

5. 把相应的K/V信息发送到对应的Redis节点上。

6. Nginx Proxy定时(60s)上报请求接口埋点的QPS,RT,Err等信息到Open-Falcon平台。

7. Redis Cluster定时(60s)上报集群相关指标的信息到Open-Falcon平台。

![image-20190309220511744](/Users/yongdongzou/Library/Application Support/typora-user-images/image-20190309220511744.png)



#### **Nginx Proxy功能介绍**

目前支持的功能：

**HTTP Restful接口：**

 解析用户Post过来的数据， 并且构建Redis协议。客户端不需要开发Smart Client, 对客户端完全透明、跨语言调用

**权限控制：**

根据用户Post数据获取AppKey,Uri, 然后去ACL Service服务里面进行认证。如果认证通过，会给用户返回相应的集群种子IP，以及相应的过期时间限制等信息

 **限制数据大小：**

获取用户Post过来的数据，对Key，Value长度进行限制，避免产生超大的Key,Value，打满网卡、阻塞Proxy

 **数据压缩/解压：**

如果是写请求，对Value进行压缩(Snappy)，然后在把压缩后的数据存储到Redis Cluster。

如果是读请求，把Value从Redis Cluster读出来，然后对Value进行解压，最后响应给用户。

 **缓存路由信息：**

维护路由信息，Slot对应的节点的Mapping信息

 **结果聚合：**

MultiOp支持

批量指令支持(Pipeline/Redis+Lua+EVALSHA进行批量指令执行)

 **资源逻辑隔离：**

 根据用户Post数据获取该用户申请的NameSpace，然后以NameSpace作为该用户请求Key的前缀，从而达到不同用户的不同NameSpace，进行逻辑资源隔离

 **重试策略：**

针对后端Redis节点出现Moved,Ask,Err,TimeOut等进行重试，重试次数可配置

 **连接池：**

维护用户请求的长连接，维护后端服务器的长连接

 **配额管理：**

根据用户的前缀(NameSpace), 定时的去抓取RANDOMKEY，根据一定的比率，估算出不同用户的容量大小值，然后在对用户的配额进行限制管理

 **过载保护：**

通过在Nginx Proxy Limit模块进行限速，超过集群的承载能力，进行过载保护。从而保证部分用户可用，不至于压垮服务器

 **监控管理：**

Nginx Proxy接入了Open-Falcon对系统级别，应用级别，业务级别进行监控和告警

例如： 接口的QPS,RT,ERR等进行采集监控，并且展示到DashBoard上

告警阈值的设置非常灵活，配置化

 

待开发的功能列表：

**层次化存储：**

利用Nginx Proxy共享内存定制化开发一套LRU本地缓存实现，从而减少网络请求

冷数据Swap到慢存储，从而实现冷热异构存储

 **主动Failover节点：**

由于Redis Cluster是通过Gossip通信, 超过半数以上Master节点通信(cluster-node-timeout)认为当前Master节点宕机，才真的确认该节点宕机。判断节点宕机时间过长，在Proxy层加入Raft算法，加快失效节点判定，主动Failover



### **Nginx Proxy性能优化**

### **批量接口优化方案**

**1. 子请求变为协程**

 案例：

用户需求调用批量接口mget(50Key)要求性能高，吞吐高，响应快。

 问题：

由于最早用的Nginx Subrequest来做批量接口请求的处理，性能一直不高，CPU利用率也不高，QPS提不起来。通过火焰图观察分析子请求开销比较大。

 解决方案：

子请求效率较低，因为它需要重新从Server Rewrite开始走一遍Request处理的PHASE。并且子请求共享父请求的内存池，子请求同时并发度过大，导致内存较高。

协程轻量级的线程，占用内存少。经过调研和测试，单机一两百万个协程是没有问题的，

并且性能也很高。

 

**优化前：**

a) 用户请求mget(k1,k2)到Proxy

b) Proxy根据k1,k2分别发起子请求subrequest1,subrequest2

c) 子请求根据key计算slotid，然后去缓存路由表查找节点

d) 子请求请求Redis Cluster的相关节点，然后响应返回给Proxy

e) Proxy会合并所有的子请求返回的结果，然后进行解析包装返回给用户

![image-20190309221429750](/Users/yongdongzou/Library/Application Support/typora-user-images/image-20190309221429750.png)

**优化后：**

a) 用户请求mget(k1,k2)到Proxy

b) Proxy根据k1,k2分别计算slotid, 然后去缓存路由表查找节点

c) Proxy发起多个协程coroutine1, coroutine2并发的请求Redis Cluster的相关节点

d) Proxy会合并多个协程返回的结果，然后进行解析包装返回给用户

![image-20190309221502114](/Users/yongdongzou/Library/Application Support/typora-user-images/image-20190309221502114.png)

**2. 合并相同槽，批量执行指令，减少网络开销**

 案例：

用户需求调用批量接口mget(50key)要求性能高，吞吐高，响应快。

 问题：

经过上面协程的方式进行优化后，发现批量接口性能还是提升不够高。通过火焰图观察分析网络开销比较大。

解决方案：

因为在Redis Cluster中，批量执行的key必须在同一个slotid。所以，我们可以合并相同slotid的key做为一次请求。然后利用Pipeline/Lua+EVALSHA批量执行命令来减少网络开销，提高性能。

 

**优化前：**

a) 用户请求mget(k1,k2,k3,k4) 到Proxy。

b) Proxy会解析请求串，然后计算k1,k2,k3,k4所对应的slotid。

c) Proxy会根据slotid去路由缓存中找到后端服务器的节点，并发的发起多个请求到后端服务器。

d) 后端服务器返回结果给Proxy,然后Proxy进行解析获取key对应的value。

e) Proxy把key,value对应数据包装返回给用户。



![image-20190309221530825](/Users/yongdongzou/Library/Application Support/typora-user-images/image-20190309221530825.png)

**优化后：**

a) 用户请求mget(k1,k2,k3,k4) 到Proxy。

b) Proxy会解析请求串，然后计算k1,k2,k3,k4所对应的slotid，然后把相同的slotid进行合并为一次Pipeline请求。

c) Proxy会根据slotid去路由缓存中找到后端服务器的节点，并发的发起多个请求到后端服务器。

d) 后端服务器返回结果给Proxy,然后Proxy进行解析获取key对应的value。

e) Proxy把key,value对应数据包装返回给用户。

![image-20190309221550920](/Users/yongdongzou/Library/Application Support/typora-user-images/image-20190309221550920.png)

**3. 对后端并发度的控制**

 案例：

当用户调用批量接口请求mset，如果k数量几百个甚至几千个时，会导致Proxy瞬间同时发起几百甚至几千个协程同时去访问后端服务器Redis Cluster。

 问题：

Redis Cluster同时并发请求的协程过多，会导致连接数瞬间会很大，甚至超过上限，CPU,连接数忽高忽低，对集群造成不稳定。

 解决方案：

单个批量请求对后端适当控制并发度进行分组并发请求，反向有利于性能提升，避免超过Redis Cluster连接数，同时Redis Cluster 波动也会小很多，更加的平滑。

 

 **优化前：**

a) 用户请求批量接口mset(200个key)。(这里先忽略合并相同槽的逻辑)

b) Proxy会解析这200个key，会同时发起200个协程请求并发的去请求Redis Cluster。

c) Proxy等待所有协程请求完成，然后合并所有协程请求的响应结果，进行解析，包装返回给用户。

![image-20190309221608234](/Users/yongdongzou/Library/Application Support/typora-user-images/image-20190309221608234.png)



**优化后：**

a) 用户请求批量接口mset(200个key)。 (这里先忽略合并相同槽的逻辑)

b) Proxy会解析这200个key，进行分组。100个key为一组，分批次进行并发请求。

c) Proxy先同时发起第一组100个协程(coroutine1, coroutine100)请求并发的去请求Redis Cluster。

d) Proxy等待所有协程请求完成，然后合并所有协程请求的响应结果。

e) Proxy然后同时发起第二组100个协程(coroutine101, coroutine200)请求并发的去请求Redis Cluster。

f) Proxy等待所有协程请求完成，然后合并所有协程请求的响应结果。

g) Proxy把所有协程响应的结果进行解析，包装，返回给用户。

![image-20190309221623094](/Users/yongdongzou/Library/Application Support/typora-user-images/image-20190309221623094.png)



**4.单Work分散到多Work**



案例：

当用户调用批量接口请求mset，如果k数量几百个甚至几千个时，会导致Proxy瞬间同时发起几百甚至几千个协程同时去访问后端服务器Redis Cluster。

 问题：

由于Nginx的框架模型是单进程单线程, 所以Proxy发起的协程都会在一个Work上,这样如果发起的协程请求过多就会导致单Work CPU打满，导致Nginx 的每个Work CPU使用率非常不均，内存持续暴涨的情况。(nginx 的内存池只能提前释放大块，不会提前释放小块)

 解决方案：

增加一层缓冲层代理，把请求的数据进行拆分为多份，然后每份发起请求，控制并发度，在转发给Proxy层，避免单个较大的批量请求打满单Work，从而达到分散多Work，达到Nginx 多个Wrok CPU使用率均衡。

 

**优化前：**

a) 用户请求批量接口mset(200个key)。(这里先忽略合并相同槽的逻辑)

b) Proxy会解析这200个key，会同时发起200个协程请求并发的去请求Redis Cluster。

c) Proxy等待所有协程请求完成，然后合并所有协程请求的响应结果，进行解析，包装返回给用户。

![image-20190309221642235](/Users/yongdongzou/Library/Application Support/typora-user-images/image-20190309221642235.png)

**优化后：**

a) 用户请求批量接口mset(200个key)。(这里先忽略合并相同槽的key的逻辑)

b) Proxy会解析这200个key，然后进行拆分分组以此来控制并发度。

c) Proxy会根据划分好的组进行一组一组的发起请求。

d) Proxy等待所有请求完成，然后合并所有协程请求的响应结果，进行解析，包装返回给用户。

![image-20190309221656071](/Users/yongdongzou/Library/Application Support/typora-user-images/image-20190309221656071.png)

总结，经过上面一系列优化，我们可以来看看针对批量接口mset(50个k/v)性能对比图，Nginx Proxy的响应时间比Java版本的响应时间快了5倍多。



#### 参考

https://mp.weixin.qq.com/s?__biz=MzU1NDA4NjU2MA==&mid=2247486494&idx=1&sn=f8bbe491a0569445e7fc065112361b33&source=41#wechat_redirect



