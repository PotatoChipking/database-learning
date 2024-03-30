# 标题：PACTree: A High Performance Persistent Range Index Using PAC Guidelines

## 🍅摘要：

Non-Volatile Memory (NVM), which provides relatively fast and byte-addressable persistence, is now commercially available. However, we cannot equate a real NVM with a slow DRAM, as it is much more complicated than we expect. In this work, we revisit and analyze both NVM and NVMspecific persistent memory indexes. We find that there is still a lot of room for improvement if we consider NVM hardware, its software stack, persistent index design, and concurrency control. Based on our analysis, we propose Packed Asynchronous Concurrency (PAC) guidelines for designing high-performance persistent index structures. The key idea behind the guidelines is to 1) access NVM hardware in a packed manner to minimize its bandwidth utilization and 2) exploit asynchronous concurrency control to decouple the long NVM latency from the critical path of the index.

📅学习日期：2023-05-09
🍤作者：Kim Wook-Hee,Krishnan R. Madhava,Fu Xinwei,Kashyap Sanidhya,Min Changwoo
🍎出版年份：2021
✉️刊于：
☀️影响因子：

## 🏁研究背景：

NVM存储设备的性能很好，随着其大规模商业化，论文研究结论在考虑到NVM的硬件、软件栈、以及并发控制等问题时，仍存在很大优化空间。

基于此分析，论文提出打包异步并发准则，并围绕准则设计了新的索引结构。

像intel傲腾DC PM此类可按字节寻址的非易失性存储器，正在打破传统的存储与存储二分法。

## 🍚研究对象：

1、详细分析了NVM的特性及工作原理，给出了NVM硬件在设计持久化索引时需要考虑的关键性能属性。

2、基于特性提出一系列设计高性能持久索引设计准则，强调如何利用系统元件。

3、根据提出的设计准则设计了新型持久化范围索引PACTREE。

## 🍟实现方案：

对索引结构进行分析，B+树与trie进行对比，结论：B+树消耗更多的NVM带宽。

PACTREE是一个持久、混合范围索引，由基于trie的内部节点（search layer）与B+树风格的叶子节点组成（data layer）。

搜索层使用优化的ART持久版本（PDL-ART）。

数据层使用带插槽的节点结构。

搜索层与数据层的解耦是为了充分利用PAC设计准则--是执行打包NVM访问的重要方面。

同时利用异步并发减少由于SMO造成的阻塞时间，从而实现高可扩展性。

![PACTree](D:\文件库\研究生\Learner\笔记\PACTree.png)

搜索层：

ART是前缀树的变体，对减少空间开销进行了优化，同样可对节点进行压缩。基于此提出Persistent Durable Linearizable ART（PDL-ART）。有几点改变：

1、持久线性化与恢复能力。

ART提供的读操作可作用到未持久化的写操作上，与持久线性化冲突。

通过乐观版本锁，读事务只可在写操作释放后才可访问（写未结束版本为奇数）。阻塞读操作保证了持久线性化。

2、无日志崩溃一致性。

版本锁提供了独占的写访问，因此执行顺序与存储属性一致，考虑两种情况持久化存储指令的执行顺序（与存储顺序一致，类似日志。）。

1、指令只修改单个缓存行：

在这种情况下，无论崩溃发生在何处，持久化顺序都等价于程序执行顺序。这是因为x86架构从不对存储指令进行重新排序，而Cache行是Optane NVM (见图1)中WPQ的持久化单元。

2、指令修改多条缓存行：

首先持久化除了元数据之外的所有指令，然后边写并立即持久化元数据。

使用ROWEX（Read-Optimized Write-Exclusive）一种同步优化技术，可以保证独占写和非阻塞读。在ROWEX中，每个节点的锁由写入者在修改一个ART节点之前获得，并且对该节点的更新始终是原子的。

结构：

以key的部分作为内部节点，对数据节点进行定位，当数据节点创建时，定义其最小key作为Anchor key。

每个数据节点的范围为两个相邻anchor key的key范围。

当split发生时，只有后面一半节点被移动到新节点，从而保证anchor节点不变。

以trie结构所谓搜索层因为该结构消耗的NVM带宽更少。



数据层：

结构：是类B+tree节点组成的双链表结构，称为数据节点data node。每个包含多个kv对：

![PACTree2](D:\文件库\研究生\Learner\笔记\PACTree2.png)

插槽式列表结构支持插入、删除、更新操作--避免昂贵的持久化内存分配。

关于指纹阵列-一个有序数组，用于对数据节点中的key进行有序索引，用于快速扫描。每次使用时对数组的版本与数据节点的版本进行比较，确定其有效性。

一个8字节的bitmap用来标记数据节点内插槽的有效性。

一个8字节的版本锁，包括4字节的生成ID和4字节的版本号。

查找操作：

分为两阶段：1、遍历搜索层。2、遍历数据层。

1、遍历搜索层，首先比较目标key与数据节点的部分anchor key。

如果搜索层与数据层不同步，会到达相邻节点，称为jump节点，否则会直接到达目标节点。

2、遍历数据节点，由于可能两层不匹配，所以无法确定jump节点便是key所在的目标key，所以先使用anchor key对jump节点的key范围进行检查。

如果目标key小于jump节点的anchor key，则遍历到左邻居数据节点，否则到右邻居节点。

到达目标数据节点后首先检查该节点的指纹数据的版本，确定其是否可用，然后从该数组中找到目标key进行全key匹配。

然后检查版本号（乐观版本锁），确定该数据是否有效。

并发控制：

搜索层和数据层都使用乐观锁的版本实现版。（操作开始、提交时检测版本号是否改动，为奇数时也重试操作。）

两个属性：1、读操作部不改变版本，没有写入。2、与HTM不同，基于锁的方法不受数据大小/占用空间的影响。

异步SMO：

插入和删除操作会触发结构变动，当作用的数据节点以存满时会触发拆分操作。同时在搜索层增加一个新节点。

删除操作可能触发合并操作，并从搜索层删除旧的数据节点。

为了避免SMO称为可扩展性的瓶颈（阻塞事务处理），提出一种异步SMO，将开销较大的搜索层更新从SMO中解耦出来。

当触发合并/拆分时，在不立即更新搜索层情况下，将被拆分/合并的节点记录到SMO日志中。

通过后台线程对日志条目的重放，对搜索层进行更新。

问题：

搜索层与数据层的不匹配需要处理。

解决：

提出一种短暂的不一致性容忍设计，来保证在不匹配的情况下可以进行正确操作。

不匹配的搜索层会指向一个错误但是相邻的数据节点，由于数据层是双链表结构，仍然可以通过遍历数据层来比较数据节点的Anchor key与搜索key来进行操作。

崩溃一致性：

不通过日志技术实现，而是利用数据节点中的插槽结构列表与bitmap来实现。

使用bitmap作为写操作的持久性点，由SMO保证崩溃一致性。

选择持久性：

保证数据节点崩溃一致性不需要持久化所有的信息--排序数组，以有序的方式存储数据节点中的key的索引，不对其保证一致性（避免持久化开销与缓存失效），在必要时按需生成。

## 🍒总结评估：

Referred in [区块链存储优化学习/数据库存储索引](zotero://note/u/2Y9JV3V2/?ignore=1&line=56)