本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

------

[TOC]

------

本文将介绍常用的线程间通信工具CountDownLatch、CyclicBarrier和Phaser的用法，并结合实例介绍它们各自的适用场景及相同点和不同点。

------

## CountDownLatch

### CountDownLatch适用场景

Java多线程编程中经常会碰到这样一种场景——某个线程需要等待一个或多个线程操作结束（或达到某种状态）才开始执行。比如开发一个并发测试工具时，主线程需要等到所有测试线程均执行完成再开始统计总共耗费的时间，此时可以通过CountDownLatch轻松实现。

### CountDownLatch实例

```java
package com.test.thread;

import java.util.Date;
import java.util.concurrent.CountDownLatch;

public class CountDownLatchDemo {
  public static void main(String[] args) throws InterruptedException {
    int totalThread = 3;
    long start = System.currentTimeMillis();
    CountDownLatch countDown = new CountDownLatch(totalThread);
    for(int i = 0; i < totalThread; i++) {
      final String threadName = "Thread " + i;
      new Thread(() -> {
        System.out.println(String.format("%s\t%s %s", new Date(), threadName, "started"));
        try {
          Thread.sleep(1000);
        } catch (Exception ex) {
          ex.printStackTrace();
        }
        countDown.countDown();
        System.out.println(String.format("%s\t%s %s", new Date(), threadName, "ended"));
      }).start();;
    }
    countDown.await();
    long stop = System.currentTimeMillis();
    System.out.println(String.format("Total time : %sms", (stop - start)));
  }
}
```

执行结果

```java
Sun Jun 19 20:34:31 CST 2016  Thread 1 started
Sun Jun 19 20:34:31 CST 2016  Thread 0 started
Sun Jun 19 20:34:31 CST 2016  Thread 2 started
Sun Jun 19 20:34:32 CST 2016  Thread 2 ended
Sun Jun 19 20:34:32 CST 2016  Thread 1 ended
Sun Jun 19 20:34:32 CST 2016  Thread 0 ended
Total time : 1072ms
```



可以看到，主线程等待所有3个线程都执行结束后才开始执行。

### CountDownLatch主要接口分析

CountDownLatch工作原理相对简单，可以简单看成一个倒计数器，在构造方法中指定初始值，每次调用*countDown()*方法时将计数器减1，而*await()*会等待计数器变为0。CountDownLatch关键接口如下

- **countDown()** 如果当前计数器的值大于1，则将其减1；若当前值为1，则将其置为0并唤醒所有通过await等待的线程；若当前值为0，则什么也不做直接返回。

- **await()** 等待计数器的值为0，若计数器的值为0则该方法返回；若等待期间该线程被中断，则抛出**InterruptedException**并清除该线程的中断状态。

- **await(long timeout, TimeUnit unit)** 在指定的时间内等待计数器的值为0，若在指定时间内计数器的值变为0，则该方法返回true；若指定时间内计数器的值仍未变为0，则返回false；若指定时间内计数器的值变为0之前当前线程被中断，则抛出**InterruptedException**并清除该线程的中断状态。

- **getCount()** 读取当前计数器的值，一般用于调试或者测试。


## CyclicBarrier

### CyclicBarrier适用场景

它能保证屏障之前的代码一定在屏障之后的代码之前被执行。CyclicBarrier可以译为循环屏障，也有类似的功能。CyclicBarrier可以在构造时指定需要在屏障前执行await的个数，所有对await的调用都会等待，直到调用await的次数达到预定指，所有等待都会立即被唤醒。

从使用场景上来说，CyclicBarrier是让多个线程互相等待某一事件的发生，然后同时被唤醒。而上文讲的CountDownLatch是让某一线程等待多个线程的状态，然后该线程被唤醒。

### CyclicBarrier实例

```java
package com.test.thread;

import java.util.Date;
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierDemo {

  public static void main(String[] args) {
    int totalThread = 5;
    CyclicBarrier barrier = new CyclicBarrier(totalThread);
    
    for(int i = 0; i < totalThread; i++) {
      String threadName = "Thread " + i;
      new Thread(() -> {
        System.out.println(String.format("%s\t%s %s", new Date(), threadName, " is waiting"));
        try {
          barrier.await();
        } catch (Exception ex) {
          ex.printStackTrace();
        }
        System.out.println(String.format("%s\t%s %s", new Date(), threadName, "ended"));
      }).start();
    }
  }
}
```

执行结果如下

```java
Sun Jun 19 21:04:49 CST 2016  Thread 1  is waiting
Sun Jun 19 21:04:49 CST 2016  Thread 0  is waiting
Sun Jun 19 21:04:49 CST 2016  Thread 3  is waiting
Sun Jun 19 21:04:49 CST 2016  Thread 2  is waiting
Sun Jun 19 21:04:49 CST 2016  Thread 4  is waiting
Sun Jun 19 21:04:49 CST 2016  Thread 4 ended
Sun Jun 19 21:04:49 CST 2016  Thread 0 ended
Sun Jun 19 21:04:49 CST 2016  Thread 2 ended
Sun Jun 19 21:04:49 CST 2016  Thread 1 ended
Sun Jun 19 21:04:49 CST 2016  Thread 3 ended
```



从执行结果可以看到，每个线程都不会在其它所有线程执行*await()*方法前继续执行，而等所有线程都执行*await()*方法后所有线程的等待都被唤醒从而继续执行。

### CyclicBarrier主要接口分析

CyclicBarrier提供的关键方法如下

- **await()** 等待其它参与方的到来（调用await()）。如果当前调用是最后一个调用，则唤醒所有其它的线程的等待并且如果在构造CyclicBarrier时指定了action，当前线程会去执行该action，然后该方法返回该线程调用await的次序（getParties()-1说明该线程是第一个调用await的，0说明该线程是最后一个执行await的），接着该线程继续执行await后的代码；如果该调用不是最后一个调用，则阻塞等待；如果等待过程中，当前线程被中断，则抛出**InterruptedException**；如果等待过程中，其它等待的线程被中断，或者其它线程等待超时，或者该barrier被reset，或者当前线程在执行barrier构造时注册的action时因为抛出异常而失败，则抛出**BrokenBarrierException**。
- **await(long timeout, TimeUnit unit)** 与*await()*唯一的不同点在于设置了等待超时时间，等待超时时会抛出**TimeoutException**。
- **reset()** 该方法会将该barrier重置为它的初始状态，并使得所有对该barrier的await调用抛出**BrokenBarrierException**。



## Phaser

### Phaser适用场景

CountDownLatch和CyclicBarrier都是JDK 1.5引入的，而Phaser是JDK 1.7引入的。Phaser的功能与CountDownLatch和CyclicBarrier有部分重叠，同时也提供了更丰富的语义和更灵活的用法。

Phaser顾名思义，与阶段相关。

Phaser比较适合这样一种场景，一种任务可以分为多个阶段，现希望多个线程去处理该批任务，对于每个阶段，多个线程可以并发进行，但是希望保证只有前面一个阶段的任务完成之后才能开始后面的任务。

这种场景可以使用多个CyclicBarrier来实现，每个CyclicBarrier负责等待一个阶段的任务全部完成。但是使用CyclicBarrier的缺点在于，需要明确知道总共有多少个阶段，同时并行的任务数需要提前预定义好，且无法动态修改。而Phaser可同时解决这两个问题。

### Phaser实例

```java
public class PhaserDemo {

  public static void main(String[] args) throws IOException {
    int parties = 3;
    int phases = 4;
    final Phaser phaser = new Phaser(parties) {
      @Override  
      protected boolean onAdvance(int phase, int registeredParties) {  
          System.out.println("====== Phase : " + phase + " ======");  
          return registeredParties == 0;  
      }  
    };
    
    for(int i = 0; i < parties; i++) {
      int threadId = i;
      Thread thread = new Thread(() -> {
        for(int phase = 0; phase < phases; phase++) {
          System.out.println(String.format("Thread %s, phase %s", threadId, phase));
          phaser.arriveAndAwaitAdvance();
        }
      });
      thread.start();
    }
  }
}
```

执行结果如下

```java
Thread 0, phase 0
Thread 1, phase 0
Thread 2, phase 0
====== Phase : 0 ======
Thread 2, phase 1
Thread 0, phase 1
Thread 1, phase 1
====== Phase : 1 ======
Thread 1, phase 2
Thread 2, phase 2
Thread 0, phase 2
====== Phase : 2 ======
Thread 0, phase 3
Thread 1, phase 3
Thread 2, phase 3
====== Phase : 3 ======
```



从上面的结果可以看到，多个线程必须等到其它线程的同一阶段的任务全部完成才能进行到下一个阶段，并且每当完成某一阶段任务时，Phaser都会执行其*onAdvance*方法。

### Phaser主要接口分析

Phaser主要接口如下

- **arriveAndAwaitAdvance()** 当前线程当前阶段执行完毕，等待其它线程完成当前阶段。如果当前线程是该阶段最后一个未到达的，则该方法直接返回下一个阶段的序号（阶段序号从0开始），同时其它线程的该方法也返回下一个阶段的序号。
- **arriveAndDeregister()** 该方法立即返回下一阶段的序号，并且其它线程需要等待的个数减一，并且把当前线程从之后需要等待的成员中移除。如果该Phaser是另外一个Phaser的子Phaser（层次化Phaser会在后文中讲到），并且该操作导致当前Phaser的成员数为0，则该操作也会将当前Phaser从其父Phaser中移除。
- **arrive()** 该方法不作任何等待，直接返回下一阶段的序号。
- **awaitAdvance(int phase)** 该方法等待某一阶段执行完毕。如果当前阶段不等于指定的阶段或者该Phaser已经被终止，则立即返回。该阶段数一般由*arrive()*方法或者*arriveAndDeregister()*方法返回。返回下一阶段的序号，或者返回参数指定的值（如果该参数为负数），或者直接返回当前阶段序号（如果当前Phaser已经被终止）。
- **awaitAdvanceInterruptibly(int phase)** 效果与*awaitAdvance(int phase)*相当，唯一的不同在于若该线程在该方法等待时被中断，则该方法抛出**InterruptedException**。
- **awaitAdvanceInterruptibly(int phase, long timeout, TimeUnit unit)** 效果与*awaitAdvanceInterruptibly(int phase)*相当，区别在于如果超时则抛出**TimeoutException**。
- **bulkRegister(int parties)** 注册多个party。如果当前phaser已经被终止，则该方法无效，并返回负数。如果调用该方法时，*onAdvance*方法正在执行，则该方法等待其执行完毕。如果该Phaser有父Phaser则指定的party数大于0，且之前该Phaser的party数为0，那么该Phaser会被注册到其父Phaser中。
- **forceTermination()** 强制让该Phaser进入终止状态。已经注册的party数不受影响。如果该Phaser有子Phaser，则其所有的子Phaser均进入终止状态。如果该Phaser已经处于终止状态，该方法调用不造成任何影响。



------

**注：**本文查看来源于http://www.jasongj.com/java/thread_communication/