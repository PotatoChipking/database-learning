# 标题：EvenDB: optimizing key-value storage for spatial locality

## 🍅摘要：

Applications of key-value (KV-)storage often exhibit high spatial locality, such as when many data items have identical composite key prefixes. This prevalent access pattern is underused by the ubiquitous LSM design underlying highthroughput KV-stores today. We present EvenDB, a general-purpose persistent KVstore optimized for spatially-local workloads. EvenDB combines spatial data partitioning with LSM-like batch I/O. It achieves high throughput, ensures consistency under multithreaded access, and reduces write amplification. In experiments with real-world data from a large analytics platform, EvenDB outperforms the state-of-the-art. E.g., on a 256GB production dataset, EvenDB ingests data 4.4× faster than RocksDB and reduces write amplification by nearly 4×. In traditional YCSB workloads lacking spatial locality, EvenDB is on par with RocksDB and significantly better than other open-source solutions we explored.

📅学习日期：2023-07-06
🍤作者：Gilad Eran,Bortnikov Edward,Braginsky Anastasia,Gottesman Yonatan,Hillel Eshcar,Keidar Idit,Moscovici Nurit,Shahout Rana
🍎出版年份：2020
✉️刊于：
☀️影响因子：

## 🏁研究背景：

kv存储的应用往往表现出很高的空间局部性，尤其是在许多数据项具有相同的复合key前缀时。

这种工作负载，在目前高吞吐量的kv存储引擎中，没有得到充分研究。

kv存储提供了简单的编程模型，数据作为有序的kv对集合，对外提供的API有随机写、随机读以及范围查询。

一种常见的设计模型是，使用复合键来表示属性的聚集。

在这种情况下，主属性即key前缀具有偏态分布。即通过复合键进行的访问呈现出空间局部性。key前缀的冷热区别引起key范围的冷热区别。

据数据表明，1%的应用触发94%的事件，其中，小于0.1%的应用覆盖了70%。

LSM Tree在此负载下的表现分析：

LSM tree在处理数据时，首先将写入聚集到文件中，而不是通过key范围来写入。（原地更新与RMW）。

然后通过后台compaction线程执行归并、排序，通过key将数据进行聚集。

结论：

该方法不是高空间局部性的理想方法。

原因：

1、热key范围被文件划分开来。

2、compaction开销过大-性能、写入放大等方面。写入放大尤其影响SSD的磨损，内存中对数据的聚集意味着compaction对数据的处理在数据冷热程度上是无差别的。

3、内存中的数据聚集需要额外通过WAL进行持久化保证。

compaction，这里为flush时，对key的处理，在mem层面是无差别的，以mem为组进行flush。
不利于冷热数据分离。

compaction时，将上层数据与下次key范围重叠的sst进行排序、合并。
然后重新写入到下层中。
但是处理下层重复的sst时，冷数据仍然需要进行重写操作。

## 🍚研究对象：

kv存储在高空间局部性负载下的针对设计。

设计目标：

提出一种适用于当今工作负载中出现的空间局部性的KV - store设计备选方案，同时不丧失LSM方法所带来的好处。

研究LSM tree在数据处理时忽略的空间局部性，利用高空间局部性的数据特征，提高数据空间放大、写入放大。

## 🍟实现方案：

设计原则：

1、访问语义与优化目标

支持并发访问，并确保强一致性，提供的读写、扫描操作为原子操作。

对于复合键的高空间局部性的负载进行设计。

最小化写入放大以减少磁盘磨损。

在sliding-local场景下追求高性能，即大多数活跃数据存放于内存中。

确保快速恢复与一致性状态。

2、设计选择

将B树的空间数据划分与LSM的优化I/O和快速访问相结合。

![img](img\EvenDB1.png)

chunks：

为利用空间局部性，将磁盘、内存中的数据都通过key进行划分。

数据结构组织成包含连续key范围的chunks的列表。

所有的chunks通过轻量级易失性元数据对象在内存中表示，用于在对象恢复时从磁盘中重建。

顺序i/o与chunk内部日志：

对于持久化，每个chunk都有一个文件表示形态，称为funk。

funk内部，采用lsm的顺序i/o方式。

分为两部分：一个SSTable和一个Log，新的更新追加到log中，log通过compaction合并到SSTable中。

LSM，处理数据时，以固定大小的文件来处理所有数据，以文件为单位对数据进行组织。

EvenDB，首先对数据进行划分，划分好的数据放入对应的文件，以数据为单位设计操作。
如此设计，从spatial-locality的角度分析：
LSM 在memtable大小内，对热数据有去重作用，随后热数据受冷数据影响，达到大小阈值随之写入下层。
EvenDB 由于对数据进行划分，将数据对应的文件进行处理。从而对热数据，空间放大更好，写入放大减少。

chunk级别缓存与内存中的compaction：

对应chunk级别的热数据，将chunk缓存到内存中，称为munk，与funk类似，数据分为有序的合并部分，和无序的新数据（等待compact）。

对于有缓存的chunk，数据不必一致追加写入到funk的log部分，而是通过内存compaction在munk部分进行合并，从而减少了空间放大与写入放大。

cache-优化读取路径，in-memory compaction-优化写入路径。

row cache与bloom filter：

raw cache对于chunks中，较少key为热数据的情况。
此时，若缓存整个块，空间浪费，所以缓存row。

EvenDB设计

![img](img\EvenDB2.png)

数据组织：

数据被划分成块，每个块都有一个连续的key范围。每个chunk的数据可以在磁盘中-funk，也可以在内存中-munk。

munk可以被替换、从磁盘中加载，chunk的元数据比这二者要小，通常存储于内存中。

有一个内存中的索引，将key映射到chunk中，该索引的更新是lazy的，即非及时更新。

funk：

由两部分组成：SSTable、Log。创建时，前者保存所有chunk的kv对，后者为空。

更新操作作用到funk时，顺序追加到log中，key重写时，旧值保留在SSTable中，新值写入到Log。

写入新值时，旧值保留在SSTable中，新值写入到log里。
当log越来越大时，查询效率下降，因此触发rebalance。

理解：

设计模式对rwm进行改动，由读取具体值-修改-写入改为，读取文件-写入。

如此修改，相较于copy on write写入放大减小。与LSM tree相比，log写入仍然会触发compaction，不过将compaction的放大粒度，由整体降低到key范围。

munk：

将kv对存储于一个基于数组的链表中。创建时，数组中的某些前缀被填充，并按照key进行排序。此时，链表中节点的顺序与数组中条目的顺序一致。

新的数据追加在此前缀之后，并写入到链表中，此时前缀定义的是key的写入顺序，而不再以key顺序。

因此，此时链表中的key顺序与数组中的key顺序部分不一致。

后续通过rebalance进行调整，确保整体有序，提高查询效率。

reorganization:

随着数据的增加、重写、删除，munk与funk需要通过reorganization进行数据整理，提供数据的有序，提高查询效率。

过程：

1、compactin

垃圾回收对于移除、重写的数据。

2、sorting

对key进行排序以提高查询效率。

3、splitting

过大的chunk进行分裂。

reorganization通过三个进程实现：munk rebalance、funk rebalance、split。

缓存：

缓存对于不同的chunk有两种方式：

对于热数据较多的chunk，生成munk。

对于热数据较少的chunk，通过row cache进行缓存，并利用bloom filter对其funk log进行过滤。

并发控制与原子扫描：

get操作是无锁的，在非同步的情况下执行。为提供scan的原子性，scan时需要与put进行同步，即等待相关put的执行，而put不需要等待scan。

即通过多版本对数据进行标识。

通过系统层面的global version -- GV，来支持原子扫描。

一次scan通过对GV进行抓取和增量操作，生成与当前GV对应的快照。

对于旧版本回收，通过pending operation（PO）数组追踪活跃的scan操作。

该数组中，每个活动线程有一个条目，也用于将put与scan同步（scan等待put）。

put与scan冲突：

定义scan时，需要等待所有对该版本的put完成，即，对于put版本小于scan获取版本的操作，都需要等待。

问题在于，put操作是非原子的。

例：put从GV处获取了版本7，在将值写入到chunk前stall，此时，scan获取GV，将其增加到8，由于put操作处于stall状态，有可能读不到最新的值。

解决：

put在处理前声明其作用的key，并且扫描等待相关的挂起的put完成。

EvenDB操作：

get、put操作首先都会通过lookup定位到目标chunk。即内存中缓存的key--chunk的索引。

get：

对于有munk的chunk：首先对优速前缀进行二分查找，然后按需遍历链表。

对于无munk的chunk：首先读取row cache，若不存在，查询bloom filter确定key是否在目标funk的log中，如果存在进行搜索，若不存在查询SSTable。

put：

在定位到chunk后，获取rebalanceLock，以确保未经过rebalance。

然后在PO中注册要进行修改的key，读取GV并将版本号写入到PO中其对应的条目中。

然后将新数据写入到funk的log中，然后写入到munk中。

对于没有munk的chunk，检查row cache并对其进行更新。

最后，撤销PO，释放rebalanceLock。

scan：

首先其从PO中获取版本的意图，以向正在进行的rebalance发送信号，避免删除其所需的版本。

然后获取并增加GV以记录自身快照事件。将作用的key范围、获取的版本写入PO。

然后等待相关的put完成，之后从所有相关的的chunk中收集相关的数据。

收集时，对于有munk的chunk。从中读取，否则读取SSTable、log中的数据，将收集的多版本的数据进行合并。

最后撤销PO中的条目。

rebalance：

munk rebalance：

改善数据组织，移除多余数据，排序数据。

由容量阈值触发，funk的阈值比munk高，以使得较多的rebalance在内存中处理。

创建一个新的munk，而不是进行原地重组，以减少对并发访问的影响。（B+树--copy on write 树。）

处理完成后，将chunk的指针更改到新的munk。通过PO进行迭代，收集scan的最小版本号。

funk rebalance：

创建一个新的SSTable、log对，如果chunk含有munk，则进行munk的rebalance。

然后将munk写入到新的SSTable中，此时log为空。

否则，将旧的SSTable与log进行合并，写入到新的SSTable中。

合并过程开销较大，因此不阻塞put处理，合并完成之后：

1、获取rebalanceLock。

2、设置新的log接收put操作。

3、修改chunk的funk指针。

4、释放rebalanceLock。

split：

如果munk的rebalance使得容量超出阈值，触发split，与chunk内部的rebalance不同。

split修改chunk列表--key to chunk的索引等。

过程：

1、在内存中执行，首相将munk拆分为两个有序压缩的两部分。然后创建两个新的chunk，每个引用一个munk，此时暂时共享一个funk。不可修改。

然后将新的chunk插入到chunk列表中，取代旧的chunk。更新索引。

2、此时，put可作用到新chunk中，但是无法进行rebalance。旧的chunk此时仍然可提供读取服务。

flush：

EvenDB提供两种持久化级别：同步、异步。

同步时，更新被持久化到磁盘之后再返回到使用者。

异步时，间断调用fsync进行持久化，减少写入延迟，提高吞吐量。



## 🍒总结评估：

Referred in [区块链存储优化学习](zotero://note/u/2Y9JV3V2/?ignore=1&line=80)