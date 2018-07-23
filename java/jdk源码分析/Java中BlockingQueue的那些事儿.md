#BlockingQueue的那些事儿

那些事儿，正常套路的步骤是，抛出不一样的概念（带出代码这是必须的）、常见的阻塞队列实现、具体某个队列实现原理，总之看目录就一清二楚。

参考博客：https://blog.csdn.net/vernonzheng/article/details/8247564

  		http://www.infoq.com/cn/articles/java-blocking-queue

<br>

##概念

想到阻塞队列就会联想起队列这数据结构，也会联想起LinkedList的实现方式，那么阻塞队列较于队列又有什么不同呢？字面上来说，在队列上元素进行取和压等一些操作会加入阻塞动作，好了，这翻译太勉强，让我们可以拭目以待吧！

阻塞队列（BlockingQueue）是一个支持两个附加操作的队列。这两个附加的操作是：在队列为空时，获取元素的线程会等待队列变为非空。当队列满时，存储元素的线程会等待队列可用。阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。阻塞队列就是生产者存放元素的容器，而消费者也只从容器里拿元素。

<br>

## 接口方法

| 方法\处理方式 | 抛出异常      | 返回特殊值    | 一直阻塞   | 超时退出               |
| ------- | --------- | -------- | ------ | ------------------ |
| 插入方法    | add(e)    | offer(e) | put(e) | offer(e,time,unit) |
| 移除方法    | remove()  | poll()   | take() | poll(time,unit)    |
| 检查方法    | element() | peek()   | 不可用    | 不可用                |

###处理方式说明：

- 抛出异常：是指当阻塞队列满时候，再往队列里插入元素，会抛出IllegalStateException("Queue full")异常。当队列为空时，从队列里获取元素时会抛出NoSuchElementException异常 。
- 返回特殊值：插入方法会返回是否成功，成功则返回true。移除方法，则是从队列里拿出一个元素，如果没有则返回null
- 一直阻塞：当阻塞队列满时，如果生产者线程往队列里put元素，队列会一直阻塞生产者线程，直到拿到数据，或者响应中断退出。当队列空时，消费者线程试图从队列里take元素，队列也会阻塞消费者线程，直到队列可用。
- 超时退出：当阻塞队列满时，队列会阻塞生产者线程一段时间，如果超过一定的时间，生产者线程就会退出。

###方法说明：

1. add(anObject):把anObject加到BlockingQueue里,即如果BlockingQueue可以容纳,则返回true,否则招聘异常。
2. )offer(anObject):表示如果可能的话,将anObject加到BlockingQueue里,即如果BlockingQueue可以容纳,则返回true,否则返回false。
3. put(anObject):把anObject加到BlockingQueue里,如果BlockQueue没有空间,则调用此方法的线程被阻断直到BlockingQueue里面有空间再继续.
4. poll(time):取走BlockingQueue里排在首位的对象,若不能立即取出,则可以等time参数规定的时间,取不到时返回null
5. take():取走BlockingQueue里排在首位的对象,若BlockingQueue为空,阻断进入等待状态直到Blocking有新的对象被加入为止

`其中：BlockingQueue` 不接受`null` 元素。试图`add`、`put` 或`offer` 一个`null` 元素时，某些实现会抛出`NullPointerException`。`null` 被用作指示`poll` 操作失败的警戒值。

<br>

###BlockingQueue的注意点

`【1】BlockingQueue` 可以是限定容量的。它在任意给定时间都可以有一个`remainingCapacity`，超出此容量，便无法无阻塞地`put` 附加元素。没有任何内部容量约束的`BlockingQueue` 总是报告`Integer.MAX_VALUE` 的剩余容量。

`【2】BlockingQueue` 实现主要用于生产者-使用者队列，但它另外还支持[`Collection`](http://blog.csdn.net/itm_hadf/article/details/7538083) 接口。因此，举例来说，使用`remove(x)` 从队列中移除任意一个元素是有可能的。然而，这种操作通常*不* 会有效执行，只能有计划地偶尔使用，比如在取消排队信息时。

`【3】BlockingQueue` 实现是线程安全的。所有排队方法都可以使用内部锁或其他形式的并发控制来自动达到它们的目的。然而，*大量的* Collection 操作（`addAll`、`containsAll`、`retainAll` 和`removeAll`）*没有* 必要自动执行，除非在实现中特别说明。因此，举例来说，在只添加了`c` 中的一些元素后，`addAll(c)` 有可能失败（抛出一个异常）。

`【4】BlockingQueue` 实质上*不* 支持使用任何一种“close”或“shutdown”操作来指示不再添加任何项。这种功能的需求和使用有依赖于实现的倾向。例如，一种常用的策略是：对于生产者，插入特殊的*end-of-stream* 或*poison* 对象，并根据使用者获取这些对象的时间来对它们进行解释。

<br>

## Java中阻塞队列常用实现及了解

- ArrayBlockingQueue ：一个由数组结构组成的有界阻塞队列。
- LinkedBlockingQueue ：一个由链表结构组成的有界阻塞队列。
- PriorityBlockingQueue ：一个支持优先级排序的无界阻塞队列。
- DelayQueue：一个使用优先级队列实现的无界阻塞队列。
- SynchronousQueue：一个不存储元素的阻塞队列。
- LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。
- LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。

###ArrayBlockingQueue

是一个用数组实现的有界阻塞队列。此队列按照先进先出（FIFO）的原则对元素进行排序。默认情况下不保证访问者公平的访问队列，所谓公平访问队列是指阻塞的所有生产者线程或消费者线程，当队列可用时，可以按照阻塞的先后顺序访问队列，即先阻塞的生产者线程，可以先往队列里插入元素，先阻塞的消费者线程，可以先从队列里获取元素。通常情况下为了保证公平性会降低吞吐量。我们接下来看它的构造器：

```java
public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
}
```

访问者的公平性是使用可重入锁实现的。所以能保证线程安全，我们可以继续看看它内部的一些属性：

```java
lock = new ReentrantLock(fair);     
notEmpty = lock.newCondition();//Condition Variable 1     
notFull =  lock.newCondition();//Condition Variable 2 
```

我们可以继续看看这几个Condition 的使用之处：

```java

public void put(E e) throws InterruptedException {     
        if (e == null) throw new NullPointerException();     
        final E[] items = this.items;     
        final ReentrantLock lock = this.lock;     
        lock.lockInterruptibly();//请求锁直到得到锁或者变为interrupted     
        try {     
            try {     
                while (count == items.length)//如果满了，当前线程进入noFull对应的等waiting状态     
                    notFull.await();     
            } catch (InterruptedException ie) {     
                notFull.signal(); // propagate to non-interrupted thread     
                throw ie;     
            }     
            insert(e);     
        } finally {     
            lock.unlock();     
        }     
}
```

Condition现在的实现只有java.util.concurrent.locks.AbstractQueueSynchoronizer(**抽象队列同步器**)内部的ConditionObject，并且通过ReentranLock的newCondition()方法暴露出来。

Condition的await()/sinal()一般在lock.lock()与lock.unlock()之间执行，当执行condition.await()方法时，它会首先释放掉本线程持有的锁，然后自己进入等待队列。直到sinal()，唤醒后又会重新试图去拿到锁，拿到后执行await()下的代码，其中释放当前锁和得到当前锁都需要ReentranLock的tryAcquire(int arg)方法来判定，并且享受ReentranLock的重进入特性。

<br>

###LinkedBlockingQueue

一个基于已链接节点的、范围任意的 blocking queue。此队列按 FIFO（先进先出）排序元素。队列的头部 是在队列中时间最长的元素。队列的尾部 是在队列中时间最短的元素。新元素插入到队列的尾部，并且队列检索操作会获得位于队列头部的元素。链接队列的吞吐量通常要高于基于数组的队列，但是在大多数并发应用程序中，其可预知的性能要低。

单向链表结构的队列。如果不指定容量默认为Integer.MAX_VALUE。通过putLock和takeLock两个锁进行同步，**两个锁分别实例化notFull和notEmpty两个Condtion**，用来协调多线程的存取动作。其中某些方法(如remove,toArray,toString,clear等)的同步需要同时获得这两个锁，并且总是先putLock.lock紧接着takeLock.lock(在同一方法fullyLock中)，这样的顺序是为了避免可能出现的死锁情况(我也想不明白为什么会是这样?)

接下来我们来看看它内部的同步属性：

```java
/** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
```

可以看出，读写锁分别维护了重入锁，条件也分别newCondition，这也从一定的程度上避免了死锁的发生，博主可能会深入分析LinkedBlockingQueue具体实现。

<br>

###PriorityBlockingQueue

一个无界的阻塞队列，它使用与类PriorityQueue 相同的顺序规则，并且提供了阻塞检索的操作。虽然此队列逻辑上是无界的，但是由于资源被耗尽，所以试图执行添加操作可能会失败（导致 `OutOfMemoryError`）。此类不允许使用 `null` 元素。依赖自然顺序的优先级队列也不允许插入不可比较的对象（因为这样做会抛出`ClassCastException`）。

我们可以来看看它的内部属性实现：

```java
private final PriorityQueue q;     
    private final ReentrantLock lock = new ReentrantLock(true);     
    private final Condition notEmpty = lock.newCondition();
```

可以看出该类的数据结构带有PriorityQueue，主要用来元素排序，只有PriorityQueue是怎么实现的，可以请看：http://jiadongkai-sina-com.iteye.com/blog/825683

<br>

###DelayQueue

是一个支持延时获取元素的无界阻塞队列。队列使用PriorityQueue来实现。队列中的元素必须实现Delayed接口，在创建元素时可以指定多久才能从队列中获取当前元素。只有在延迟期满时才能从队列中提取元素。我们可以将DelayQueue运用在以下应用场景：

- 缓存系统的设计：可以用DelayQueue保存缓存元素的有效期，使用一个线程循环查询DelayQueue，一旦能从DelayQueue中获取元素时，表示缓存有效期到了。
- 定时任务调度。使用DelayQueue保存当天将会执行的任务和执行时间，一旦从DelayQueue中获取到任务就开始执行，从比如TimerQueue就是使用DelayQueue实现的。

队列中的Delayed必须实现compareTo来指定元素的顺序。接下来让我们来分析下compareTo方法：

```java
public int compareTo(Delayed other) {
           if (other == this) // compare zero ONLY if same object
                return 0;
            if (other instanceof ScheduledFutureTask) {
                ScheduledFutureTask x = (ScheduledFutureTask)other;
                long diff = time - x.time;
                if (diff < 0)
                    return -1;
                else if (diff > 0)
                    return 1;
	   else if (sequenceNumber < x.sequenceNumber)
                    return -1;
                else
                    return 1;
            }
            long d = (getDelay(TimeUnit.NANOSECONDS) -
                      other.getDelay(TimeUnit.NANOSECONDS));
            return (d == 0) ? 0 : ((d < 0) ? -1 : 1);
        }
```

这方法有个作用是让延时时间最长的放在队列的末尾，在上面那段代码中可以看到ScheduledFutureTask这类，那么我们继续看看这类是如何实现**Delayed接口**的：

```java
ScheduledFutureTask(Runnable r, V result, long ns, long period) {
            super(r, result);
            this.time = ns;
            this.period = period;
            this.sequenceNumber = sequencer.getAndIncrement();
}
```

在对象创建的时候，使用time记录前对象什么时候可以使用。

```java
public long getDelay(TimeUnit unit) {
            return unit.convert(time - now(), TimeUnit.NANOSECONDS);
        }
```

然后使用getDelay可以查询当前元素还需要延时多久。

注意：通过构造函数可以看出延迟时间参数ns的单位是纳秒，自己设计的时候最好使用纳秒，因为getDelay时可以指定任意单位，一旦以纳秒作为单位，而延时的时间又精确不到纳秒就麻烦了。使用时请注意当time小于当前时间时，getDelay会返回负数。

#### 如何实现延时队列

延时队列关键点就是如何控制延时这操作，当消费者从队列里获取元素时，如果元素没有达到延时时间，就阻塞当前线程。可以看出使用信号量方式来进行await等待。

```java
long delay = first.getDelay(TimeUnit.NANOSECONDS);
                    if (delay <= 0)
                        return q.poll();
                    else if (leader != null)
                        available.await();
```

<br>

###SynchronousQueue

是一个不存储元素的阻塞队列。每一个put操作必须等待一个take操作，否则不能继续添加元素。SynchronousQueue可以看成是一个传球手，负责把生产者线程处理的数据直接传递给消费者线程。队列本身并不存储任何元素，非常适合于传递性场景,比如在一个线程中使用的数据，传递给另外一个线程使用。大家看到这，会联系到线程池中Executors.newCachedThreadPool()，其实它底层还是基于Transferer内部类。

```java
public SynchronousQueue(boolean fair) {
    transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
}
```

看到这Transferer，从字面上翻译的意思的是传输，也就是元素传递。

注：SynchronousQueue的吞吐量高于LinkedBlockingQueue 和 ArrayBlockingQueue。

<br>

###LinkedTransferQueue

是一个由链表结构组成的无界阻塞TransferQueue队列。相对于其他阻塞队列，LinkedTransferQueue多了tryTransfer和transfer方法。

transfer方法。如果当前有消费者正在等待接收元素（消费者使用take()方法或带时间限制的poll()方法时），transfer方法可以把生产者传入的元素立刻transfer（传输）给消费者。如果没有消费者在等待接收元素，transfer方法会将元素存放在队列的tail节点，并等到该元素被消费者消费了才返回。transfer方法的关键代码如下：

```java
Node pred = tryAppend(s, haveData);
return awaitMatch(s, pred, e, (how == TIMED), nanos);
```

第一行代码是试图把存放当前元素的s节点作为tail节点。第二行代码是让CPU自旋等待消费者消费元素。因为自旋会消耗CPU，所以自旋一定的次数后使用Thread.yield()方法来暂停当前正在执行的线程，并执行其他线程。

tryTransfer方法。则是用来试探下生产者传入的元素是否能直接传给消费者。如果没有消费者等待接收元素，则返回false。和transfer方法的区别是tryTransfer方法无论消费者是否接收，方法立即返回。而transfer方法是必须等到消费者消费了才返回。

对于带有时间限制的tryTransfer(E e, long timeout, TimeUnit unit)方法，则是试图把生产者传入的元素直接传给消费者，但是如果没有消费者消费该元素则等待指定的时间再返回，如果超时还没消费元素，则返回false，如果在超时时间内消费了元素，则返回true。

<br>

###LinkedBlockingDeque

是一个由链表结构组成的双向阻塞队列。所谓双向队列指的你可以从队列的两端插入和移出元素

让我们来看看这实现比LinkedBlockingQueue又有什么不一样呢？双端队列因为多了一个操作队列的入口，在多线程同时入队时，也就减少了一半的竞争。相比其他的阻塞队列，LinkedBlockingDeque多了addFirst，addLast，offerFirst，offerLast，peekFirst，peekLast等方法，以First单词结尾的方法，表示插入，获取（peek）或移除双端队列的第一个元素。以Last单词结尾的方法，表示插入，获取或移除双端队列的最后一个元素。另外插入方法add等同于addLast，移除方法remove等效于removeFirst。但是take方法却等同于takeFirst，不知道是不是Jdk的bug，使用时还是用带有First和Last后缀的方法更清楚。

除了在方法比LinkedBlockingQueue多了些实现，那主要的区别是它的同步属性不一样，它只用一个lock来维护读写操作，并由这个lock实例化出两个Condition notEmpty及notFull，而LinkedBlockingQueue读和写分别维护一个lock。

在初始化LinkedBlockingDeque时可以设置容量防止其过渡膨胀。另外双向阻塞队列可以运用在“工作窃取”模式中。

<br>

## 阻塞队列的实现原理

我们已经大致地了解阻塞队列的概念、接口、实现，那么我们继续来看看它的实现原理。

当我们进行分析一个问题或者技术时，我们经过一个入门的了解后，这时应该自问一些实现方式，带着问题去深入分析，那么：

1、如果队列是空的，消费者会一直等待，**当生产者添加元素时候，消费者是如何知道当前队列有元素的呢**？

2、如果让你来设计阻塞队列你会**如何设计，让生产者和消费者能够高效率的进行通讯呢**？

这些问题，第二问题是基于第一个问题指数的，那么第一个问题就显得是队列的通知模式机制。

所谓通知模式，就是当生产者往满的队列里添加元素时会阻塞住生产者，当消费者消费了一个队列中的元素后，会通知生产者当前队列可用。

###ArrayBlockingQueue代码实现原理分析

我们以ArrayBlockingQueue的JDK源码进行分析，在上面我们也看了ArrayBlockingQueue的一些同步属性及同步条件。

哎，还是直接贴代码：

```java
 /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;
```

```java
public ArrayBlockingQueue(int capacity, boolean fair) {
        //省略其他代码
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }

public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            insert(e);
        } finally {
            lock.unlock();
        }
}

public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            return extract();
  } finally {
            lock.unlock();
        }
}

private void insert(E x) {
        items[putIndex] = x;
        putIndex = inc(putIndex);
        ++count;
        notEmpty.signal();
    }
```

当我们往队列里插入一个元素时，如果队列不可用，阻塞生产者主要通过LockSupport.park(this)来实现，讲到这个LockSupport类，它的底层实现还是基于sun.misc.Unsafe，大家可能对这个有点感冒，那我继续说说这sun.misc.Unsafe的作用是什么？因为这类在concurrency使用或者间接使用LockSupport，主要替代传统那种同步方式，synchronized/await方式，好了，我们来大致看看Unsafe实现吧。

<br>

### sun.misc.Unsafe

JVM的实现可以自由选择如何实现Java对象的“布局”，也就是在内存里Java对象的各个部分放在哪里,包括对象的实例字段和一些元数据之类。

sun.misc.Unsafe里关于对象字段访问的方法把对象布局抽象出来，它提供了objectFieldOffset()方法用于获取某个字段相对 Java对象的“起始地址”的偏移量，也提供了getInt、getLong、getObject之类的方法可以使用前面获取的偏移量来访问某个Java 对象的某个字段。

在LockSupport有个属性记录着Thread对象parkBlockerOffset这字段的在内存偏移量。

parkBlocker就是用于记录线程被谁阻塞的，用于线程监控和分析工具来定位原因的，可以通过LockSupport的getBlocker获取到阻塞的对象。

我们可以看看LockSupport 主要方法：

```java
public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        unsafe.park(false, 0L);
        setBlocker(t, null);
    }
```

unsafe.park这个本地方法会阻塞当前线程，只有以下四种情况中的一种发生时，该方法才会返回。

- 与park对应的unpark执行或已经执行时。注意：已经执行是指unpark先执行，然后再执行的park。
- 线程被中断时。
- 如果参数中的time不是零，等待了指定的毫秒数时。
- 发生异常现象时。这些异常事先无法确定。

本来想继续深入分析unsafe.park实现，但是JVM是如何实现park方法的，park在不同的操作系统使用不同的方式实现。

我是不会讲openjdk源码实现的，有兴趣的小伙伴们可以在src/os/linux/vm/os_linux.cpp 文件中看到os::PlatformEvent::park方法实现。

我还是贴下这方法：

```c
void os::PlatformEvent::park() {      
     	     int v ;
	     for (;;) {
		v = _Event ;
	     if (Atomic::cmpxchg (v-1, &_Event, v) == v) break ;
	     }
	     guarantee (v >= 0, "invariant") ;
	     if (v == 0) {
	     // Do this the hard way by blocking ...
	     int status = pthread_mutex_lock(_mutex);
	     assert_status(status == 0, status, "mutex_lock");
	     guarantee (_nParked == 0, "invariant") ;
	     ++ _nParked ;
	     while (_Event < 0) {
	     status = pthread_cond_wait(_cond, _mutex);
	     // for some reason, under 2.7 lwp_cond_wait() may return ETIME ...
	     // Treat this the same as if the wait was interrupted
	     if (status == ETIME) { status = EINTR; }
	     assert_status(status == 0 || status == EINTR, status, "cond_wait");
	     }
	     -- _nParked ;
	     
	     // In theory we could move the ST of 0 into _Event past the unlock(),
	     // but then we'd need a MEMBAR after the ST.
	     _Event = 0 ;
	     status = pthread_mutex_unlock(_mutex);
	     assert_status(status == 0, status, "mutex_unlock");
	     }
	     guarantee (_Event >= 0, "invariant") ;
	     }

     }
```

看了半天这还是使用了系统方法pthread_cond_wait实现的，醉了，继续网上找资料看看这鬼方法是什么情况？

pthread_cond_wait是一个多线程的条件变量函数，cond是condition的缩写，字面意思可以理解为线程在等待一个条件发生，这个条件是一个全局变量。这个方法接收两个参数，一个共享变量_cond，一个互斥量_mutex。而unpark方法在linux下是使用pthread_cond_signal实现的。park 在windows下则是使用WaitForSingleObject实现的。

原谅我对计算机系统不够深入。。。

我现在只能对Unsafe和LockSupport讲到这，实在文章篇幅有限，大家可以继续关注我后续动作讲解。

<br>

### 大家回归到ArrayBlockingQueue

老规矩，继续贴剩下的代码：

```java
public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
				reportInterruptAfterWait(interruptMode);
        }
```

继续进入源码，发现调用setBlocker先保存下将要阻塞的线程，然后调用unsafe.park阻塞当前线程。

<br>

###总结put实现

在每次进入put都会先获取锁，最后finally 释放锁，这是ReentrantLock或者Lock使用套路，

然后判断队列中的元素是否满了count == items.length ，满了就使用条件condition.await方法阻塞，否则插入。

在插入后就通知该线程插入成功完毕，还是condition.signal()方法，哎，这不是和上面介绍ArrayBlockingQueue时讲到Condition的使用套路 Condition的await()/sinal()一般在lock.lock()与lock.unlock()之间执行，你看，这还真的是，因为这种使用方式就Lock实现类普遍套路。

怎么阻塞线程，就使用LockSupport /unsafe.park。

好了，其他操作方式大致也是一样的，底层关键技术还是LockSupport类。

告诉大家一个好消息，我会打算抽出个时间来总结下java.util.concurrent.locks.AbstractQueueSynchoronizer 的实现。

<br>

本文章来源：https://github.com/Zeb-D/my-review



