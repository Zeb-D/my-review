# Spring事务管理详解

<br>

## 事务概念

事物是逻辑上的一组操作，要么都执行，要么都不执行.

### 事物的特性（ACID）

1. **原子性：** 事物是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么完全不起作用；
2. **一致性：** 执行事物前后，数据保持一致；
3. **隔离性：** 并发访问数据库时，一个用户的事物不被其他事物所干扰，各并发事物之间数据库是独立的；
4. **持久性:** 一个事物被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。

<br>

## Spring事务管理接口介绍

###接口

- **PlatformTransactionManager：** （平台）事务管理器
- **TransactionDefinition：** 事务定义信息(事务隔离级别、传播行为、超时、只读、回滚规则)
- **TransactionStatus：** 事务运行状态

**所谓事务管理，其实就是“按照给定的事务规则来执行提交或者回滚操作”。**

**Spring并不直接管理事务，而是提供了多种事务管理器** ，他们将事务管理的职责委托给Hibernate或者JTA等持久化机制所提供的相关平台框架的事务来实现。

 Spring事务管理器的接口是： **org.springframework.transaction.PlatformTransactionManager**，通过这个接口，Spring为各个平台如JDBC、Hibernate等都提供了对应的事务管理器，但是具体的实现就是各个平台自己的事情了。

<br>

### PlatformTransactionManager接口介绍

```java
public interface PlatformTransactionManager {
//根据指定的传播行为，返回当前活动的事务或创建一个新事务。
TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
//使用事务目前的状态提交事务
void commit(TransactionStatus status) throws TransactionException;
//对执行的事务进行回滚
void rollback(TransactionStatus status) throws TransactionException;
}
```

我们看看Spring中PlatformTransactionManager根据不同持久层框架所对应的接口实现类,几个比较常见：

```java
org.springframework.transaction.jta.JtaTransactionManager //多个数据源
org.springframework.orm.hibernate4.HibernateTransactionManager //hibernate
org.springframework.orm.jpa.JpaTransactionManager //JPA
org.springframework.jdbc.datasource.DataSourceTransactionManager //JDBC或者Ibatis持久化
```

<br>

## TransactionDefinition接口介绍

事务管理器接口 **PlatformTransactionManager** 通过 **getTransaction(TransactionDefinition definition)** 方法来得到一个事务，这个方法里面的参数是 **TransactionDefinition类** ，这个类就定义了一些基本的事务属性。

### 事务属性

事务属性可以理解成事务的一些基本配置，描述了事务策略如何应用到方法上。事务属性包含了5个方面。

###TransactionDefinition接口定义如下：

<br>

```java
public interface TransactionDefinition {
	int PROPAGATION_REQUIRED = 0;
  	int PROPAGATION_SUPPORTS = 1;
  	int PROPAGATION_MANDATORY = 2;
  	int PROPAGATION_REQUIRES_NEW = 3;
  	int PROPAGATION_NOT_SUPPORTED = 4;
  	int PROPAGATION_NEVER = 5;
  	int PROPAGATION_NESTED = 6;
  	
  	int ISOLATION_DEFAULT = -1;
  	int ISOLATION_READ_UNCOMMITTED = Connection.TRANSACTION_READ_UNCOMMITTED;
  	int ISOLATION_READ_COMMITTED = Connection.TRANSACTION_READ_COMMITTED;
  	int ISOLATION_REPEATABLE_READ = Connection.TRANSACTION_REPEATABLE_READ;
  	int ISOLATION_SERIALIZABLE = Connection.TRANSACTION_SERIALIZABLE;
  	int TIMEOUT_DEFAULT = -1;
  	
  	int getPropagationBehavior();// 返回事务的传播行为
  	// 返回事务的隔离级别，事务管理器根据它来控制另外一个事务可以看到本事务内的哪些数据
  	int getIsolationLevel();
  	int getTimeout();// 返回事务必须在多少秒内完成
  	boolean isReadOnly();// 返回是否优化为只读事务。
  	String getName();//返回事务的名字
}
```

该接口定义了一些事物传播属性、事物隔离级别

<br>

##事务隔离级别

定义了一个事务可能受其他并发事务影响的程度

我们先来看一下 **并发事务带来的问题** ，然后再来介绍一下 **TransactionDefinition 接口**中定义了五个表示隔离级别的常量。<br>

#### 并发事务带来的问题

在典型的应用程序中，多个事务并发运行，经常会操作相同的数据来完成各自的任务（多个用户对统一数据进行操作）。并发虽然是必须的，但可能会导致一下的问题。

- **脏读（Dirty read）:** 当一个事务正在访问数据并且对数据进行了修改，而这种修改还没有提交到数据库中，这时另外一个事务也访问了这个数据，然后使用了这个数据。因为这个数据是还没有提交的数据，那么另外一个事务读到的这个数据是“脏数据”，依据“脏数据”所做的操作可能是不正确的。
- **丢失修改（Lost to modify）:** 指在一个事务读取一个数据时，另外一个事务也访问了该数据，那么在第一个事务中修改了这个数据后，第二个事务也修改了这个数据。这样第一个事务内的修改结果就被丢失，因此称为丢失修改。例如：事务1读取某表中的数据A=20，事务2也读取A=20，事务1修改A=A-1，事务2也修改A=A-1，最终结果A=19，事务1的修改被丢失。
- **不可重复读（Unrepeatableread）:** 指在一个事务内多次读同一数据。在这个事务还没有结束时，另一个事务也访问该数据。那么，在第一个事务中的两次读数据之间，由于第二个事务的修改导致第一个事务两次读取的数据可能不太一样。这就发生了在一个事务内两次读到的数据是不一样的情况，因此称为不可重复读。
- **幻读（Phantom read）:** 幻读与不可重复读类似。它发生在一个事务（T1）读取了几行数据，接着另一个并发事务（T2）插入了一些数据时。在随后的查询中，第一个事务（T1）就会发现多了一些原本不存在的记录，就好像发生了幻觉一样，所以称为幻读。

<br>

### 不可重复度和幻读区别

不可重复读的重点是修改，幻读的重点在于新增或者删除。

```
例1（同样的条件, 你读取过的数据, 再次读取出来发现值不一样了 ）：事务1中的A先生读取自己的工资为 1000的操作还没完成，事务2中的B先生就修改了A的工资为2000，导 致A再读自己的工资时工资变为 2000；这就是不可重复读。

例2（同样的条件, 第1次和第2次读出来的记录数不一样 ）：假某工资单表中工资大于3000的有4人，事务1读取了所有工资大于3000的人，共查到4条记录，这时事务2 又插入了一条工资大于3000的记录，事务1再次读取时查到的记录就变为了5条，这样就导致了幻读。
```

<br>

### 隔离级别解读

- **TransactionDefinition.ISOLATION_DEFAULT:** 使用后端数据库默认的隔离级别，Mysql 默认采用的 REPEATABLEREAD隔离级别 Oracle 默认采用的 READ_COMMITTED隔离级别.
- **TransactionDefinition.ISOLATION_READ_UNCOMMITTED:** 最低的隔离级别，允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**
- **TransactionDefinition.ISOLATION_READ_COMMITTED:** 允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**
- **TransactionDefinition.ISOLATION_REPEATABLE_READ:** 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生。**
- **TransactionDefinition.ISOLATION_SERIALIZABLE:** 最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

<br><br>

## 事务传播行为

为了解决业务层方法之间互相调用的事务问题

当事务方法被另一个事务方法调用时，必须指定事务应该如何传播。例如：方法可能继续在现有事务中运行，也可能开启一个新事务，并在自己的事务中运行。

<br>

### 事物传播行为解读

**支持当前事务的情况：**

- **TransactionDefinition.PROPAGATION_REQUIRED：** 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
- **TransactionDefinition.PROPAGATION_SUPPORTS：** 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
- **TransactionDefinition.PROPAGATION_MANDATORY：** 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。

<br>

**不支持当前事务的情况：**

- **TransactionDefinition.PROPAGATION_REQUIRES_NEW：** 创建一个新的事务，如果当前存在事务，则把当前事务挂起。
- **TransactionDefinition.PROPAGATION_NOT_SUPPORTED：** 以非事务方式运行，如果当前存在事务，则把当前事务挂起。
- **TransactionDefinition.PROPAGATION_NEVER：** 以非事务方式运行，如果当前存在事务，则抛出异常。

<br>

**其他情况：**

- **TransactionDefinition.PROPAGATION_NESTED：** 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。

<br>

**注：** 前面的六种事务传播行为是 Spring 从 EJB 中引入的，他们共享相同的概念。而 **PROPAGATION_NESTED** 是 Spring 所特有的。以 PROPAGATIONNESTED 启动的事务内嵌于外部事务中（如果存在外部事务的话），此时，内嵌事务并不是一个独立的事务，它依赖于外部事务的存在，只有通过外部的事务提交，才能引起内部事务的提交，嵌套的子事务不能单独提交。如果熟悉 JDBC 中的保存点（SavePoint）的概念，那嵌套事务就很容易理解了，其实嵌套的子事务就是保存点的一个应用，一个事务中可以包括多个保存点，每一个嵌套子事务。另外，外部事务的回滚也会导致嵌套子事务的回滚。

<br>

##事务超时属性

一个事务允许执行的最长时间

所谓事务超时，就是指一个事务所允许执行的最长时间，如果超过该时间限制但事务还没有完成，则自动回滚事务。在 TransactionDefinition 中以 int 的值来表示超时时间，其单位是秒。

<br>

## 事务只读属性

对事物资源是否执行只读操作

事务的只读属性是指，对事务性资源进行只读操作或者是读写操作。所谓事务性资源就是指那些被事务管理的资源，比如数据源、 JMS 资源，以及自定义的事务性资源等等。如果确定只对事务性资源进行只读操作，那么我们可以将事务标志为只读的，以提高事务处理的性能。在 TransactionDefinition 中以 boolean 类型来表示该事务是否只读。

<br>

## 回滚规则

定义事务回滚规则

定义了哪些异常会导致事务回滚而哪些不会。默认情况下，事务只有遇到运行期异常时才会回滚，而在遇到检查型异常时不会回滚（这一行为与EJB的回滚行为是一致的）。
但是你可以声明事务在遇到特定的检查型异常时像遇到运行期异常那样回滚。同样，你还可以声明事务遇到特定的异常不回滚，即使这些异常是运行期异常。

<br>

<br>

本文章已同步到：https://github.com/Zeb-D/my-review

本文章已同步到：https://blog.csdn.net/u014229282/article/details/81012897

