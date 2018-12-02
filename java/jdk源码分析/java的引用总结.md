本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]



### 背景

前几年在看jvm虚拟机有关的书籍时，看到过java对象的各种引用，强引用、软引用、弱引用、幻象引用；

上半年也看了下ThreadLocal源码，从内存及线程等方面去研读，这方面看完后，其中有个词引起了我的关注，就是ThreadLocal中有个内部类ThreadLocalMap的实现方式，

它其实 `Entry extends WeakReference<ThreadLocal<?>>` 实现的，

- 那么WeakReference （弱引用） 这个又是什么？
- Reference 这又是什么？它的子类又有什么不一样，在jvm的表现又是怎么样的情况？



### 基础知识

jvm各个运行区域的数据也是不同，如栈大多数是基本数据类型、引用指针等；堆就是对象、数组存储在这；

那么总结下：

- java除了原始数据类型的变量，其他所有都是引用类型。
- 引用分为强引用、软引用、弱引用、幻象引用，这几种引用影响着对象的回收

#### 强引用

- 强引用：形如Object object = new Object();这样就是典型的强引用，被强引用引用的对象不会被垃圾收集器主动回收，JVM宁愿抛出OutOfMemoryError运行时错误（OOM），使程序异常终止，也不会靠随意回收具有强引用的“存活”对象来解决内存不足的问题。对于一个普通的对象，如果没有其他的引用关系，只要超过了引用的作用域或者显式地将相应强引用赋值为 null，这个被引用的对象就是可以被垃圾回收器回收的(具体回收时机还是要看垃圾收集策略)。

#### 引用（Reference类）

- 先在这里说一下，软引用（SoftReference）、弱引用（WeakReference）、幻象引用（PhantomReference）都是java.lang.ref.Reference的子类，这个Reference类主要有4个方法 
  - void clean();清除此参考对象。（此方法仅由Java代码调用; 当垃圾收集器清除引用时，它直接执行，而不调用此方法。）
  - boolean enqueue();将此引用对象添加到其注册的队列（如果有）。
  - T get();返回此引用对象的指示。(通过这个方法可以返回Reference所引用的对象，可以重新变成强引用) 例如：软引用引用的一个对象

```java
MyObject aRef = new MyObject();
SoftReference aSoftRef=new SoftReference(aRef);
aRef = null;
//现在只有一个软引用指向MyObject的这个对象，
//如果这个对象还没有被回收，可以把他再次变为强引用
if(aSoftRef.get() != null)
  MyObject bRef = aSoftRef.get();
//这个时候MyObject这个对象又变成强引用
```

- boolean isEnqueued();通过程序或垃圾收集器来告知这个引用对象是否已经入队;
- 其中enqueue 和 isEnqueued 这两个方法涉及到引用队列，我们后面会讲到。这里就先不解释，留个印象就行。

#### 软引用（SoftReference）

- 软引用通过SoftReference类实现。软引用的生命周期比强引用短一些。只有当 JVM 认为内存不足时，才会去试图回收软引用指向的对象：即JVM 会确保在抛出 OutOfMemoryError 之前，清理软引用指向的对象。软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个软引用（注意是引用本身这个对象（就是Reference自己，并不是引用所引用的对象）加入到与之关联的引用队列中。后续，我们可以调用ReferenceQueue的poll()方法来检查是否有它所关心的对象被回收（因为在这个队列里面的引用所指向的对象都被回收了）。如果队列为空，将返回一个null,否则该方法返回队列中前面的一个Reference对象。
- 应用场景：软引用通常用来实现内存敏感的缓存。如果还有空闲内存，就可以暂时保留缓存，当内存不足时清理掉，这样就保证了使用缓存的同时，不会耗尽内存。

#### 弱引用（WeakReference）

- 弱引用通过WeakReference类实现。 弱引用的生命周期比软引用短。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。由于垃圾回收器是一个优先级很低的线程，因此不一定会很快回收弱引用的对象。弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中（和引用队列一起使用同上面的软引用）。
- 应用场景：弱应用同样可用于内存敏感的缓存。

#### 幻象引用（PhantomReference）

- 幻象引用也叫虚引用，通过PhantomReference类来实现。无法通过虚引用访问对象的任何属性或函数。幻象引用仅仅是提供了一种确保对象被 finalize 以后，做某些事情的机制。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收。虚引用必须和引用队列 （ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。如果程序发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取一些程序行动（和引用队列一起使用同上面软引用跟弱引用）。
- 应用场景：可用来跟踪对象被垃圾回收器回收的活动，当一个虚引用关联的对象被垃圾收集器回收之前会收到一条系统通知

#### 引用队列（ReferenceQueue）

- Reference对象已经不再具有存在的价值，需要一个适当的清除机制，避免大量SoftReference对象带来的内存泄漏。在java.lang.ref包里还提供了ReferenceQueue。
- 前面说到在使用软引用、虚引用、幻象引用的时候可以指定一个引用队列，在引用所引用的对象被回收后引用本身就会进入引用队列。 
  - 使用例子如下

```java
ReferenceQueue queue = new ReferenceQueue();
SoftReference ref=new SoftReference(aMyObject,queue);
```

- 通过引用队列可以看到哪些Reference对象所引用的对象已经被回收，当调用引用队列的poll（）方法就可以返回除队列中的失去所引用对象的Reference对象
- 利用这个方法，我们可以检查哪个SoftReference所软引用的对象已经被回收。于是我们可以把这些失去所软引用的对象的SoftReference对象清除掉。

```java
SoftReference ref = null;
while ((ref = (EmployeeRef) q.poll()) != null) {
// 清除ref
}
```



### 总结

回归ThreadLocal的内部类分析：

1. 软引用和弱引用可以用来做一些内存敏感的缓存，空间足够的时候就缓存对象，不够的时候就回收，不会抛出oom（内存溢出异常）
2. 在ThreadLocal中使用了弱引用，大概就是ThreadLocalMap中的Entry 对key是弱引用，因为当value没被除ThreadLocal之外的其他强引用引用时，如果Entry对value是强引用那么就会造成value没办法被回收，可能导致oom，换成弱引用的话就不会出现由于Entry的引用而导致value没办法被回收。







