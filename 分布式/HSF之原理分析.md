# HSF的原理分析

参考博客：https://blog.csdn.net/qq_16681169/article/details/72512819

<br><br>

## 基本概念

HSF全称为High-Speed Service Framework，旨在为淘系的应用提供一个分布式的服务框架，HSF从分布式应用层面以及统一的发布/调用方式层面为大家提供支持，从而可以很容易的开发分布式的应用以及提供或使用公用功能模块，而不用考虑分布式领域中的各种细节技术，例如远程通讯、性能损耗、调用的透明化、同步/异步调用方式的实现等等问题。

<br><br>

## 涉及技术点

### 序列化

对象的序列化过程在RPC过程中是很重要的一个环节，序列化对于远程调用的响应速度、吞吐量、网络带宽消耗等同样也起着至关重要的作用。
在HSF1.0时只支持两种序列化方式：java 和 hessian，在HSF2.0之后就支持了五种序列化方式：java、hessian、hessian2、json、kyro。
但是目前版本中常用的序列化方式还是java 和 hessian两种。如果想细致的了解也可以多做了解。

### 动态代理

对于消费方来说，所存在的只有一个接口，虽然底层的实现原理我们知道，但是为了在使用时的高度透明，在JAVA语言层面上的表现形式则是通过动态代理的方式实现的，很多的逻辑都在InvocationHandler 中处理的。关于如何实现动态代理，还动态代理的一些使用的细节也可以稍作了解。

### 网络通信NIO

如果在网络传输过程中，采取普通的BIO,会有很多的问题存在，例如如果调用端有多个请求过来，那么就得需要多个线程去处理，每个线程都使用独立的连接，在远端的提供者端有对应的多个线程来执行相应的服务。这种方式会使得调用者和提供者之间建立大量的连接，而且是阻塞的方式，连接并不能得到充分的利用(摘自《大型网站系统与JAVA中间件》)。采用NIO则就可以避免这样的损耗，但是HSF在使用时并不是采用直接的NIO编程，而是通过第三方的框架Netty。

<br><br>

## 实现原理

### 流程

 server启动时候向configserver注册
• client启动时候向configserver请求list
• client缓存list，发现不可用的server，从缓存中remove
• configserver通过心跳包维护可用server的list
• list有更新的时候，configserver通过带version的报文通知client更新

![img](https://img-blog.csdn.net/20170518235624390?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTY2ODExNjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

https://img-blog.csdn.net/20170518235718484?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTY2ODExNjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center

**ConfigServer**：远程调用对端的地址就是由ConfigServer 来推送的，这样用户只需要配置自己的服务端或者消费端，不需要对自己的地址进行管理。

**Diamond**：持久化的配置中心，用于配置服务调用的规则。

**服务：**服务是调用方和提供方交流的依凭，一般是一个接口，表示一个业务行为以及相关的数据含义。通过使用HSFApiProviderBean能够暴露一个服务，将机器的地址注册到configserver，并且能够通过12200端口进行服务提供，通过HSFApiConsumerBean能够包装出一个客户端，它是服务接口的一个代理，并且它从configserver上订阅了服务的地址列表，能够在这个列表上完成随机调用，做到负载均衡与HA((High Available,高可用性群集)。

**网络通信:**HSF的底层网络通信是使用netty框架实现的，是基于epoll的NIO的网络通讯框架，HSF在此使用的是长连接，通过合理的服务部署及负债均衡，基本不存在I/O方面的限制。

<br><br>

## 架构

![img](https://img-blog.csdn.net/20170518235753109?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTY2ODExNjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

Server端除了configServer外还有一个diamond用来保存一些持久化的配置信息，这里不进行过多的介绍。

Client是HSF的重点，下面是各模块的功能介绍：
Proxy：这一层主要负责接口的代理。基本上所有的RPC框架都会用到代理模式，相信大家不陌生。需要注意的是HSF的代理层还进行了软负载和单元化的处理。
Remoting：这一层是HSF的应用层协议，定义了报文格式，各个字段的含义等信息，内容比较多，之后单独写一篇文章来介绍。
Processer：这一层主要是处理HSF自身的业务逻辑，包括埋点、限流、鉴权等。
Netty：上面三层会将一次服务调用或者服务返回包装成一个报文，然后通过这层传输。

### 调用流程

![img](https://img-blog.csdn.net/20170518235829391?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTY2ODExNjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

第一条线路相当于consumer进行服务调用的过程，首先经过proxy层，将请求经过代理类包装出去；然后是Remoting层进行协议的包装，最后io层发送出去。
第二条线路相当于provider将结果返回后解析的过程，与上一流程刚好相反。
右边的provider两条调用流程相信大家都能按照上面的过程理解，就不一一讲解了。









