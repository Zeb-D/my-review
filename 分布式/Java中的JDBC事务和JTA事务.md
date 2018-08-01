# Java——JDBC事务和JTA事务



Java事务的类型有三种：`JDBC事务`、`JTA(Java Transaction API)事务`、`容器事务`。 常见的容器事务如Spring事务，容器事务主要是J2EE应用服务器提供的，容器事务大多是基于JTA完成，这是一个基于JNDI的，相当复杂的API实现。所以本文暂不讨论容器事务。本文主要介绍J2EE开发中两个比较基本的事务：`JDBC事务`和`JTA事务`。

## JDBC事务

JDBC的一切行为包括事务是基于一个`Connection`的，在JDBC中是通过`Connection`对象进行事务管理。在JDBC中，常用的和事务相关的方法是： `setAutoCommit`、`commit`、`rollback`等。

[![Java_jdbc](http://www.hollischuang.com/wp-content/uploads/2016/08/Java_jdbc.png)](../image/Java_jdbc.png)

下面看一个简单的JDBC事务代码：

```java
public void JdbcTransfer() { 
    java.sql.Connection conn = null;
     try{ 
        conn = conn =DriverManager.getConnection("jdbc:oracle:thin:@host:1521:SID","username","userpwd"）;
         // 将自动提交设置为 false，
         //若设置为 true 则数据库将会把每一次数据更新认定为一个事务并自动提交
         conn.setAutoCommit(false);

         stmt = conn.createStatement(); 
         // 将 A 账户中的金额减少 500 
         stmt.execute("\
         update t_account set amount = amount - 500 where account_id = 'A'");
         // 将 B 账户中的金额增加 500 
         stmt.execute("\
         update t_account set amount = amount + 500 where account_id = 'B'");

         // 提交事务
         conn.commit();
         // 事务提交：转账的两步操作同时成功
     } catch(SQLException sqle){            
         try{ 
             // 发生异常，回滚在本事务中的操做
            conn.rollback();
             // 事务回滚：转账的两步操作完全撤销
             stmt.close(); 
             conn.close(); 
         }catch(Exception ignore){ 

         } 
         sqle.printStackTrace(); 
     } 
}
```

上面的代码实现了一个简单的转账功能，通过事务来控制转账操作，要么都提交，要么都回滚。

### JDBC事务的优缺点

JDBC为使用Java进行数据库的事务操作提供了最基本的支持。通过JDBC事务，我们可以将多个SQL语句放到同一个事务中，保证其ACID特性。JDBC事务的主要优点就是API比较简单，可以实现最基本的事务操作，性能也相对较好。

但是，JDBC事务有一个局限：**一个 JDBC 事务不能跨越多个数据库！！！**所以，如果涉及到多数据库的操作或者分布式场景，JDBC事务就无能为力了。

<br>

##JTA事务

###为什么需要JTA

通常，JDBC事务就可以解决数据的一致性等问题，鉴于他用法相对简单，所以很多人关于Java中的事务只知道有JDBC事务，或者有人知道框架中的事务（比如Hibernate、Spring）等。但是，由于JDBC无法实现分布式事务，而如今的分布式场景越来越多，所以，JTA事务就应运而生。

如果，你在工作中没有遇到JDBC事务无法解决的场景，那么只能说你做的项目还都太小。拿电商网站来说，我们一般把一个电商网站横向拆分成商品模块、订单模块、购物车模块、消息模块、支付模块等。然后我们把不同的模块部署到不同的机器上，各个模块之间通过远程服务调用(RPC)等方式进行通信。以一个分布式的系统对外提供服务。

一个支付流程就要和多个模块进行交互，每个模块都部署在不同的机器中，并且每个模块操作的数据库都不一致，这时候就无法使用JDBC来管理事务。我们看一段代码：

```java
/** 支付订单处理 **/
@Transactional(rollbackFor = Exception.class)
public void completeOrder() {
    orderDao.update(); // 订单服务本地更新订单状态
    accountService.update(); // 调用资金账户服务给资金帐户加款
    pointService.update(); // 调用积分服务给积分帐户增加积分
    accountingService.insert(); // 调用会计服务向会计系统写入会计原始凭证
    merchantNotifyService.notify(); // 调用商户通知服务向商户发送支付结果通知
}
```

上面的代码是一个简单的支付流程的操作，其中调用了五个服务，这五个服务都通过RPC的方式调用，请问使用JDBC如何保证事务一致性？我在方法中增加了`@Transactional`注解，但是由于采用调用了分布式服务，该事务并不能达到ACID的效果。

JTA事务比JDBC事务更强大。一个JTA事务可以有多个参与者，而一个JDBC事务则被限定在一个单一的数据库连接。下列任一个Java平台的组件都可以参与到一个JTA事务中：`JDBC`连接、`JDO PersistenceManager` 对象、`JMS` 队列、`JMS` 主题、企业JavaBeans（`EJB`）、一个用`J2EE Connector Architecture` 规范编译的资源分配器。

###JTA的定义

Java事务API（`Java Transaction API`，简称JTA ） 是一个Java企业版 的应用程序接口，在Java环境中，允许完成跨越多个XA资源的分布式事务。

![JTA](http://www.hollischuang.com/wp-content/uploads/2016/08/JTA.png)

JTA和它的同胞Java事务服务(JTS；Java TransactionService)，为J2EE平台提供了分布式事务服务。不过JTA只是提供了一个接口，并没有提供具体的实现，而是由j2ee服务器提供商 根据JTS规范提供的，常见的JTA实现有以下几种：

- 1.J2EE容器所提供的JTA实现(JBoss)
- 2.独立的JTA实现:如JOTM，Atomikos.这些实现可以应用在那些不使用J2EE应用服务器的环境里用以提供分布事事务保证。如Tomcat,Jetty以及普通的java应用。

JTA里面提供了 `java.transaction.UserTransaction` ,里面定义了下面几个方法

> `begin`：开启一个事务
>
> `commit`：提交当前事务
>
> `rollback`：回滚当前事务
>
> `setRollbackOnly`：把当前事务标记为回滚
>
> `setTransactionTimeout`：设置事务的事件，超过这个事件，就抛出异常，回滚事务

这里，值得注意的是，不是使用了`UserTransaction`就能把普通的JDBC操作直接转成JTA操作，JTA对DataSource、Connection和Resource 都是有要求的，只有符合[XA规范](http://www.hollischuang.com/archives/681)，并且实现了XA规范的相关接口的类才能参与到JTA事务中来，这里，提一句，目前主流的数据库都支持XA规范。

要想使用用 JTA 事务，那么就需要有一个实现 `javax.sql.XADataSource` 、 `javax.sql.XAConnection` 和 `javax.sql.XAResource` 接口的 JDBC 驱动程序。一个实现了这些接口的驱动程序将可以参与 JTA 事务。一个 `XADataSource` 对象就是一个 `XAConnection` 对象的工厂。`XAConnection` 是参与 JTA 事务的 JDBC 连接。

要使用JTA事务，必须使用`XADataSource`来产生数据库连接，产生的连接为一个XA连接。

XA连接（`javax.sql.XAConnection`）和非XA（`java.sql.Connection`）连接的区别在于：XA可以参与JTA的事务，而且不支持自动提交。

<br>

##标准的分布式事务

> 一个分布式事务（Distributed Transaction）包括一个**事务管理器**（`transaction manager`）和一个或多个**资源管理器**(`resource manager`)。一个**资源管理器**（`resource manager`）是任意类型的持久化数据存储。**事务管理器**（`transaction manager`）承担着所有事务参与单元者的相互通讯的责任。

JTA的实现方式也是基于以上这些分布式事务参与者实现的，具体的关于JTA的实现细节不是本文的重点，感兴趣的同学可以阅读[JTA 深度历险 – 原理与实现](https://www.ibm.com/developerworks/cn/java/j-lo-jta/)

- 看上面关于分布式事务的介绍是不是和2PC中的事务管理比较像？的却，2PC其实就是符合XA规范的事务管理器协调多个资源管理器的一种实现方式。 我之前有几篇文章关于2PC和3PC的，那几篇文章中介绍过分布式事务中的事务管理器是如何协调多个事务的统一提交或回滚的，后面我还会有几篇文章详细的介绍一下和分布式事务相关的内容，包括但不限于全局事务、DTP模型、柔性事务等。

##JTA的优缺点

JTA的优点很明显，就是提供了分布式事务的解决方案，严格的ACID。但是，标准的JTA方式的事务管理在日常开发中并不常用，因为他有很多缺点:

- 实现复杂
  - 通常情况下，JTA UserTransaction需要从JNDI获取。这意味着，如果我们使用JTA，就需要同时使用JTA和JNDI。
- JTA本身就是个笨重的API
- 通常JTA只能在应用服务器环境下使用，因此使用JTA会限制代码的复用性。

## 总结

Java事务的类型有三种：`JDBC事务`、`JTA(Java Transaction API)事务`、`容器事务`，其中JDBC的事务操作用法比较简单，适合于处理同一个数据源的操作。JTA事务相对复杂，可以用于处理跨多个数据库的事务，是分布式事务的一种解决方案。

总之，虽然JTA事务是Java提供的可用于分布式事务的一套API，但是不同的J2EE平台的实现都不一样，并且都不是很方便使用，所以，一般在项目中不太使用这种较为负责的API。现在业内比较常用的分布式事务解决方案主要有异步消息确保型、TCC、最大努力通知等。

参考链接：http://www.hollischuang.com/archives/1658