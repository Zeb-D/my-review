本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 基础概念

#### 什么是RPC

Remote Procedure Call 即远程过程调用，对标的是本地过程调用。啥是本地过程调用?你实现的方法与方法之间 的调用，就是本地过程调用，所以你本地方法调用了远程机器上的某个程序的某个方法，就是远程过程调用。

RPC跟HTTP是啥关系?RPC调用远程的方法，可以通过HTTP来实现，也可以基于TCP你自定义协议。RPC跟HTTP 根本不是同一个层级的东西!



#### 什么是dubbo

dubbo是一个分布式的服务框架，致力于提高性能和透明化的RPC远程服务调用方案，以及SOA服务治理方案。

简言之，dubbo就是一个服务框架，如果没有分布式的需求，其实不需要用的，只有分布式的时候，才需要dubbo这样的分布式框架；



#### RPC框架

RPC框架的目的是啥?它主要是为了能让你像使用本地过程调用一样简单方便的进行远程过程调用，让用户无感知。它会解决掉远程调用过程中会遇到的一些问题，同时还会提供更多的类似远程调用监控等功能。

那如何设计一个合格的RPC框架呢?

从底而上，既然你需要远程调用，那第一步必不可免的就是网络调用，你需 要知道对方的IP+端口。

网络连接问题解决后，具体该通过什么协议进行网络连接和交互呢?所以你需要确定协议。

ok，通道问题解决了，

接下来就是数据传输，我们需要把数据转换为二进制字节流，这就需要序列化工具了。 

序列化问题解决后，接下来其实就是业务了，

当你开放一个API接口的时候，第一步是啥，肯定是定义好入参和返 回参数的通用格式吧!(Request和Response)序列化前和反序列化后的请求和响应数据，需要用统一的格式封 装。

封装完了之后，其实用户还是不太愿意关心这个数据，也不关心这个调用流程，最好还有个统一封装:将数据 通信交互的过程封装为invoke调用。

有了invoke，距离用户想要的已经很近了，我们可以通过动态代理的方式帮 助客户实现动态进行对应的invoke调用。

走到这，一个基本的RPC框架就基本成型了，这个时候我们再简单优化一 下，给invoke远程调用过程添加一些监控，协助我们对框架的管控。

同时，最底层的IP+端口，我们不可能每次都 要修改，这就需要一个动态发现和暴露的服务。

最后，因为远程调用的不稳定性，我们肯定要处理呀，那我们就做 一个集群容错层，专门来负责负载均衡和容错策略。



#### 服务提供者

提供者实现具体的业务逻辑，同时通过注册中心暴露自己的服务。

如果有请求过来需要处理，提供者先根据约定好 的协议接收数据后，进行反序列化，然后丢进自己的线程池去处理。

某个线程接收到任务后去根据类+方法名+参 数，找到这个对应的方法，通过早已经动态代理好的代理类，调用服务实现逻辑，然后获取到返回值后原路返回。



#### 注册中心

注册中心主要是方便服务提供者和消费者的信息动态更新和维护，所以要成为一个注册中心，你起码得有数据存

储、数据变更通知等能力。Dubbo默认是zookeeper。



### Dubbo架构

首先我们来看官网提供的一张图:

![dubbo-base-arch.png](../../../image/dubbo-base-arch.png)

简单描述一下大致的过程:

0.服务器负责启动、加载、运行服务提供者

1.服务提供者在启动时，向注册中心注册自己提供的服务

2.服务消费者在启动时，向注册中心订阅自己所需的服务

3.注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者

4.服务消费者，从提供者列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，更换另一台调用

5.服务消费者和服务提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心



### Dubbo分层架构

Dubbo总体分三层，细分为十层。同时这十层也可以分为API层和SPI层。

![dubbo-base-arch-flow.png](../../../image/dubbo-base-arch-flow.png)

大三层分为 Business(业务层)、RPC 层、Remoting层。

这是跟网络分层一样的意思，只有层次分明，职责边界 清晰才能更好的扩展。

- API层和SPI层，是Dubbo的成功之处，具体的我们后面在介绍。 我们先看看这十层具体是干啥的:
- Service，业务层，就是咱们开发的业务逻辑层。
-  Config，配置层，主要围绕 ServiceConfig 和 ReferenceConfig，初始化配置信息。
- Proxy，代理层，服务提供者还是消费者都会生成一个代理类，使得服务接口透明化，代理层做远程调用和返 回结果。
- Register，注册层，封装了服务注册和发现。 
- Cluster，路由和集群容错层，负责选取具体调用的节点，处理特殊的调用要求和负责远程调用失败的容错措施。
- Monitor，监控层，负责监控统计调用时间和次数。
- Portocol，远程调用层，主要是封装 RPC 调用，主要负责管理Invoker，Invoker代表一个抽象封装了的执行 体，之后再做详解。
- Exchange，信息交换层，用来封装请求响应模型，同步转异步。 Transport，网络传输层，抽象了网络传输的统一接口，这样用户想用Netty 就用 Netty，想用 Mina 就用Mina。
- Serialize，序列化层，将数据序列化成二进制流，当然也做反序列化。 可以结合我上面介绍RPC框架的一个设计过程，能更好的理解Dubbo的这些分层的用意。



### SPI

SPI全称Service Provider Interface，是Java提供的一套用来被第三方实现或者扩展的API，它可以用来启用框架扩展和替换组件。
整体机制图如下:

![dubbo-base-spi.png](../../../image/dubbo-base-spi.png)

Java SPI 实际上是“基于接口的编程+策略模式+配置文件”组合实现的动态加载机制。 

一个系统设计的一些抽象模块，往往有很多不同的实现，Java面向对象的设计中，是推荐面向接口编程，业务代码

中不应该对具体的实现类进行硬编码，因为这会违反可插拔的原则。 

为了满足在模块装配的时候能不在程序里动态指明，这就需要一种服务发现机制:SPI，其主要目的就是解耦。 SPI使用场景:

数据库驱动加载接口实现类的加载:JDBC加载不同类型数据库的驱动。

日志门面接口实现类加载:SLF4J加载不同提供商的日志实现类。

要使用Java SPI，需要遵循如下约定:

- 当服务提供者提供了接口的一种具体实现后，在jar包的META-INF/services目录下创建一个以“接口全限名”为命名的文件，内容为实现类的全限定名;
- 接口实现类所在的jar包放在主程序的classpath中; 
- 主程序通过java.util.ServiceLoder动态装载实现模块，它通过扫描META-INF/services目录下的配置文件找到实现类的全限定名，把类加载到JVM;
- SPI的实现类必须携带一个不带参数的构造方法;



那么dubbo中spi对应实现为[dubbo-spi机制详解.md](./dubbo-spi机制详解.md);



### URL

URL也就是Uniform Resource Locator，中文叫统一资源定位符。Dubbo中无论是服务消费方，或者服务提供方，或者注册中心。都是通过URL进行定位资源的。所以今天来聊聊Dubbo中的统一URL资源模型是怎么样的。

```
protocol://username:password@host:port/path?key=value&key=value
```



#### dubbo URL

```java
public URL(String protocol, String username, String password, String host, int port, String path, Map<String, String> parameters) {
   if ((username == null || username.length() == 0)
         && password != null && password.length() > 0) {
      throw new IllegalArgumentException("Invalid url, password without username!");
   }
   this.protocol = protocol;
   this.username = username;
   this.password = password;
   this.host = host;
   this.port = (port < 0 ? 0 : port);
   this.path = path;
   // trim the beginning "/"
   while(path != null && path.startsWith("/")) {
       path = path.substring(1);
   }
   if (parameters == null) {
       parameters = new HashMap<String, String>();
   } else {
       parameters = new HashMap<String, String>(parameters);
   }
   this.parameters = Collections.unmodifiableMap(parameters);
}
```

可以从上面源码看出：

- - protocol：一般是 dubbo 中的各种协议 如：dubbo、thrift、http、zk
  - username/password：用户名/密码
  - host/port：主机IP地址/端口号
  - path：接口名称
  - parameter：参数键值对

大致样子如下：

```
dubbo://192.168.1.6:20880/com.yd.scala.HelloService?timeout=3000
描述一个 dubbo 协议的服务

zookeeper://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=demo-consumer&dubbo=2.0.2&interface=org.apache.dubbo.registry.RegistryService&pid=1214&qos.port=33333&timestamp=1545721981946
描述一个 zookeeper 注册中心

consumer://30.5.120.217/org.apache.dubbo.demo.DemoService?application=demo-consumer&category=consumers&check=false&dubbo=2.0.2&interface=org.apache.dubbo.demo.DemoService&methods=sayHello&pid=1209&qos.port=33333&side=consumer&timestamp=1545721827784
描述一个消费者
```



#### Dubbo中有关URL的服务

**1、解析服务**

- - Spring在遇到dubbo名称空间时，会回调DubboNamespaceHandler。这个类也是Dubbo基于spring扩展点编写的解析xml文件的类。
  - 解析的xml标签使用DubboBeanDefinitionParser将其转化为bean对象。
  - 服务提供方在ServiceConfig.export()初始化时将bean对象转化为URL格式，所有Bean属性转换成URL参数。这时候的URL就会传给协议扩展点。根据URL中protocol的值通过扩展点自适应机制进行不同协议的服务暴露或引用。
  - 而服务消费方则是ReferenceConfig.export()方法。

**2、直接暴露服务端口**

- - 在没有注册中心时，ServiceConfig解析出的URL格式为：`dubbo://service-host/com.yd.FooService?version=1.0.0`
  - 基于扩展点自适应机制。通过URL的dubbo://协议头识别，这时候就调用DubboProtocol中的export方法进行暴露服务端口

**3、向注册中心暴露服务端口**

- - 有注册中心时。ServiceConfig解析出的URL格式就类似：`registry://registry-host/org.apache.dubbo.registry.RegistryService?export=URL.encode("dubbo://service-host/com.yd.FooService?version=1.0.0")`
  - 基于扩展点自适应机制，识别到URL以registry://开头，就会调用RegistryProtocol中的export方法先将该URL注册到注册中心里
  - 再传给Protocol扩展点进行暴露，这时候就只剩下`dubbo://service-host/com.yd.FooService?version=1.0.0`。同样的基于dubbo://协议头识别，通过DubboProtocol的export方法打开服务端口

**4、直接引用服务**

- - 在没有注册中心，ReferenceConfig解析出的URL格式就为dubbo://service-host/com.foo.FooService?version=1.0.0
  - 基于扩展点自适应机制，通过 URL 的dubbo://协议头识别，直接调用DubboProtocol的refer方法，返回提供者引用

**5、从注册中心引用服务**

- - 有注册中心时，ReferenceCofig解析出来的URL格式为：`registry://registry-host/org.apache.dubbo.registry.RegistryService?refer=URL.encode("consumer://consumer-host/com.yd.FooService?version=1.0.0")`
  - 同样先识别URL的协议头，调用RegistryProtocol中的refer方法
  - 通过refer参数中的条件查询到提供者的URL。如`dubbo://service-host/com.yd.FooService?version=1.0.0`。此时就会调用DubboProtocol中的refer方法得到提供者引用
  - 最后若是存在集群Cluster扩展点，需要伪装成单个提供者引用返回



### Invoker

在dubbo中，万物皆是Invoker，即便是Exporter也是由Invoker进化而成的

> Protocol 是服务域，它是 Invoker 暴露和引用的主功能入口，它负责 Invoker 的生命周期管理。
>
> Invoker 是实体域，它是 Dubbo 的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起 invoke 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。
>
> Invocation 是会话域，它持有调用过程中的变量，比如方法名，参数等。
