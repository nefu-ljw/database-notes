## CMU15-445 Lecture #05 Buffer Pools

课程链接：[15-445/645 Database Systems (Fall 2020)](https://15445.courses.cs.cmu.edu/fall2020)

本文由 [nefu-ljw](https://github.com/nefu-ljw) 翻译于Notes：[https://15445.courses.cs.cmu.edu/fall2020/notes/05-bufferpool.pdf](https://15445.courses.cs.cmu.edu/fall2020/notes/05-bufferpool.pdf)

所有Notes已同步更新于我的github仓库：[https://github.com/nefu-ljw/database-notes](https://github.com/nefu-ljw/database-notes)

**目录**

  * [1\. Introduction](#1-introduction)
  * [2\. Locks vs\. Latches](#2-locks-vs-latches)
  * [3\. Buffer Pool](#3-buffer-pool)
  * [4\. Buffer Pool Optimizations](#4-buffer-pool-optimizations)
  * [5\. OS Page Cache](#5-os-page-cache)
  * [6\. Buffer Replacement Policies](#6-buffer-replacement-policies)
  * [7\. Other Memory Pools](#7-other-memory-pools)

### 1. Introduction

DBMS负责管理其内存以及从磁盘来回移动数据。因为在大多数情况下，数据不能直接在磁盘上操作，所以任何数据库都必须能够有效地将其磁盘上表示为文件的数据移动到内存中，以便可以使用。这种交互的图表如图1所示。DBMS面临的一个障碍是将移动数据的速度降至最低的问题。理想情况下，它应该“看起来”好像所有的数据都已经在内存中了。

![image-20211126154143798](https://i.loli.net/2021/11/26/kuZm4d5v8rxfejK.png)

考虑这个问题的另一种方式是从空间和时间控制的角度。

空间控制(Spatial Control)指的是在磁盘上物理写入页面的位置。空间控制的目标是使经常在一起使用的页面在物理上尽可能靠近磁盘。

时间控制(Temporal Control)指的是何时将页面读入内存以及何时将其写入磁盘。时间控制旨在最大限度地减少不得不从磁盘读取数据的停顿次数。

### 2. Locks vs. Latches

在讨论DBMS如何保护其内部元素时，我们需要区分锁(locks)和锁存器(latches)。

**Locks**: 锁是保护数据库(例如，元组、表、数据库)的内容不受其他事务影响的更高级别的逻辑原语。事务将在其整个持续时间内保持锁定。当运行查询时，数据库系统可以向用户暴露哪些锁被持有。锁需要能够回滚更改。

**Latches**: 锁存器是DBMS用于其内部数据结构(例如，哈希表、存储器区域)中的临界区的低级保护原语。仅在进行操作的持续时间内保持锁存。锁存器不需要能够回滚更改。

### 3. Buffer Pool

**缓冲池是从磁盘读取的页面的内存缓存。它本质上是在数据库内部分配的一个大内存区域，用于存储从磁盘获取的页面。**

缓冲池的内存区域，组织为固定大小的页面数组。每个数组条目都称为一帧(**frame**)。**当DBMS请求页面时，会将一个完全相同的副本放入缓冲池的一帧中。然后，当请求页面时，数据库系统可以首先搜索缓冲池。如果找不到该页，则系统将从磁盘获取该页的副本。**有关缓冲池的内存组织图，请参见图2。

![image-20211126160911872](https://i.loli.net/2021/11/26/cVjOtpBax7dGhZm.png)

**Buffer Pool Meta-data**

缓冲池必须维护某些元数据才能有效和正确地使用。

首先，页表(**page table**)是内存中的哈希表，跟踪当前内存中的页面。**页表将页面ID映射到缓冲池中的帧位置。**因为缓冲池中的页面顺序不一定反映磁盘上的顺序，所以这个额外的间接层允许标识缓冲池中的页面位置。请注意，不要将页表与页目录(**page directory**)混淆，**页目录是从页面ID映射到数据库文件中的页位置。**

页表还维护每页的附加元数据、脏标志和固定/引用计数器。

脏标志(**dirty-flag**)是由线程在修改页面时设置的。这向存储管理器指示该页必须写回磁盘。

固定/引用计数器(**Pin/Reference Counter**)跟踪当前正在访问该页面(读取或修改该页面)的线程数。线程在访问页面之前必须递增计数器。**如果页面的计数大于零，则不允许存储管理器将该页面从内存中逐出。**

**Memory Allocation Policies**

数据库中的内存根据两个策略分配给缓冲池。

全局策略(global policies)处理DBMS应该做出的决策，以使正在执行的整个工作负载受益。它会考虑所有活动事务，以找到分配内存的最佳决策。

另一种选择是本地策略(local policies)做出决策，使单个查询或事务运行得更快，即使这对整个工作负载不好。本地策略将帧分配给特定事务，而不考虑并发事务的行为。

大多数系统同时使用全局视角和本地视角。

### 4. Buffer Pool Optimizations

有许多方法可以优化缓冲池以使其适应应用程序的工作负载。

**Multiple Buffer Pools**

DBMS可以为不同的目的维护多个缓冲池(即每个数据库的缓冲池、每个页面类型的缓冲池)。然后，每个缓冲池都可以采用为其中存储的数据量身定做的本地策略。此方法有助于减少锁存争用并改善局部性。

将所需页面映射到缓冲池的两种方法是对象ID和散列。

对象ID(Object IDs)涉及扩展记录ID(record IDs)以包括有关每个缓冲池管理的数据库对象的元数据。然后，通过对象标识符，可以维护从对象到特定缓冲池的映射。

另一种方法是DBMS散列(hashing)页面ID以选择要访问的缓冲池。

**Pre-fetching**

DBMS还可以根据查询计划预取页面进行优化。然后，在处理第一组页面时，可以将第二组页面预取到缓冲池中。当数据库管理系统(DBMS)连续访问多个页面时，通常使用这种方法。

**Scan Sharing**

查询游标可以重用从存储或操作符计算中检索到的数据。这允许多个查询附加到一个扫描表的游标上。如果一个查询开始扫描，并且已经有另一个查询这么做过了，那么DBMS将附加到第二个查询的游标上。DBMS跟踪第二个查询与第一个查询的连接位置，以便在到达数据结构的末尾时完成扫描。

**Buffer Pool Bypass**

顺序扫描操作符不会将获取的页面存储在缓冲池中，以避免开销。相反，内存对于正在运行的查询是本地的。如果操作符需要读取磁盘上连续的大量页序列，则这种方法很有效。缓冲池旁路(Bypass)也可以用于临时数据(排序、连接)。

### 5. OS Page Cache

大多数磁盘操作都通过操作系统API进行。除非明确告知，否则操作系统维护自己的文件系统缓存。

大多数DBMS使用直接I/O来绕过(bypass)操作系统的缓存，以避免页面的冗余拷贝和不得不管理不同的驱逐策略。

**Postgres**是一个使用操作系统的页面缓存的数据库系统的例子。

### 6. Buffer Replacement Policies

当DBMS需要释放一个帧来为一个新页腾出空间时，它必须决定从缓冲池中驱逐哪个页。

替换策略是DBMS实现的一种算法，它可以在缓冲池需要空间时决定从缓冲池中驱逐哪些页面。

替换策略的实现目标是提高正确性、准确性、速度和(降低)元数据开销。

**Least Recently Used (LRU)**

最近最少使用的替换策略维护了每个页面最后访问时间的时间戳。这个时间戳可以存储在一个单独的数据结构中，比如一个队列，以便进行排序并提高效率。DBMS选择删除时间戳最旧的页面。此外，页面按照排序的顺序保持，以减少收回时的排序时间。

**CLOCK**

CLOCK策略近似于LRU，但不需要每个页面单独的时间戳。在CLOCK策略中，每页都有一个参考位。当访问页面时，设置为1。

为了直观地看到这一点，可以使用“时钟指针”将页面组织在一个循环缓冲区中。扫描时检查页面的位是否设置为1。如果是，设置为0，如果不是，驱逐它。这样，时钟指针就能记住两次驱逐之间的位置。

**Alternatives**

LRU和CLOCK策略存在许多问题。

![image-20211126164044698](https://i.loli.net/2021/11/26/FN8h9l6ZLpuOj5m.png)

（图3：时钟替换策略的可视化。第1页被引用并设置为1。当时钟指针扫描时，它将第1页的参考位设置为0，并驱逐第5页）

也就是说，LRU和CLOCK很容易受到顺序泛洪(sequential flooding)的影响，即由于顺序扫描，缓冲池的内容被破坏。由于顺序扫描读取每一页，所读取的页的时间戳可能不能反映我们真正想要的页。换句话说，最近使用的页面实际上是最不需要的页面。

有三种解决方案可以解决LRU和CLOCK策略的缺点。

一种解决方案是LRU-K，它将最后K个引用的历史作为时间戳进行跟踪，并计算后续访问之间的间隔。此历史记录用于预测下一次访问页面的时间。

另一个优化是对每个查询进行本地化(localization)。DBMS根据每个事务/查询选择要逐出的页面。这最大限度地减少了每次查询对缓冲池的污染。

最后，优先级提示(priority hints)，允许事务在查询执行期间根据每个页面的上下文告诉缓冲池页面是否重要。

**Dirty Pages**

有两种方法可以处理含有脏位的页面。最快的选择是删除缓冲池中任何不脏的页面。一种较慢的方法是将脏页写回磁盘，以确保将其更改持久化。

这两种方法说明了快速收回与将来不会再次读取的脏写入页之间的权衡。

一种避免不必要地写页面的方法是背景写入(background writing)。通过背景写入，DBMS可以定期遍历页表并将脏页写入磁盘。当安全写入脏页时，DBMS可以驱逐该页或只是取消脏标志的设置。

### 7. Other Memory Pools

除了元组和索引之外，DBMS还需要内存。根据实施情况，这些其他内存池可能并不总是由磁盘支持。

- Sorting + Join Buffers（排序+连接缓冲区）
- Query Caches（查询缓存）
- Maintenance Buffers（维护缓冲区）
- Log Buffers（日志缓冲区）
- Dictionary Caches（字典缓存）
