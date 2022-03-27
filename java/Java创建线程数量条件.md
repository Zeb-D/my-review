本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

在跟同事讨论推荐 java 应用在 K8S 容器中 jvm 参数推荐配置的时候， heap 区比较容易达成共识,而 stack 区开始都比较模糊,本篇文章记录一下研究 java thread stack 内存及相关限制。

1、java 最多能创建多少线程? 由以下因素限制:

​    a. stack_size

​    b. max_user_processes

​    c. sys.vm.max_map_count

​    d. sys.kernel.threads-max

​    e. sys.kernel.pid_max

2、java 线程的栈深能有多深?

​    a. stack_size

​    b. 本地变量表



### Java创建线程数量上限

java 中的线程跟 linux 线程是 1:1 关系, 在java 中 new Thread() 为创建一个java 线程, Thread().start() 为启动java 线程, 但是 new Thread() 不会创建一个真正的 linux 线程,而是调用 start 方法之后 Thread().start()，才会创建一个真正 linux 线程。Thread的构造函数是纯Java代码，start方法会调到一个native方法start0里，而start0其实就是JVM_StartThread这个方法。



openjdk: 1.8

jdk/src/hotspot/share/prims/jvm.cpp#2634

```
JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread))
  ...
      jlong size =
             java_lang_Thread::stackSize(JNIHandles::resolve_non_null(jthread));
      size_t sz = size > 0 ? (size_t) size : 0;
      native_thread = new JavaThread(&thread_entry, sz);
      ...
  if (native_thread->osthread() == NULL) {
    ...
    THROW_MSG(vmSymbols::java_lang_OutOfMemoryError(),
              "unable to create new native thread");
  }
  Thread::start(native_thread);
JVM_END
```

关注下最后的那个if判断if (native_thread->osthread() == NULL)，如果osthread为空，那将会抛出大家比较熟悉的unable to create new native thread OOM异常，因此osthread为空非常关键，后面会看到什么情况下osthread会为空。

另外大家应该注意到了native_thread = new JavaThread(&thread_entry, sz)，在这里才会真正创建一个线程。



openjdk: 1.8

jdk/src/hotspot/share/runtime/thread.cpp#1443

```
JavaThread::JavaThread(ThreadFunction entry_point, size_t stack_sz) :
  Thread()
#ifndef SERIALGC
  , _satb_mark_queue(&_satb_mark_queue_set),
  _dirty_card_queue(&_dirty_card_queue_set)
#endif // !SERIALGC
{
  if (TraceThreadEvents) {
    tty->print_cr("creating thread %p", this);
  }
  initialize();
  _jni_attach_state = _not_attaching_via_jni;
  set_entry_point(entry_point);
  // Create the native thread itself.
  // %note runtime_23
  os::ThreadType thr_type = os::java_thread;
  thr_type = entry_point == &compiler_thread_entry ? os::compiler_thread :
                                                     os::java_thread;
  os::create_thread(this, thr_type, stack_sz);
}

```

上面代码里的os::create_thread(this, thr_type, stack_sz)会通过pthread_create来创建线程，对应 代码如下：



openjdk: 1.8

jdk/src/hotspot/os/linux/os_linux.cpp#891

```
bool os::create_thread(Thread* thread, ThreadType thr_type, size_t stack_size) {
  assert(thread->osthread() == NULL, "caller responsible");
  // Allocate the OSThread object
  OSThread* osthread = new OSThread(NULL, NULL);
  if (osthread == NULL) {
    return false;
  }
  // set the correct thread state
  osthread->set_thread_type(thr_type);
  // Initial state is ALLOCATED but not INITIALIZED
  osthread->set_state(ALLOCATED);
  thread->set_osthread(osthread);
  // init thread attributes
  pthread_attr_t attr;
  pthread_attr_init(&attr);
  pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
  // stack size
  if (os::Linux::supports_variable_stack_size()) {
    // calculate stack size if it's not specified by caller
    if (stack_size == 0) {
      stack_size = os::Linux::default_stack_size(thr_type);
      switch (thr_type) {
      case os::java_thread:
        // Java threads use ThreadStackSize which default value can be
        // changed with the flag -Xss
        assert (JavaThread::stack_size_at_create() > 0, "this should be set");
        stack_size = JavaThread::stack_size_at_create();
        break;
      case os::compiler_thread:
        if (CompilerThreadStackSize > 0) {
          stack_size = (size_t)(CompilerThreadStackSize * K);
          break;
        } // else fall through:
          // use VMThreadStackSize if CompilerThreadStackSize is not defined
      case os::vm_thread:
      case os::pgc_thread:
      case os::cgc_thread:
      case os::watcher_thread:
        if (VMThreadStackSize > 0) stack_size = (size_t)(VMThreadStackSize * K);
        break;
      }
    }
    stack_size = MAX2(stack_size, os::Linux::min_stack_allowed);
    pthread_attr_setstacksize(&attr, stack_size);
  } else {
    // let pthread_create() pick the default value.
  }
  // glibc guard page
  pthread_attr_setguardsize(&attr, os::Linux::default_guard_size(thr_type));
  ThreadState state;
  {
    // Serialize thread creation if we are running with fixed stack LinuxThreads
    bool lock = os::Linux::is_LinuxThreads() && !os::Linux::is_floating_stack();
    if (lock) {
      os::Linux::createThread_lock()->lock_without_safepoint_check();
    }
    pthread_t tid;
    int ret = pthread_create(&tid, &attr, (void* (*)(void*)) java_start, thread);
    pthread_attr_destroy(&attr);
    if (ret != 0) {
      if (PrintMiscellaneous && (Verbose || WizardMode)) {
        perror("pthread_create()");
      }
      // Need to clean up stuff we've allocated so far
      thread->set_osthread(NULL);
      delete osthread;
      if (lock) os::Linux::createThread_lock()->unlock();
      return false;
    }
    // Store pthread info into the OSThread
    osthread->set_pthread_id(tid);
    ...
  }
  ...
  return true;
}
```

如果在new OSThread的过程中就失败了，那显然osthread为NULL，那再回到上面第一段代码，此时会抛出java.lang.OutOfMemoryError: unable to create new native thread的异常，而什么情况下new OSThread会失败，比如说内存不够了，而这里的内存其实是C Heap，而非Java Heap，指的是Linux 剩余内存。由此可见从JVM的角度来说，影响线程创建的因素包括了Xmx，MaxPermSize，MaxDirectMemorySize，ReservedCodeCacheSize等，因为这些参数会影响剩余的内存 。

另外注意到如果pthread_create执行失败，那通过thread->set_osthread(NULL)会设置空值，这个时候osthread也为NULL，因此也会抛出上面的OOM异常，导致创建线程失败，因此接下来要分析下pthread_create失败的因素。



#### glibc

pthread_create的实现在glibc里。

glibc



glibc/sysdeps/unix/sysv/linux/createthread.c#624

```
int
__pthread_create_2_1 (pthread_t *newthread, const pthread_attr_t *attr,
                      void *(*start_routine) (void *), void *arg)
{
  STACK_VARIABLES;
  const struct pthread_attr *iattr = (struct pthread_attr *) attr;
  struct pthread_attr default_attr;
  ...
  struct pthread *pd = NULL;
  int err = ALLOCATE_STACK (iattr, &pd);
  int retval = 0;
  if (__glibc_unlikely (err != 0))
    {
      retval = err == ENOMEM ? EAGAIN : err;
      goto out;
    }
  ...
 }
```

上面我主要想说的一段代码是int err = ALLOCATE_STACK (iattr, &pd)，顾名思义就是分配线程栈，简单来说就是根据iattr里指定的stackSize，通过mmap分配一块内存出来给线程作为栈使用。

那我们来说说stackSize，这个大家应该都明白，线程要执行，要有一些栈空间，试想一下，如果分配栈的时候内存不够了，是不是创建肯定失败？而stackSize在JVM下是可以通过-Xss指定的，当然如果没有指定也有默认的值，下面是JDK6之后(含)默认值的情况。



openjdk: 1.8

文件位置:jdk/src/hotspot/os_cpu/linux_x86/os_linux_x86.cpp#494

```
// return default stack size for thr_type
size_t os::Posix::default_stack_size(os::ThreadType thr_type) {
  // default stack size (compiler thread needs larger stack)
#ifdef AMD64
  size_t s = (thr_type == os::compiler_thread ? 4 * M : 1 * M);
#else
  size_t s = (thr_type == os::compiler_thread ? 2 * M : 512 * K);
#endif // AMD64
  return s;
}
```

估计不少人有一个疑问，栈内存到底属于-Xmx控制的Java Heap里的部分吗，这里明确告诉大家不属于，因此从glibc的这块逻辑来看，JVM里的Xss也是影响线程创建的一个非常重要的因素。也就是 stack_size。



#### Linux

如果栈分配成功，那接下来就要创建线程了，大概逻辑如下：



openjdk: 1.8

jdk/src/hotspot/os/linux/os_linux.cpp#891

```
static int create_thread (struct pthread *pd, const struct pthread_attr *attr,
        bool *stopped_start, void *stackaddr,
        size_t stacksize, bool *thread_ran)
{
  struct clone_args args =
    {
      .flags = clone_flags,
      .pidfd = (uintptr_t) &pd->tid,
      .parent_tid = (uintptr_t) &pd->tid,
      .child_tid = (uintptr_t) &pd->tid,
      .stack = (uintptr_t) stackaddr,
      .stack_size = stacksize,
      .tls = (uintptr_t) tp,
    };
  int ret = __clone_internal (&args, &start_thread, pd);
  if (__glibc_unlikely (ret == -1))
    return errno;
 ...
}
```

而create_thread其实是调用的系统调用clone ,clone系统调用最终会调用do_fork方法，接下来通过剖解这个方法来分析Kernel里还存在哪些因素。

#### max_user_process



linux: 3.10

kernel/fork.c#1199

```
retval = -EAGAIN;
  if (atomic_read(&p->real_cred->user->processes) >=
      task_rlimit(p, RLIMIT_NPROC)) {
    if (!capable(CAP_SYS_ADMIN) && !capable(CAP_SYS_RESOURCE) &&
        p->real_cred->user != INIT_USER)
      goto bad_fork_free;
  }

```

先看这么一段，这里其实就是判断用户的进程数有多少，大家知道在linux下，进程和线程其数据结构都是一样的，因此这里说的进程数可以理解为轻量级线程数，而这个最大值是可以通过ulimit -u可以查到的，所以如果当前用户起的线程数超过了这个限制，那肯定是不会创建线程成功的。



#### max_map_count

在这个过程中不乏有malloc的操作，底层是通过系统调用brk来实现的，或者上面提到的栈是通过mmap来分配的，不管是malloc还是mmap，在底层都会有类似的判断

linux: 3.10

mm/mmap.c#2480

```
if (mm->map_count >= sysctl_max_map_count)
    return -ENOMEM;

```

如果进程被分配的内存段超过sysctl_max_map_count就会失败，而这个值在linux下对应/proc/sys/vm/max_map_count，默认值是65530，可以通过修改上面的文件来改变这个阈值。

#### max_threads

linux: 3.10

kernel/fork.c#1217

```
/*
   * If multiple threads are within copy_process(), then this check
   * triggers too late. This doesn't hurt, the check is only there
   * to stop root fork bombs.
   */
  retval = -EAGAIN;
  if (nr_threads >= max_threads)
    goto bad_fork_cleanup_count;
```

该值是系统全局（包括非 JVM 进程）最大线程数。查看和修改该值: /proc/sys/kernel/threads-max,这个值是受到物理内存的限制，在fork_init的时候就计算好了，也就是说linux 主机内存越大改值越大。



linux: 3.10

kernel/fork.c#267

```
/*
   * The default maximum number of threads is set to a safe
   * value: the thread structures can take up at most half
   * of memory.
   */
  max_threads = mempages / (8 * THREAD_SIZE / PAGE_SIZE);
  /*
   * we need to allow at least 20 threads to boot a system
   */
  if (max_threads < 20)
    max_threads = 20;
1.2.3.4.5.6.7.8.9.10.11.
```

#### pid_max

pid也存在限制



linux: 3.10

kernel/fork.c#1350

```
if (pid != &init_struct_pid) {
    retval = -ENOMEM;
    pid = alloc_pid(p->nsproxy->pid_ns);
    if (!pid)
      goto bad_fork_cleanup_io;
  }
1.2.3.4.5.6.
```



而alloc_pid的定义如下：



linux: 3.10

kernel/pid.c#287

```
struct pid *alloc_pid(struct pid_namespace *ns)
{
  struct pid *pid;
  enum pid_type type;
  int i, nr;
  struct pid_namespace *tmp;
  struct upid *upid;
  pid = kmem_cache_alloc(ns->pid_cachep, GFP_KERNEL);
  if (!pid)
    goto out;
  tmp = ns;
  pid->level = ns->level;
  for (i = ns->level; i >= 0; i--) {
    nr = alloc_pidmap(tmp);
    if (nr < 0)
      goto out_free;
    pid->numbers[i].nr = nr;
    pid->numbers[i].ns = tmp;
    tmp = tmp->parent;
  }
  ...
}
```

在alloc_pidmap中会判断pid_max,而这个值的定义如下：

linux: 3.10

include/linux/threads.h#27

```
/*
 * This controls the default maximum pid allocated to a process
 */
#define PID_MAX_DEFAULT (CONFIG_BASE_SMALL ? 0x1000 : 0x8000)
/*
 * A maximum of 4 million PIDs should be enough for a while.
 * [NOTE: PID/TIDs are limited to 2^29 ~= 500+ million, see futex.h.]
 */
#define PID_MAX_LIMIT (CONFIG_BASE_SMALL ? PAGE_SIZE * 8 : \
  (sizeof(long) > 4 ? 4 * 1024 * 1024 : PID_MAX_DEFAULT))
```

linux: 3.10

kernel/pid.c#48

```
int pid_max = PID_MAX_DEFAULT;
#define RESERVED_PIDS    300
int pid_max_min = RESERVED_PIDS + 1;
int pid_max_max = PID_MAX_LIMIT;
```

这个值可以通过/proc/sys/kernel/pid_max来查看或者修改。



#### 小结:

通过对JVM，glibc，Linux kernel的源码分析，我们暂时得出了一些结论:

- java thread stack 使用的内存属于系统可用内存不归 jvm heap,no heap 管理。
-  java thread stack size 默认是 1MB，改值也是影响能创建多少线程数的因素之一。
- linux kernel 层面因素有

-  max_user_processes
-  max_map_count
- mac_threads
-  pid_max



### Java 线程的栈深上限

线程中的 栈结构如下：

![img](https://s8.51cto.com/oss/202203/21/06aca95468b82584cd2731d7b27a833f732456.jpg)

每个栈帧包含：本地变量表，操作数栈，动态链接，返回地址等东西...也就是说栈调用深度越大，栈帧就越多，就越耗内存。

#### 小结:

- java threand stack depth 栈帧 没有一个固定的参数设置。由系统内存, stack size 动态限制。
-  java thread stack size 指的是一个线程能用到的最多内存。
- java thread stack size 越大，拥有的 栈帧就越多,栈深也就越深。
-   线程中每一个栈帧如本地变量表越小，在固定的 stack size 下，栈帧数量也就越多，栈深也就越深。



### 实验

#### 环境

| 主机配置         | 32C 128G   |
| ---------------- | ---------- |
| 操作系统         | Linux 4.19 |
| max_user_process | 506354     |
| max_threads      | 1012708    |
| pid_max          | 4194303    |
| max_map_count    | 262144     |

#### java demo:

```
private static Object s = new Object();
    private static int count = 0;
    @RequestMapping(value = "/threads", method = RequestMethod.GET)
    public String threads() {
        log.info("threads");
        for (; ; ) {
            new Thread(new Runnable() {
                public void run() {
                    synchronized (s) {
                        count += 1;
                        log.info("New thread #" + count);
                    }
                    for (; ; ) {
                        try {
                            Thread.sleep(1000);
                        } catch (Exception e) {
                            System.err.println(e);
                        }
                    }
                }
            }).start();
        }
    }
```

在该主机上直接运行: 能创建 130,702 个线程。

```
2022-03-20 14:19:25.642  INFO 3112536 --- [  Thread-130703] com.example.opsdemo.IndexController      : New thread #130702
Java HotSpot(TM) 64-Bit Server VM warning: INFO: os::commit_memory(0x00007efcebec6000, 12288, 0) failed; error='Cannot allocate memory' (errno=12)
#
# There is insufficient memory for the Java Runtime Environment to continue.
# Native memory allocation (mmap) failed to map 12288 bytes for committing reserved memory.
# An error report file with more information is saved as:
# /tmp/opsdemo/hs_err_pid3112536.log
2022-03-20 14:19:25.652 ERROR 3112536 --- [nio-8888-exec-1] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Handler dispatch failed; nested exception is java.lang.OutOfMemoryError: unable to create new native thread] with root cause
java.lang.OutOfMemoryError: unable to create new native thread
```

在该主机上以 pod 的方式部署容器运行,能创建 16,334

```
2022-03-20 13:11:34.403  INFO 7 --- [   Thread-16335] com.example.opsdemo.IndexController      : New thread #16334
2022-03-20 13:11:34.419 ERROR 7 --- [nio-8888-exec-1] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Handler dispatch failed; nested exception is java.lang.OutOfMemoryError: unable to create new native thread] with root cause
java.lang.OutOfMemoryError: unable to create new native thread
  at java.lang.Thread.start0(Native Method) [na:1.8.0_242]
  at java.lang.Thread.start(Thread.java:744) [na:1.8.0_242]
```

#### 小结

- 在主机上直接运行能看见直接受限于内存能创建 13 W线程。
- 在 pod 上运行，能创建 1.6W 线程，显然还受限于其它机制,大概率还是 cgroup memory 相关限制。