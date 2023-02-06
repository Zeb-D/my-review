本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 0 引言

`context`是Go中广泛使用的程序包，由Google官方开发，在1.7版本引入。它用来简化在多个go routine传递上下文数据、(手动/超时)中止routine树等操作，

比如，官方http包使用context传递请求的上下文数据，gRpc使用context来终止某个请求产生的routine树(可参考[grpc-go超时传递原理.md](./grpc-go超时传递原理.md))。

由于它使用简单，现在基本成了编写go基础库的通用规范。笔者在使用context上有一些经验，遂分享下。

本文主要谈谈以下几个方面的内容：

1. context的使用。
2. context实现原理，哪些是需要注意的地方。
3. 在实践中遇到的问题，分析问题产生的原因。



### 1 使用

#### 1.1 核心接口Context

```go
type Context interface {
    // Deadline returns the time when work done on behalf of this context
    // should be canceled. Deadline returns ok==false when no deadline is
    // set.
    Deadline() (deadline time.Time, ok bool)
    // Done returns a channel that's closed when work done on behalf of this
    // context should be canceled.
    Done() <-chan struct{}
    // Err returns a non-nil error value after Done is closed.
    Err() error
    // Value returns the value associated with this context for key.
    Value(key interface{}) interface{}
}
```

简单介绍一下其中的方法：
- Done会返回一个channel，当该context被取消的时候，该channel会被关闭，同时对应的使用该context的routine也应该结束并返回。
- Context中的方法是协程安全的，这也就代表了在父routine中创建的context，可以传递给任意数量的routine并让他们同时访问。
- Deadline会返回一个超时时间，routine获得了超时时间后，可以对某些io操作设定超时时间。
- Value可以让routine共享一些数据，当然获得数据是协程安全的。

在请求处理的过程中，会调用各层的函数，每层的函数会创建自己的routine，是一个routine树。所以，context也应该反映并实现成一棵树。



要创建context树，第一步是要有一个根结点。`context.Background`函数的返回值是一个空的context，经常作为树的根结点，它一般由接收请求的第一个routine创建，不能被取消、没有值、也没有过期时间。

```go
func Background() Context
```

之后该怎么创建其它的子孙节点呢？context包为我们提供了以下函数：

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key interface{}, val interface{}) Context
```



这四个函数的第一个参数都是父context，返回一个Context类型的值，这样就层层创建出不同的节点。子节点是从复制父节点得到的，并且根据接收的函数参数保存子节点的一些状态值，然后就可以将它传递给下层的routine了。

`WithCancel`函数，返回一个额外的CancelFunc函数类型变量，该函数类型的定义为：

```go
type CancelFunc func()
```

调用CancelFunc对象将撤销对应的Context对象，这样父结点的所在的环境中，获得了撤销子节点context的权利，当触发某些条件时，可以调用CancelFunc对象来终止子结点树的所有routine。在子节点的routine中，需要用类似下面的代码来判断何时退出routine：

```go
select {
    case <-cxt.Done():
        // do some cleaning and return
}
```

根据cxt.Done()判断是否结束。当顶层的Request请求处理结束，或者外部取消了这次请求，就可以cancel掉顶层context，从而使整个请求的routine树得以退出。

`WithDeadline`和`WithTimeout`比`WithCancel`多了一个时间参数，它指示context存活的最长时间。如果超过了过期时间，会自动撤销它的子context。所以context的生命期是由父context的routine和`deadline`共同决定的。

`WithValue`返回parent的一个副本，该副本保存了传入的key/value，而调用Context接口的Value(key)方法就可以得到val。注意在同一个context中设置key/value，若key相同，值会被覆盖。

关于更多的使用示例，可参考[官方博客](https://blog.golang.org/context)。



### 2 原理

#### 2.1 上下文数据的存储与查询

```go
type valueCtx struct {
    Context
    key, val interface{}
}

func WithValue(parent Context, key, val interface{}) Context {
    if key == nil {
        panic("nil key")
    }
    ......
    return &valueCtx{parent, key, val}
}

func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)
}
```

context上下文数据的存储就像一个树，每个结点只存储一个key/value对。`WithValue()`保存一个key/value对，它将父context嵌入到新的子context，并在节点中保存了key/value数据。`Value()`查询key对应的value数据，会从当前context中查询，如果查不到，会递归查询父context中的数据。

值得注意的是，**context中的上下文数据并不是全局的，它只查询本节点及父节点们的数据，不能查询兄弟节点的数据。**



#### 2.2 手动cancel和超时cancel

`cancelCtx`中嵌入了父Context，实现了canceler接口：

```go
type cancelCtx struct {
    Context      // 保存parent Context
    done chan struct{}
    mu       sync.Mutex
    children map[canceler]struct{}
    err      error
}

// A canceler is a context type that can be canceled directly. The
// implementations are *cancelCtx and *timerCtx.
type canceler interface {
    cancel(removeFromParent bool, err error)
    Done() <-chan struct{}
}
```

`cancelCtx`结构体中`children`保存它的所有`子canceler`， 当外部触发cancel时，会调用`children`中的所有`cancel()`来终止所有的`cancelCtx`。`done`用来标识是否已被cancel。当外部触发cancel、或者父Context的channel关闭时，此done也会关闭。

```go
type timerCtx struct {
    cancelCtx     //cancelCtx.Done()关闭的时机：1）用户调用cancel 2）deadline到了 3）父Context的done关闭了
    timer    *time.Timer
    deadline time.Time
}

func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc) {
    ......
    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  deadline,
    }
    propagateCancel(parent, c)
    d := time.Until(deadline)
    if d <= 0 {
        c.cancel(true, DeadlineExceeded) // deadline has already passed
        return c, func() { c.cancel(true, Canceled) }
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.err == nil {
        c.timer = time.AfterFunc(d, func() {
            c.cancel(true, DeadlineExceeded)
        })
    }
    return c, func() { c.cancel(true, Canceled) }
}
```

`timerCtx`结构体中`deadline`保存了超时的时间，当超过这个时间，会触发`cancel`。



可以看出，**cancelCtx也是一棵树，当触发cancel时，会cancel本结点和其子树的所有cancelCtx**。



### 3 遇到的问题

#### 3.1 背景

某天，为了给我们的系统接入内部的链路跟踪系统，需要在gRpc/Mysql/Redis/MQ操作过程中传递requestId、rpcId，我们的解决方案是`Context`。

所有Mysql、MQ、Redis的操作接口的第一个参数都是context，如果这个context(或其父context)被cancel了，则操作会失败。

```go
func (tx *Tx) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error)
func(process func(context.Context, redis.Cmder) error) func(context.Context, redis.Cmder) error
func (ch *Channel) Consume(ctx context.Context, handler Handler, queue string, dc <-chan amqp.Delivery) error
func (ch *Channel) Publish(ctx context.Context, exchange, key string, mandatory, immediate bool, msg Publishing) (err error)
```

上线后，遇到一系列的坑......



#### 3.2 Case 1

**现象：**上线后，5分钟后所有用户登录失败，不断收到报警。

**原因：**程序中使用localCache，会每5分钟Refresh(调用注册的回调函数)一次所缓存的变量。localCache中保存了一个context，在调用回调函数时会传进去。如果回调函数依赖context，可能会产生意外的结果。

程序中，回调函数`getAppIDAndAlias`的功能是从mysql中读取相关数据。如果ctx被cancel了，会直接返回失败。

```text
func getAppIDAndAlias(ctx context.Context, appKey, appSecret string) (string, string, error)
```

第一次localCache.Get(ctx, appKey, appSeret)传的ctx是gRpc call传进来的context，而gRpc在请求结束或失败时会cancel掉context，导致之后cache Refresh()时，执行失败。

**解决方法：**在Refresh时不使用localCache的context，使用一个不会cancel的context。



#### 3.3 Case 2

**现象：**上线后，不断收到报警(sys err过多)。看log/etrace产生2种sys err：

- context canceled
- sql: Transaction has already been committed or rolled back

##### 3.3.1 背景及原因

![img](https://pic2.zhimg.com/80/v2-7104badb8672c8e53950b992f2f70449_1440w.webp)

`Ticket`是处理Http请求的服务，它使用Restful风格的协议。由于程序内部使用的是gRpc协议，需要某个组件进行协议转换，我们引入了[grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)，用它来实现Restful转成gRpc的互转。

**复现context canceled的流程如下：**

1. 客户端发送http restful请求。
2. grpc-gateway与客户端建立连接，接收请求，转换参数，调用后面的grpc-server。
3. grpc-server处理请求。其中，grpc-server会对每个请求启一个stream，由这个stream创建context。
4. 客户端连接断开。
5. grpc-gateway收到连接断开的信号，导致context cancel。grpc client在发送rpc请求后由于外部异常使它的请求终止了(即它的context被cancel)，会发一个RST_STREAM。
6. grpc server收到后，马上终止请求（即grpc server的stream context被cancel）。

可以看出，是因为gRpc handler在处理过程中连接被断开。



**sql: Transaction has already been committed or rolled back产生的原因：**

程序中使用了官方database包来执行db transaction。其中，在db.BeginTx时，会启一个协程awaitDone：

```text
func (tx *Tx) awaitDone() {
    // Wait for either the transaction to be committed or rolled
    // back, or for the associated context to be closed.
    <-tx.ctx.Done()

    // Discard and close the connection used to ensure the
    // transaction is closed and the resources are released.  This
    // rollback does nothing if the transaction has already been
    // committed or rolled back.
    tx.rollback(true)
}
```

在context被cancel时，会进行rollback()，而rollback时，会操作原子变量。之后，在另一个协程中tx.Commit()时，会判断原子变量，如果变了，会抛出错误。

##### 3.3.2 解决方法

这两个error都是由连接断开导致的，是正常的。可忽略这两个error。



#### 3.4 Case 3

上线后，每两天左右有1~2次的mysql事务阻塞，导致请求耗时达到120秒。在内部的mysql运维平台中查询到所有阻塞的事务在处理同一条记录。



##### 3.4.1 处理过程

1. 初步怀疑是跨机房的多个事务操作同一条记录导致的。由于跨机房操作，耗时会增加，导致阻塞了其他机房执行的db事务。

2. 出现此现象时，暂时将某个接口降级。降低多个事务操作同一记录的概率。

3. 减少事务的个数。

- 将单条sql的事务去掉
- 通过业务逻辑的转移减少不必要的事务

4. 调整db参数`innodb_lock_wait_timeout`(120s->50s)。这个参数指示mysql在执行事务时阻塞的最大时间，将这个时间减少，来减少整个操作的耗时。考虑过在程序中指定事务的超时时间，但是`innodb_lock_wait_timeout`要么是全局，要么是session的。担心影响到session上的其它sql，所以没设置。

5. 考虑使用分布式锁来减少操作同一条记录的事务的并发量。但由于时间关系，没做这块的改进。

6. DAL同事发现有事务没提交，查看代码，找到root cause。



原因是golang官方包database/sql会在某种竞态条件下，导致事务既没有commit，也没有rollback。

##### 3.4.2 源码描述

开始事务[BeginTxx()](https://github.com/golang/go/blob/master/src/database/sql/sql.go#L1595)时会启一个协程：

```text
// awaitDone blocks until the context in Tx is canceled and rolls back
// the transaction if it's not already done.
func (tx *Tx) awaitDone() {
    // Wait for either the transaction to be committed or rolled
    // back, or for the associated context to be closed.
    <-tx.ctx.Done()

    // Discard and close the connection used to ensure the
    // transaction is closed and the resources are released.  This
    // rollback does nothing if the transaction has already been
    // committed or rolled back.
    tx.rollback(true)
}
```

`tx.rollback(true)`中，会先判断原子变量tx.done是否为1，如果1，则返回；如果是0，则加1，并进行rollback操作。

在提交事务Commit()时，会先操作原子变量tx.done，然后判断context是否被cancel了，如果被cancel，则返回；如果没有，则进行commit操作。

```text
// Commit commits the transaction.
func (tx *Tx) Commit() error {
    if !atomic.CompareAndSwapInt32(&tx.done, 0, 1) {
        return ErrTxDone
    }

    select {
    default:
    case <-tx.ctx.Done():
        return tx.ctx.Err()
    }
    var err error
    withLock(tx.dc, func() {
        err = tx.txi.Commit()
    })
    if err != driver.ErrBadConn {
        tx.closePrepared()
    }
    tx.close(err)
    return err
}
```

如果先进行commit()过程中，先操作原子变量，然后context被cancel，之后另一个协程在进行rollback()会因为原子变量置为1而返回。导致commit()没有执行，rollback()也没有执行。

##### 3.4.3 解决方法

解决方法可以是如下任一个：

- 在执行事务时传进去一个`不会cancel的context`
- 修正`database/sql`源码，然后在编译时指定新的go编译镜像

我们之后给Golang提交了[patch](https://link.zhihu.com/?target=https%3A//github.com/golang/go/commit/bcf964de5e16486cec2e102c929768778f50eea2)，修正了此问题(已合入go 1.9.3)。



### 4 经验教训

由于go大量的官方库、第三方库使用了context，所以调用`接收context的函数`时要小心，要清楚context在什么时候cancel，什么行为会触发cancel。笔者在程序经常使用gRpc传出来的context，产生了一些非预期的结果，之后花时间总结了gRpc、内部基础库中context的生命期及行为，以避免出现同样的问题。



**参考资料**

- https://zhuanlan.zhihu.com/p/34417106