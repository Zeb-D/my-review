本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

无论是线程还是进程进行用户读写（并不一定是数据库方面的）操作，都有可能发生用户态与内核进行数据交互，那么为什么会有这个交互的过程，其实在设计OS时是为了辅助CPU高速运转；

本文结合需要计算[Java对象内存布局及大小](../java/jvm/Java对象内存布局.md)，不要问为什么是JAVA，说白了，带你走java底层世界是如何在CPU呈现的；

将尝试理清java 与cpu cache line 各个关系，所以你需要一定的知识来保证你能尽量看懂。



### CPU Cache介绍

随着CPU频率的不断提升，内存的访问速度却并没有什么突破。所以，为了弥补内存访问速度慢的硬伤，便出现了CPU缓存。它的工作原理如下：

- 当CPU要读取一个数据时，首先从缓存中查找，如果找到就立即读取并送给CPU处理；
- 如果没有找到，就用相对慢的速度从内存中读取并送给CPU处理，同时把这个数据所在的数据块调入缓存中，可以使得以后对整块数据的读取都从缓存中进行，不必再调用内存。

为了充分发挥CPU的计算性能和吞吐量，现代CPU引入了一级缓存、二级缓存和三级缓存，结构如下图所示：

![img](http://upload-images.jianshu.io/upload_images/5401975-8e9eba8ad5656e7f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/739/format/webp)

图中所示的是三级缓存的架构，可以看到，级别越小的缓存，越接近CPU，但访问速度也会越**快**。

- L1 Cache分为D-Cache和I-Cache，D-Cache用来存储数据，I-Cache用来存放指令，一般L1 Cache的大小是32k；
- L2 Cache 更大一些,例如256K, 速度要慢一些, 一般情况下每个核上都有一个独立的L2 Cache；
- L3 Cache是三级缓存中最大的一级，例如12MB，同时也是最慢的一级，在同一个CPU插槽之间的核共享一个L3 Cache。

当CPU计算时，首先去L1去寻找需要的数据，如果没有则去L2寻找，接着从L3中寻找，如果都没有，则从内存中读取数据。所以，如果某些数据需要经常被访问，那么这些数据存放在L1中的效率会最高。

#### CPU到各缓存和内存之间的大概速度：

| 从CPU到    | 大约需要的CPU周期      | 大约需要的时间(单位ns) |
| -------- | --------------- | ------------- |
| 寄存器      | 1 cycle         |               |
| L1 Cache | ~3-4 cycles     | ~0.5-1 ns     |
| L2 Cache | ~10-20 cycles   | ~3-7 ns       |
| L3 Cache | ~40-45 cycles   | ~15 ns        |
| 跨槽传输     | ~20 ns          |               |
| 内存       | ~120-240 cycles | ~60-120ns     |

在Linux中可以通过如下命令查看CPU Cache：

```
cat /sys/devices/system/cpu/cpu0/cache/index0/size
32K
cat /sys/devices/system/cpu/cpu0/cache/index1/size
32K
cat /sys/devices/system/cpu/cpu0/cache/index2/size
256K
cat /sys/devices/system/cpu/cpu0/cache/index3/size
20480K
cat /sys/devices/system/cpu/cpu0/cache/index0/type
Data
cat /sys/devices/system/cpu/cpu0/cache/index1/type
Instruction

```

这里的index0和index1对应着L1 D-Cache和L1 I-Cache。

### 缓存行Cache Line

#### 什么是伪共享

缓存系统中是以缓存行（cache line）为单位存储的。缓存行是2的整数幂个连续字节，一般为32-256个字节。最常见的缓存行大小是64个字节。当多线程修改互相独立的变量时，如果这些变量共享同一个缓存行，就会无意中影响彼此的性能，这就是伪共享。

缓存行上的写竞争是运行在SMP系统中并行线程实现可伸缩性最重要的限制因素。有人将伪共享描述成无声的性能杀手，因为从代码中很难看清楚是否会出现伪共享。

#### 为什么会伪共享

![img](https://upload-images.jianshu.io/upload_images/5401975-5121c0c34e2ce9bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/557/format/webp)



​	假设在核心1上运行的线程想更新变量X，同时核心2上的线程想要更新变量Y。不幸的是，这两个变量在同一个缓存行中。每个线程都要去竞争缓存行的所有权来更新变量。如果核心1获得了所有权，缓存子系统将会使核心2中对应的缓存行失效。当核心2获得了所有权然后执行更新操作，核心1就要使自己对应的缓存行失效。这会来来回回的经过L3缓存，大大影响了性能。如果互相竞争的核心位于不同的插槽，就要额外横跨插槽连接，问题可能更加严重。

#### 如何设置缓存行大小

缓存是由缓存行组成的。一般一行缓存行有64字节。CPU在操作缓存时是以缓存行为单位的，可以通过如下命令查看缓存行的大小：

```
cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size
64

```

由于CPU存取缓存都是按行为最小单位操作的。对于long类型来说，一个long类型的数据有64位，也就是8个字节，所以对于数组来说，由于数组中元素的地址是连续的，所以在加载数组中第一个元素的时候会把后面的元素也加载到缓存行中。

​	如果一个long类型的数组长度是8，那么也就是64个字节了，CPU这时操作该数组，会把数组中所有的元素都放入缓存行吗？答案是否定的，原因就是在Java中，对象在内存中的结构包含对象头，可以参考[Java对象内存布局](../java/jvm/Java对象内存布局.md)来了解。

<br>

#### 避免伪共享

假设有一个类中，只有一个long类型的变量：

```
public final static class VolatileLong {
    public volatile long value = 0L;
}

```

这时定义一个VolatileLong类型的数组，然后让多个线程同时并发访问这个数组，这时可以想到，在多个线程同时处理数据时，数组中的多个VolatileLong对象可能存在同一个缓存行中，通过上文可知，这种情况就是伪共享。

怎么样避免呢？

##### 在Java 7之前

可以在属性的前后进行padding，例如：

```
public final static class VolatileLong {
    volatile long p0, p1, p2, p3, p4, p5, p6;
    public volatile long value = 0;
    volatile long q0, q1, q2, q3, q4, q5, q6;
}

```

通过[Java对象内存布局](../java/jvm/JVM之内存布局与对象大小.md)文章中对paddign的分析可知，由于都是long类型的变量，这里就是按照声明的顺序分配内存，那么这可以保证在同一个缓存行中只有一个VolatileLong对象。

这里有一个问题：

据说Java7优化了无用字段，会使这种形式的补位无效，但经过测试，无论是在JDK 1.7 还是 JDK 1.8中，这种形式都是有效的。网上有关伪共享的文章基本都是来自Martin的两篇博客，这种优化方式也是在他的博客中提到的。但国内的文章貌似根本就没有验证过而直接引用了此观点，这也确实迷惑了一大批同学

##### 在Java 8中

提供了@sun.misc.Contended注解来避免伪共享，**[原理]**是在使用此注解的对象或字段的前后各增加128字节大小的padding，使用2倍于大多数硬件缓存行的大小来避免相邻扇区预取导致的伪共享冲突。具体可以参考[http://mail.openjdk.java.net/pipermail/hotspot-dev/2012-November/007309.html](https://link.jianshu.com?t=http://mail.openjdk.java.net/pipermail/hotspot-dev/2012-November/007309.html)。

在Java8中提供了@sun.misc.Contended来避免伪共享，在运行时需要设置JVM启动参数`-XX:-RestrictContended`

##### @sun.misc.Contended注解

将@sun.misc.Contended注解用在了对象上，@sun.misc.Contended注解还可以指定某个字段，并且可以为字段进行分组，

下面通过代码来看下：

```java
/**
 * VM Options: 
 * -javaagent:/Users/sangjian/dev/source-files/classmexer-0_03/classmexer.jar
 * -XX:-RestrictContended
 */
public class ContendedTest {

    byte a;
    @sun.misc.Contended("a")
    long b;
    @sun.misc.Contended("a")
    long c;
    int d;

    private static Unsafe UNSAFE;

    static {
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            UNSAFE = (Unsafe) f.get(null);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws NoSuchFieldException {
        System.out.println("offset-a: " + UNSAFE.objectFieldOffset(ContendedTest.class.getDeclaredField("a")));
        System.out.println("offset-b: " + UNSAFE.objectFieldOffset(ContendedTest.class.getDeclaredField("b")));
        System.out.println("offset-c: " + UNSAFE.objectFieldOffset(ContendedTest.class.getDeclaredField("c")));
        System.out.println("offset-d: " + UNSAFE.objectFieldOffset(ContendedTest.class.getDeclaredField("d")));

        ContendedTest contendedTest = new ContendedTest();

        // 打印对象的shallow size
        System.out.println("Shallow Size: " + MemoryUtil.memoryUsageOf(contendedTest) + " bytes");
        // 打印对象的 retained size
        System.out.println("Retained Size: " + MemoryUtil.deepMemoryUsageOf(contendedTest) + " bytes");
    }

}
```

这里还是使用到了classmexer.jar，可以参考[Java对象内存布局](../java/jvm/JVM之内存布局与对象大小.md)中的说明。

这里在变量b和c中使用了@sun.misc.Contended注解，并将这两个变量分为1组，执行结果如下：

```java
offset-a: 16
offset-b: 152
offset-c: 160
offset-d: 12
Shallow Size: 296 bytes
Retained Size: 296 bytes
```

可见int类型的变量的偏移地址是12，也就是在对象头后面，因为它正好是4个字节，然后是变量a。@sun.misc.Contended注解的变量会加到对象的最后面，这里就是b和c了，那么b的偏移地址是152，之前说过@sun.misc.Contended注解会在变量前后各加128字节，而byte类型的变量a分配完内存后这时起始地址应该是从17开始，因为byte类型占1字节，那么应该补齐到24，所以b的起始地址是24+128=152，而c的前面并不用加128字节，因为b和c被分为了同一组。

我们算一下c分配完内存后，这时的地址应该到了168，然后再加128字节，最后大小就是296。内存结构如下：

| d:12~16 | --- | a:16~17 | --- | 17~24 | --- | 24~152 | --- | b:152~160 | --- | c:160~168 | --- | 168~296 |

现在把b和c分配到不同的组中，代码做如下修改：

```java
/**
 * VM Options:
 * -javaagent:/Users/sangjian/dev/source-files/classmexer-0_03/classmexer.jar
 * -XX:-RestrictContended
 */
public class ContendedTest {

    byte a;
    @sun.misc.Contended("a")
    long b;
    @sun.misc.Contended("b")
    long c;
    int d;

    private static Unsafe UNSAFE;

    static {
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            UNSAFE = (Unsafe) f.get(null);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws NoSuchFieldException {
        System.out.println("offset-a: " + UNSAFE.objectFieldOffset(ContendedTest.class.getDeclaredField("a")));
        System.out.println("offset-b: " + UNSAFE.objectFieldOffset(ContendedTest.class.getDeclaredField("b")));
        System.out.println("offset-c: " + UNSAFE.objectFieldOffset(ContendedTest.class.getDeclaredField("c")));
        System.out.println("offset-d: " + UNSAFE.objectFieldOffset(ContendedTest.class.getDeclaredField("d")));

        ContendedTest contendedTest = new ContendedTest();

        // 打印对象的shallow size
        System.out.println("Shallow Size: " + MemoryUtil.memoryUsageOf(contendedTest) + " bytes");
        // 打印对象的 retained size
        System.out.println("Retained Size: " + MemoryUtil.deepMemoryUsageOf(contendedTest) + " bytes");
    }

}
```

运行结果如下：

```java
offset-a: 16
offset-b: 152
offset-c: 288
offset-d: 12
Shallow Size: 424 bytes
Retained Size: 424 bytes
```

可以看到，这时b和c中增加了128字节的padding，结构也就变成了：

| d:12~16 | --- | a:16~17 | --- | 17~24 | --- | 24~152 | --- | b:152~160 | --- | 160~288 | --- | c:288~296 | --- | 296~424 |

<br>

------



### 测试Cache Miss

下面的代码引用自[http://coderplay.iteye.com/blog/1485760](https://link.jianshu.com?t=http://coderplay.iteye.com/blog/1485760)：

```java
public class L1CacheMiss {
    private static final int RUNS = 10;
    private static final int DIMENSION_1 = 1024 * 1024;
    private static final int DIMENSION_2 = 62;

    private static long[][] longs;

    public static void main(String[] args) throws Exception {
        longs = new long[DIMENSION_1][];
        for (int i = 0; i < DIMENSION_1; i++) {
            longs[i] = new long[DIMENSION_2];
        }
        System.out.println("starting....");

        final long start = System.nanoTime();
        long sum = 0L;
        for (int r = 0; r < RUNS; r++) {
            // 1. slow
            for (int j = 0; j < DIMENSION_2; j++) {
                for (int i = 0; i < DIMENSION_1; i++) {
                    sum += longs[i][j];
                }
            }
            
            // 2. fast
//            for (int i = 0; i < DIMENSION_1; i++) {
//                for (int j = 0; j < DIMENSION_2; j++) {
//                    sum += longs[i][j];
//                }
//            }
        }
        System.out.println("duration = " + (System.nanoTime() - start));
    }
}
```

这里测试的环境是macOS 10.12.4，JDK 1.8，Java HotSpot(TM) 64-Bit Server VM (build 25.60-b23, mixed mode)。

这里定义了一个二维数组，第一维长度是1024*1024，*第二维长度是62，这里遍历二维数组。由于二维数组中每一个数组对象的长度是62，那么根据[Java对象内存布局](../java/jvm/Java对象内存布局.md)的介绍，可以知道，long类型的数组对象头的大小是16字节（这里默认开启了指针压缩），每个long类型的数据大小是8字节，那么一个long类型的数组大小为16+862=512字节。先看一下第一种慢的方式运行的时间：

```
starting....
duration = 11883939677
```

运行时间是11秒多，再来看下快的方式：

```
starting....
duration = 888085368
```

运行时间是888毫秒，还不到1秒，为什么相差这么多？

首先来分析一下第一种情况，因为二维数组中的每一个数组对象占用的内存大小是512字节，而缓存行的大小是64字节，那么使用第一种遍历方式，假设当前遍历的数据是longs[i][j]，那么下一个遍历的数据是longs[i+1][j]，也就是说遍历的不是同一个数组对象，那么这两次遍历的数据肯定不在同一个缓存行内，也就是产生了Cache Miss；

在第二种情况中，假设当前遍历的数据是longs[i][j]，那么下一个遍历的数据是longs[i][j+1]，遍历的是同一个数组对象，所以当前的数据和下一个要遍历的数据可能都是在同一个缓存行中，这样发生Cache Miss的情况就大大减少了。

<br>

#### Cache Miss总结

一般来说，Cache Miss有三种情况：

1. 第一次访问数据时cache中不存在这条数据；
2. cache冲突；
3. cache已满。

这里的第二种情况也比较常见，同时会产生一个问题，就是伪共享，有时间会单独写一篇文章来介绍一下Java中对伪共享的处理方式。

注：本文参考：https://www.jianshu.com/p/900554f11881

<br>

### 展望

本文涉及到几个重要的知识点尚未完全理清，个人的水平有待提升

1、对象的Padding部分在CUP cache是如何变化的，比如在指针压缩情况？

2、java 要如何在cache line  发挥最好的性能？



