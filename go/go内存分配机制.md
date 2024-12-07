本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 序

golang 自带内存分配、GC等功能，这些功能称为golang runtime，这里会开两篇来分开将内存机制、以及GC机制。后面会给出优化实践篇。

Golang的内存管理是建立在OS的内存管理之上的，它目的就是尽可能的会发挥操作系统层面的优势，而避开导致低效情况。Golang的内存管理机制受到了TCMalloc内存分配算法库的深刻影响，并在此基础上进行了优化。

Golang 的内存机制也是建立在操作系统OS的内存基础上，所以我们先科普一些OS内存概念，然后在细讲golang的内存分配。



内存管理是指软件运行时对计算机内存资源的分配和使用的技术。其最主要的目的是如何高效，快速的分配，并且在适当的时候释放和回收内存资源。



### 操作系统内存管理

操作系统将RAM空间分成两部分：

一部分用于存放内核映像（也就是内核代码和内核静态数据结构）；

另一部分通常由虚拟内存系统处理，这部分RAM称为动态内存（Dynamic Memory），不仅是进程所需要的宝贵资源，也是内核本身所需的宝贵资源。



#### 虚拟内存

虚拟内存（Virtual Memory）是操作系统提供的一种抽象，作为一种逻辑层，处于应用程序的内存请求与硬件内存管理单元（Memory Management Unit，MMU）之间。

进程所用的一组内存地址不同于物理内存地址。当进程使用一个虚拟地址时，内核和MMU协同定位其在内存的实际物理位置。

虚拟内存有很多用途和优点：

- 若干个进程可以并发地执行。
- 应用程序所需内存大于可用物理内存时也可以运行。
- 程序只有部分代码装入内存时进程可以执行它。
- 允许每个进程访问可用物理内存的子集。
- 进程可以共享库函数或程序的一个单独[内存映像](https://zhida.zhihu.com/search?content_id=240958713&content_type=Article&match_order=1&q=内存映像&zhida_source=entity)。
- 程序是可以重定位的，也就是可以把程序放在物理内存的任何地方。
- 程序员可以编写与机器无关的代码，因为他们不必关心有关物理内存的组织结构。



#### 内存地址

逻辑地址（Logical Address）：是程序员所使用的地址，每个逻辑地址都由一个段（Segment）和偏移量（Offset）组成，偏移量指明了从段开始的地方到实际地址之间的距离。（只有在Intel实模式下，逻辑地址才和物理地址相等，因为实模式没有分段或分页机制，CPU不进行自动地址转换）

线性地址（Linear Address）：也称为虚拟地址（Virtual Address），线性地址是一个连续的地址空间，对于32位系统来说，其地址范围从0x0000000到0xFFFFFFF。 通过分段机制将逻辑地址转换为线性地址。

物理地址（Physical Address）：内存控制器实际用来访问主存芯片上的存储单元的实际地址。在线性地址的基础上，通过分页机制将其再次转换为物理地址。每一个物理地址对应着主存中的唯一位置。

内存管理单元（MMU）通过一种称为分段单元（Segmentation Unit）的硬件电路把一个逻辑地址转换成线性地址；

接着，第二个称为分页单元（Paging Unit）的硬件电路把线性地址转换成物理地址。



#### 分段

##### 硬件中的分段

一个逻辑地址由段选择符（Segment Selector）和段内偏移量（Offset）组成。段选择符指向一个段描述符（Segment Descriptor），段描述符包含段基址（Base Address），段基址加上偏移量就得到了线性地址。

为了快速方便找到段选择符，处理器提供了段寄存器，段寄存器的唯一目的是存放段选择符，有6个段寄存器：CS（Code Segment）、DS（Data Segment）、SS（Stack Segment）、ES（Extra Segment）、FS（Flag Segment）、GS（Global Segment）。

在分段存储中，一般分为以下几个段

1. 代码段（Code Segment）： 存储程序的指令，也称为可执行代码。这部分内存包含了程序的机器指令，用于实际执行程序的操作。
2. 数据段（Data Segment）： 存储已初始化的全局和静态变量。这些变量在程序开始运行时就已经被初始化并占用内存空间。
3. 未初始化数据段（BSS Segment）： 存储未初始化的全局和静态变量。这些变量在程序开始运行时会被初始化为默认值（通常为0），但实际的初始化操作会在运行时完成。
4. 堆栈段（Stack Segment）： 存储程序的执行堆栈。堆栈用于存储函数调用和局部变量等数据，以及管理函数的调用和返回。
5. 堆段（Heap Segment）： 存储动态分配的内存，用于程序在运行时动态申请和释放的内存。例如，使用 malloc() 或 new 函数分配的内存就位于堆段。



每个段由一个8字节的段描述符表示，它描述了段的特征。段描述符存放在GDT（Global Descriptor Table）或LDT（Local Descriptor Table）中。



逻辑地址转换为线性地址分段单元执行一下操作：

1、处理器从段寄存器中取出段选择符，根据段选择符的TI字段来确定段描述符是在GDT或LDT中（获取到GDT或LDT的线性基地址）。

2、从段选择符的index字段计算段描述符的地址。

3、从上述段描述符的地址取出段描述符，将段描述符的Base字段的值加上逻辑地址的偏移量得到线性地址。



##### 操作系统中的分段

Linux以非常有限的方式使用分段。实际上分段和分页在某种程度上有点多余，因为它们都可以划分进程的物理空间：分段可以给每个一个进程分配不同的线性地址空间；

而分页可以把同一线性地址空间映射到不同的物理空间。与分段相比，Linux更喜欢使用分页方式，因为当所有进程使用相同的段寄存器值时，内存管理变得更简单，也就是说它们能共享同样的一组线性地址空间。

 Linux只有在80x86架构下才需要使用分段，在多处理器系统中每个CPU对应一个GDT，每个GDT中包含18个Linux段的段描述符，主要有用户代码段、用户数据段、内核代码段、内核数据段和局部线程存储（Thread-Local Storage, TLS）段。

大多数用户态下的Linux程序不使用LDT。所有的段都是从0x00000000开始，因此Linux下的逻辑地址与线性地址是一致的，即逻辑地址的偏移量字段的值与相应的线性地址的值总是一致的。



#### 分页

##### 硬件中的分页

分页单元把线性地址转换成物理地址。其中的一个关键任务是把所有请求的访问类型与线性地址的权限比较，如果这次访问是无效的，就会产生一个缺页异常。

为了效率起见，线性地址被分成以固定长度为单位的组，称为页（Page）。

页内部连续的线性地址被映射到连续的物理地址中。这样内核可以指定一个页的物理地址和其存取权限，而不用指定页所包含的全部线性地址的存取权限。分页单元把所有的RAM分成固定长度的页框（Page Frame）。

每一个页框包含一个页（Page），也就是一个页框的长度与一个页的长度一致。页框是主存的一部分，因此也是一个存储区域。区分一页和一个页框是很重要的，前者只是一个数据块，可以存放在任何页框或磁盘中。把线性地址映射到物理地址空间的数据结构称为页表（Page Table）。页表存放在主存中，并在启用分页单元之前必须由内核对页表进行适当的初始化。

 从80386架构开始，Intel处理器的分页单元处理4KB的页。

32位的线性地址空间被分成3个域：Directory（目录，最高10位）、Table（页表，中间10位）、Offset（偏移量）。

线性地址的转换分两步完成，每一步都基于一种转换表，第一种转换表称为页目录表（Page Directory），第二种转换表称为页表（Page Table）。

页目录存放在一个页框中，页表存放在另一个页框中。使用这种二级模式的目的在于减少每个进程页表所需RAM的数量。然后两级分页机制并不适用于64位处理器。

同时为了缩小CPU与RAM之前的速度不匹配，CPU引入了硬件高速缓存内存，硬件高速缓存基于局部性原理（Locality Principle）。

高速缓存单元插在分页单元和主存之间，它包含一个硬件高速缓存（Hardware Cache Memory）和一个高速缓存控制器（Cache Controller）。高速缓存内存存放内存中真正的行（Line）。



##### 操作系统中的分页

Linux采用了一种同时适用于32位和64位系统的普通分页模型。

线性地址被分成5个部分：Page Global Directory（页全局目录，PGD）、Page Upper Directory（页上级目录，PUD）、Page Middle Directory（页中间目录，PMD）、Page Table（页表，PT）和Offset（偏移量）。

每个部分的大小与具体的计算机体系结构有关。对于没有启用物理地址扩展的32位系统，两级页表已经足够了，Linux通过使页上级目录位和页中间目录位为0，取消了页上级目录和页中间目录。

启用了物理地址扩展的32位系统，取消了页上级目录。64位系统使用三级还是四级分页取决硬件对线性地址的划分。



#### 物理内存

在初始化阶段，内核必须建立一个物理地址映射来指定哪些物理地址范围对内核可用而哪些不可用。



##### 页框管理

Linux采用4KB页框大小作为标准的内存分配单位。内核必须记录每个页框的当前状态。

例如，内核必须能区分哪些页框包含的是属于进程的页，而哪写页框包含的是内核代码或内核数据；以及能够确定动态内存中的页框是否空闲。页框的状态信息保存在一个类型为page的页描述符中。

Linux 2.6支持非一致性内存访问（Non-Uniform Memory Access， NUMA）模型，在这种模型中，给定CPU对不同内存单元的访问时间可能不一样。系统的物理内存被划分为几个节点（Node）。

每个节点的物理内存又可以分为3个管理区（Zone）：ZONE_DMA、ZONE_NORMAL、ZONE_HIGHMEM（在64位硬件平台，因为可使用的线性地址空间远大于能安装的RAMRAM大小，这些体系结构的ZONE_HIGHMEM管理区总是空的）。

当内核调用一个内存分配函数式，必须指明请求页框所在的管理区。被称为分区页框分配器（Zoned Page Frame Allocator）的内核子系统，处理对连续页框组的内存分配请求。

然后在每个管理区内，页框被名为伙伴系统（Buddy System）的部分来处理。使用函数/宏`alloc_pages()`、`__get_free_pages()`等请求页框和`__free_pages()`、`free_pages()`释放页框。



伙伴系统（Buddy System）：内核应该为分配一组连续的页框而建立一种健壮、高效的分配策略。

为此必须解决著名的内存管理问题，也就是所谓的外碎片（External Fragmentation）。频繁地请求和释放不同大小的一组连续页框，必然导致在已分配页框的块间分散了许多小块的空闲页框。

由此带来的问题是，即使有足够的空闲页框可以满足请求，但要分配一个大块的连续页框就可能无法满足。

本质上说，避免外碎片的方法有两种：

1. 利用分页单元把一组非连续的空闲页框映射到连续的线性地址空间。
2. 开发一种适当的技术来记录现存的空闲连续页框块的情况，以尽量避免为满足对小块的请求而分割大的空闲块。



Linux内核首先采用第二种方法，采用了著名的伙伴系统（Buddy System）算法来解决外碎片问题。把所用的空闲页框分组为11个块链表，每个块链表分别包含大小为1，2，4，8，16，32，64，128，256，512和1024个连续页框。

对1024个页框的最大请求对应着4MB大小的连续RAM块。每个块的第一个页框的物理地址是该块大小的整数倍。该算法的工作原理：假设要请求一个256个页框的连续RAM块（即1MB），算法先在256个页框的链表中检查是否有一个空闲块。

如果没有这样的块，算法会查找下一个更大的页块，也就是在512个页框的链表中找一个空闲块。如果存在这样的块，内核就把512的页框分成两等份，一半用作满足请求，另一半用插入256个页框的链表中。如果在512个页框的链表中也没找到空闲块，就继续找更大的块1024个页框的块。

如果这样的块存在，内核把1024个页框块的256个页框用作请求，然后从剩余的768个页框中拿512个插入到512个页框的链表中，再把最后的256个插入到256个页框的链表中。如果1024个页框的链表还是空的，算法就放弃并发出错误信号。

上述过程的逆过程就是页框块的释放过程，也是该算法名字的由来。内核试图把大小为b的一对空闲伙伴合并为一个大小为2b的单独块。满足以下条件的两个块称为伙伴：

- 两个块具有相同的大小，记作b。
- 它们的物理地址是连续的。
- 第一个块的第一个页框的物理地址是2 * b * 2^12的整数倍。

该算法是迭代的，如果它成功合并所释放的块，它会试图合并2b的块，以再次试图形成更大的块。



##### 内存区管理

内存区（Memory Area）是具有连续的物理地址和任意长度的内存单元序列。伙伴系统算法采用页框作为基本内存区，这适合于对大块内存的请求，但如何处理对小内存的请求呢，比如几十或几百个字节？

使用一种新的数据结构来描述在同一页框中如何分配小内存区。但这样也引入了一个新问题，即所谓的内碎片（Internal Fragmentation）。内碎片的产生主要是由于请求内存的大小与分配给它的大小不匹配而造成的。

一种更好的内存区分配算法源自于slab分配器模式，该模式最早用于Sun公司的Solaris操作系统中。该算法基于以下前提：

- 所存放数据的类型可以影响内存区的分配方式。
- 内核函数倾向于反复请求同一类型的内存区。
- 对内存区的请求可以根据它们发生的频率来分类。
- 引入的对象大小不是几何分布的情况下，就是说，数据结构的起始物理地址不是2的幂次，这可以借助处理器硬件高速缓存而导致较好的性能。
- 硬件高速缓存的高性能又是尽可能地限制对伙伴系统分配器调用的另一个理由，因为对伙伴系统函数的每次调用都“弄脏”硬件高速缓存，所以增加了对内存的平均访问时间。

slab分配器把对象分组放进高速缓存。每个高速缓存都是同种类型对象的一种“储备”。包含高速缓存的主内存区被划分为多个slab，每个slab由一个或多个页框组成，这些页框中既包含已分配的对象，也包含空闲的对象。高速缓存中的每个slab都有自己的类型为slab的描述符。



##### 非连续内存区管理

上述页框管理和内存区管理都是对连续物理内存区的处理，把内存区映射到一组连续的页框是最好的选择，这样会充分利用高速缓存并获得较低的平均访问时间。不过，如果对内存区的请求不是很频繁，那么通过连续的线性地址来访问非连续的页框这样一种分配模式就会很有意义。这中模式的主要优点是避免了外碎片，而缺点是必须打乱内核页表。



#### 页面置换策略

要知道固定分配局部置换、可变分配全部置换、可变分配局部置换的意思，首先需要知道以下几个概念：

1.固定分配：操作系统为每个进程分配一组固定数目大小的物理块。在程序运行过程中，不允许改变！即驻留集大小固定不变。

2.可变分配：先为每个进程分配一定大小的物理块，在程序运行过程中，可以动态改变物理块的大小。即，驻留集大小可变。

3.局部置换：进程发生缺页时，只能选择当前进程中的物理块进行置换。

4.全局置换：可以将操作系统进程中保留的空闲物理块分配给缺页进程，还可以将别的进程持有的物理块置换到外存，再将这个物理块分配给缺页的进程。



##### 固定分配局部置换

**概念**：系统为每个进程分配一定数量的内存块（物理块），在整个运行期都不改变。若进程在运行过程中发生了缺页，则只能在本进程的内存页面中选出一个进行换出，然后再调用需要的页面。
**缺点**：很难确定一个进程到底应该分配多大的实际内存才合理



##### 可变分配局部置换

**概念**：刚开始会为每个进程分配一定数量的物理块。当进程发生缺页时，只允许从当前进程的物理块中选出一个换出外存。如果当前进程在运行的时候频繁缺页，系统会为该进程动态增加一些物理块，直到该进程缺页率趋于适中程度；如果说一个进程在运行过程中缺页率很低或者不缺页，则可以适当减少该进程分配的物理块。通过这些操作可以保持多道程序的并发度较高。
**缺点：**使用可变分配局部置换时，每个进程会被分配一个可变大小的页面块，这可能会导致内存中出现许多不同大小的空闲块，从而增加内存碎片的问题。这些碎片可能会导致内存空间不连续，使得无法为较大的进程分配足够的内存。



##### 可变分配全局置换

**概念**：系统为每个进程分配一定数量的内存块（物理块）。操作系统还会保持一个空闲物理块的队列。若某进程发生缺页，可以从空闲物理块中取出一块分配给该进程。如果空闲物理块没有了，那么会选择一个未锁定（不是那么重要）的页面换出到外存，再将物理块分配给缺页的进程。
**缺点**：在空闲物理块没有的情况下，如果将其他进程的页面调出到外存，那么这个进程就会拥有较小的驻留集，如此会导致该进程的缺页率上升
看到这，为什么没有固定分配全局置换呢？
如果理解上面的分配和置换，那应该很容易明白，这种情况下，固定和全局本身就是矛盾的，不可能在固定的情况下，还进行全局的置换，因为全局的置换会分配其他的空闲物理块。



#### 页面调度时机

预调页策略：基于局部性原理，一次调入若干个相邻页面可能比一次调入一个页面更高效。
缺点：如果调入的若干页面是不会被马上访问的，那么这样效率又会很低。
请求调页策略：只有在进程处于运行期，且发生缺页的时候才被调入内存。
缺点：缺页时每次只会调入一页，每次从外存到内存的调入都会进行I/O操作，因此I/O开销较大。



#### 页面置换算法

这个部分的内容暂且就先简略的介绍一下，因为感觉理论基本都很容易看明白。
OPT是最好的页面置换算法，所有的置换算法都是为了接近它的性能而进行优化。

1. 最佳（OPT，Optimal）： 这是一种理论上的置换策略，它总是选择未来最长时间内不会被访问的页面进行置换。虽然 OPT 算法可以获得最小的缺页次数，但是在实际情况下，无法预知未来的访问模式，因此很难实现。
2. 先进先出（FIFO）： 这是最简单的页面置换策略之一。它总是选择最早进入物理内存的页面进行置换。然而，FIFO 算法可能会导致 "Belady's Anomaly"，即增加页面数时，缺页次数反而增加。
3. 最近未使用（LRU，Least Recently Used）： 这个策略选择最长时间未被使用的页面进行置换。具体来说，它会将最近最少被访问的页面换出。LRU 算法可以有效地减少缺页次数，但实现起来较为复杂，可能需要较大的开销来维护页面的访问历史。
4. 最近最少使用（LFU，Least Frequently Used）： 这个策略选择在一段时间内被访问次数最少的页面进行置换。LFU 算法考虑了页面的使用频率，但同样需要维护访问计数，可能增加开销。
5. 时钟置换算法（Clock Replacement Algorithm）:也称为"二次机会"（Second-Chance）页面置换算法，是一种用于虚拟内存管理的页面置换策略。它是基于近似最近未使用（LRU）算法的一种改进，旨在降低实现复杂度，同时在某种程度上模拟LRU的效果。



#### 页面缓冲思想

页面缓冲思想（Page Buffering）是指在计算机系统中，通过缓存（缓冲）页面数据来优化数据访问和管理的策略。这种思想通常应用于磁盘和文件系统等存储系统，旨在减少数据读取和写入的延迟，提高数据访问速度和效率。
页面缓冲的核心思想是将最常用的数据页（或数据块）暂时存储在内存中，以便在需要时能够更快地访问。具体来说，当应用程序需要从磁盘读取数据时，系统首先检查缓冲区中是否已经存在所需的数据页。如果数据页已经在缓冲区中，那么可以直接从内存中读取，避免了慢速的磁盘访问。如果数据页不在缓冲区中，系统会将它从磁盘读取到缓冲区，并且可能还会替换掉缓冲区中的其他数据页。
页面缓冲的优点包括：

1. 加速数据访问： 缓冲数据可以显著加快数据的读取速度，因为内存的访问速度比磁盘快得多。
2. 降低磁盘访问频率： 缓冲数据可以减少对磁盘的频繁访问，从而减少磁盘的负载，延长磁盘寿命，并提高整体系统性能。
3. 平滑访问流量： 缓冲数据可以平滑应用程序对存储系统的访问流量，避免突发性的大量磁盘访问，从而提高系统的稳定性。
4. 缓解抖动： 页面缓冲可以减少抖动问题，即减少频繁的页面置换，从而避免系统性能下降。

然而，页面缓冲也可能带来一些挑战，如：

1. 内存管理： 页面缓冲需要占用一部分内存空间，因此需要合理管理内存资源，避免内存不足导致性能下降。
2. 缓冲命中率： 缓冲命中率指缓冲区中已缓存的数据在总的数据访问中的比例。缓冲命中率越高，性能提升越明显。但如果缓冲命中率很低，则可能带来额外的开销。
3. 一致性： 缓冲数据与磁盘上的数据之间需要保持一致性，因此需要适当的缓冲管理机制，以确保数据的正确性。



### TCMalloc

TCMalloc最大优势就是每个线程都会独立维护自己的内存池。在之前章节介绍的自定义实现的Golang内存池版BufPool实则是所有Goroutine或者所有线程共享的内存池，其关系如图18所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651132777839-f6077cf7-f8e4-40d0-9fa0-1167208508da.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_58%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

图 18 BufPool内存池与线程Thread的关系



这种内存池的设计缺点显而易见，应用方全部的内存申请均需要和全局的BufPool交互，为了线程的并发安全，那么频繁的BufPool的内存申请和退还需要加互斥和同步机制，影响了内存的使用的性能。

TCMalloc则是为每个Thread预分配一块缓存，每个Thread在申请内存时首先会先从这个缓存区ThreadCache申请，且所有ThreadCache缓存区还共享一个叫CentralCache的中心缓存。这里假设目前Golang的内存管理用的是原生TCMalloc模式，那么线程与内存的关系将如图19所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651132869540-a130e8b3-1f7d-45ec-8413-52bba81426a0.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_57%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

图 19 TCMalloc内存池与线程Thread的关系



这样做的好处其一是ThreadCache做为每个线程独立的缓存，能够明显的提高Thread获取高命中的数据，其二是ThreadCache也是从堆空间一次性申请，即只触发一次系统调用即可。

每个ThreadCache还会共同访问CentralCache，这个与BufPool的类似，但是设计更为精细一些。CentralCache是所有线程共享的缓存，当ThreadCache的缓存不足时，就会从CentralCache获取，当ThreadCache的缓存充足或者过多时，则会将内存退还给CentralCache。

但是CentralCache由于共享，那么访问一定是需要加锁的。ThreadCache作为线程独立的第一交互内存，访问无需加锁，CentralCache则作为ThreadCache临时补充缓存。

TCMalloc的构造不仅于此，提供了ThreadCache和CentralCache可以解决小对象内存块的申请，但是对于大块内存Cache显然是不适合的。      

 TCMalloc将内存分为三类，如表4所示。



表4 TCMalloc的内存分离

| **对象** | **容量**     |
| -------- | ------------ |
| 小对象   | (0,256KB]    |
| 中对象   | (256KB, 1MB] |
| 大对象   | (1MB, +∞)    |



所以为了解决中对象和大对象的内存申请，TCMalloc依然有一个全局共享内存堆PageHeap，如图20所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133032054-ea888b96-0fb0-46ea-ac26-4c38abc2b66f.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_89%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

图 20 TCMalloc中的PageHeap



PageHeap也是一次系统调用从虚拟内存中申请的，PageHeap很明显是全局的，所以访问一定是要加锁。其作用是当CentralCache没有足够内存时会从PageHeap取，当CentralCache内存过多或者充足，则将低命中内存块退还PageHeap。如果Thread需要大对象申请超过的Cache容纳的内存块单元大小，也会直接从PageHeap获取。



#### TCMalloc模型相关基础结构

在了解TCMalloc的一些内部设计结构时，首要了解的是一些TCMalloc定义的基本名词Page、Span和Size Class。



##### Page

TCMalloc中的Page与之前章节介绍操作系统对虚拟内存管理的MMU定义的物理页有相似的定义，TCMalloc将虚拟内存空间划分为多份同等大小的Page，每个Page默认是8KB。

对于TCMalloc来说，虚拟内存空间的全部内存都按照Page的容量分成均等份，并且给每份Page标记了ID编号，如图21所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133095495-53138cb4-89b8-4833-ac41-7957a1c19354.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_82%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

图 21 TCMalloc将虚拟内存平均分层N份Page

将Page进行编号的好处是，可以根据任意内存的地址指针，进行固定算法偏移计算来算出所在的Page。



##### Span

多个连续的Page称之为是一个Span，其定义含义有操作系统的管理的页表相似，Page和Span的关系如图22所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133180209-09abdb85-cf15-40d4-8e7c-c9acb973e107.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_81%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 22 TCMalloc中Page与Span的关系

TCMalloc是以Span为单位向操作系统申请内存的。每个Span记录了第一个起始Page的编号Start，和一共有多少个连续Page的数量Length。

为了方便Span和Span之间的管理，Span集合是以双向链表的形式构建，如图23所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133255704-7c07cb59-d879-468f-a925-d3494454cb7d.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_82%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 23 TCMalloc中Span存储形式

#### 3.Size Class

参考表3-3所示，在256KB以内的小对象，TCMalloc会将这些小对象集合划分成多个内存刻度[[6\]](https://www.yuque.com/aceld/golang/qzyivn#_ftn6)，同属于一个刻度类别下的内存集合称之为属于一个Size Class。这与之前章节自定义实现的内存池，将Buf划分多个刻度的BufList类似。

每个Size Class都对应一个大小比如8字节、16字节、32字节等。在申请小对象内存的时候，TCMalloc会根据使用方申请的空间大小就近向上取最接近的一个Size Class的Span（由多个等空间的Page组成）内存块返回给使用方。

如果将Size Class、Span、Page用一张图来表示，则具体的抽象关系如图24所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133299709-8c33bad3-a31f-4844-b07c-ad54a0dc64d4.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_67%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 24 TCMalloc中Size Class、Page、Span的结构关系

接下来剖析一下ThreadCache、CentralCache、PageHeap的内存管理结构。

### 5.3 ThreadCache

在TCMalloc中每个线程都会有一份单独的缓存，就是ThreadCache。ThreadCache中对于每个Size Class都会有一个对应的FreeList，FreeList表示当前缓存中还有多少个空闲的内存可用，具体的结构布局如图25所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133403346-3d07b578-45df-41b1-880e-d1a591d106ff.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_97%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 25 TCMalloc中ThreadCache

使用方对于从TCMalloc申请的小对象，会直接从TreadCache获取，实则是从FreeList中返回一个空闲的对象，如果对应的Size Class刻度下已经没有空闲的Span可以被获取了，则ThreadCache会从CentralCache中获取。当使用方使用完内存之后，归还也是直接归还给当前的ThreadCache中对应刻度下的的FreeList中。

整条申请和归还的流程是不需要加锁的，因为ThreadCache为当前线程独享，但如果ThreadCache不够用，需要从CentralCache申请内存时，这个动作是需要加锁的。不同Thread之间的ThreadCache是以双向链表的结构进行关联，是为了方便TCMalloc统计和管理。

### 5.4 CentralCache

CentralCache是各个线程共用的，所以与CentralCache获取内存交互是需要加锁的。CentralCache缓存的Size Class和ThreadCache的一样，这些缓存都被放在CentralFreeList中，当ThreadCache中的某个Size Class刻度下的缓存小对象不够用，就会向CentralCache对应的Size Class刻度的CentralFreeList获取，同样的如果ThreadCache有多余的缓存对象也会退还给响应的CentralFreeList，流程和关系如图26所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133530033-bd9265dc-fd49-4a77-a845-f175ab317ea9.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_97%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 26 TCMalloc中CentralCache

CentralCache与PageHeap的角色关系与ThreadCache与CentralCache的角色关系相似，当CentralCache出现Span不足时，会从PageHeap申请Span，以及将不再使用的Span退还给PageHeap。

### 5.5 PageHeap

PageHeap是提供CentralCache的内存来源。PageHead与CentralCache不同的是CentralCache是与ThreadCache布局一模一样的缓存，主要是起到针对ThreadCache的一层二级缓存作用，且只支持小对象内存分配。而PageHeap则是针对CentralCache的三级缓存。弥补对于中对象内存和大对象内存的分配，PageHeap也是直接和操作系统虚拟内存衔接的一层缓存，当ThreadCache、CentralCache、PageHeap都找不到合适的Span，PageHeap则会调用操作系统内存申请系统调用函数来从虚拟内存的堆区中取出内存填充到PageHeap当中，具体的结构如图27所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133596465-fd16a3cc-256a-464c-a066-b896043a9f63.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_101%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 27 TCMalloc中PageHeap

PageHeap内部的Span管理，采用两种不同的方式，对于128个Page以内的Span申请，每个Page刻度都会用一个链表形式的缓存来存储。对于128个Page以上内存申请，PageHeap是以有序集合（C++标准库STL中的Std::Set容器）来存放。

### 5.6 TCMalloc的小对象分配

上述已经将TCMalloc的几种基础结构介绍了，接下来总结一下TCMalloc针对小对象、中对象和大对象的分配流程。小对象分配流程如图28所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133672724-0ac13b26-1623-444a-8c81-0b2120b2e2fa.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_80%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 28 TCMalloc小对象分配流程

小对象为占用内存小于等于256KB的内存，参考图中的流程，下面将介绍详细流程步骤：

（1）Thread用户线程应用逻辑申请内存，当前Thread访问对应的ThreadCache获取内存，此过程不需要加锁。

（2）ThreadCache的得到申请内存的SizeClass（一般向上取整，大于等于申请的内存大小），通过SizeClass索引去请求自身对应的FreeList。

（3）判断得到的FreeList是否为非空。

（4）如果FreeList非空，则表示目前有对应内存空间供Thread使用，得到FreeList第一个空闲Span返回给Thread用户逻辑，流程结束。

（5）如果FreeList为空，则表示目前没有对应SizeClass的空闲Span可使用，请求CentralCache并告知CentralCache具体的SizeClass。

（6）CentralCache收到请求后，加锁访问CentralFreeList，根据SizeClass进行索引找到对应的CentralFreeList。

（7）判断得到的CentralFreeList是否为非空。

（8）如果CentralFreeList非空，则表示目前有空闲的Span可使用。返回多个Span，将这些Span（除了第一个Span）放置ThreadCache的FreeList中，并且将第一个Span返回给Thread用户逻辑，流程结束。

（9）如果CentralFreeList为空，则表示目前没有可用是Span可使用，向PageHeap申请对应大小的Span。

（10）PageHeap得到CentralCache的申请，加锁请求对应的Page刻度的Span链表。

（11）PageHeap将得到的Span根据本次流程请求的SizeClass大小为刻度进行拆分，分成N份SizeClass大小的Span返回给CentralCache，如果有多余的Span则放回PageHeap对应Page的Span链表中。

（12）CentralCache得到对应的N个Span，添加至CentralFreeList中，跳转至第（8）步。

综上是TCMalloc一次申请小对象的全部详细流程，接下来分析中对象的分配流程。

### 5.7 TCMalloc的中对象分配

中对象为大于256KB且小于等于1MB的内存。对于中对象申请分配的流程TCMalloc与处理小对象分配有一定的区别。对于中对象分配，Thread不再按照小对象的流程路径向ThreadCache获取，而是直接从PageHeap获取，具体的流程如图29所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133803693-486e9b4a-ffb1-4932-a989-1df013b601c1.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_71%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 29 TCMalloc中对象分配流程

PageHeap将128个Page以内大小的Span定义为小Span，将128个Page以上大小的Span定义为大Span。由于一个Page为8KB，那么128个Page即为1MB，所以对于中对象的申请，PageHeap均是按照小Span的申请流程，具体如下：

（1）Thread用户逻辑层提交内存申请处理，如果本次申请内存超过256KB但不超过1MB则属于中对象申请。TCMalloc将直接向PageHeap发起申请Span请求。

（2）PageHeap接收到申请后需要判断本次申请是否属于小Span（128个Page以内），如果是，则走小Span，即中对象申请流程，如果不是，则进入大对象申请流程，下一节介绍。

（3）PageHeap根据申请的Span在小Span的链表中向上取整，得到最适应的第K个Page刻度的Span链表。

（4）得到第K个Page链表刻度后，将K作为起始点，向下遍历找到第一个非空链表，直至128个Page刻度位置，找到则停止，将停止处的非空Span链表作为提供此次返回的内存Span，将链表中的第一个Span取出。如果找不到非空链表，则当错本次申请为大Span申请，则进入大对象申请流程。

（5）假设本次获取到的Span由N个Page组成。PageHeap将N个Page的Span拆分成两个Span，其中一个为K个Page组成的Span，作为本次内存申请的返回，给到Thread，另一个为N-K个Page组成的Span，重新插入到N-K个Page对应的Span链表中。

综上是TCMalloc对于中对象分配的详细流程。

### 5.8 TCMalloc的大对象分配

对于超过128个Page（即1MB）的内存分配则为大对象分配流程。大对象分配与中对象分配情况类似，Thread绕过ThreadCache和CentralCache，直接向PageHeap获取。详细的分配流程如图30所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133987470-28f3feb2-8a9e-45be-a41b-596b1bd54e8d.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_81%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###### 图 30 TCMalloc大对象分配流程

进入大对象分配流程除了申请的Span大于128个Page之外，对于中对象分配如果找不到非空链表也会进入大对象分配流程，大对象分配的具体流程如下：

（1）Thread用户逻辑层提交内存申请处理，如果本次申请内存超过1MB则属于大对象申请。TCMalloc将直接向PageHeap发起申请Span      。

（2）PageHeap接收到申请后需要判断本次申请是否属于小Span（128个Page以内），如果是，则走小Span中对象申请流程（上一节已介绍），如果不是，则进入大对象申请流程。

（3）PageHeap根据Span的大小按照Page单元进行除法运算，向上取整，得到最接近Span的且大于Span的Page倍数K，此时的K应该是大于128。如果是从中对象流程分过来的（中对象申请流程可能没有非空链表提供Span），则K值应该小于128。

（4）搜索Large Span Set集合，找到不小于K个Page的最小Span（N个Page）。如果没有找到合适的Span，则说明PageHeap已经无法满足需求，则向操作系统虚拟内存的堆空间申请一堆内存，将申请到的内存安置在PageHeap的内存结构中，重新执行（3）步骤。

（5）将从Large Span Set集合得到的N个Page组成的Span拆分成两个Span，K个Page的Span直接返回给Thread用户逻辑，N-K个Span退还给PageHeap。其中如果N-K大于128则退还到Large Span Set集合中，如果N-K小于128，则退还到Page链表中。

综上是TCMalloc对于大对象分配的详细流程。







https://www.cnblogs.com/chenchen4396/p/17625471.html

https://mp.weixin.qq.com/s/MXkUunybffkF0CF8i6jIKA

https://www.cnblogs.com/jiujuan/p/13869547.html

https://zhuanlan.zhihu.com/p/29216091

https://mp.weixin.qq.com/s/wKqcAAU7aq-i34ZhW7NH9A
