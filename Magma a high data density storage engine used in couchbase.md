# 标题：Magma: a high data density storage engine used in couchbase

## 🍅摘要：

We present Magma, a write-optimized high data density key-value storage engine used in the Couchbase NoSQL distributed document database. Today’s write-heavy data-intensive applications like ad-serving, internet-of-things, messaging, and online gaming, generate massive amounts of data. As a result, the requirement for storing and retrieving large volumes of data has grown rapidly. Distributed databases that can scale out horizontally by adding more nodes can be used to serve the requirements of these internetscale applications. To maintain a reasonable cost of ownership, we need to improve storage eciency in handling large data volumes per node, such that we don’t have to rely on adding more nodes. Our current generation storage engine, Couchstore is based on a log-structured append-only copy-on-write B+Tree architecture. To make substantial improvements to support higher data density and write throughput, we needed a storage engine architecture that lowers write amplication and avoids compaction operations that rewrite the whole database les periodically.

📅学习日期：2023-05-24
🍤作者：Lakshman Sarath,Gupta Apaar,Suri Rohan,Lashley Scott,Liang John,Duvuru Srinath,Mayuram Ravi
🍎出版年份：2022
✉️刊于：Proceedings of the VLDB Endowment
☀️影响因子：

## 🏁研究背景：

couchbase是一款分布式文档数据库，面对写密集、数据密集的应用，存储和取出大数据的需求越来越大。要求数据库有更大的存储容量、更高的事务吞吐量。

需要对原有的存储引擎进行升级以提供更好的性能。分布式数据库虽然可以通过增加节点来提高容量，但是为了保持合理的成本，也需要提高每个节点处理大数据量的存储效率，这样就不必依赖增加更多的节点。

原有引擎采用的是COW-B+树，在数据密度高的情况下存在一些问题：

1、写入较慢，在Copy-on-writeB+树上进行写类似于read-modify-write，需要随机读i/o。数据密度变高时，读会面临大量的缓存miss（MPT使用的levelDB也有此问题）。

2、合并挑战，当数据库碎片化时，需要通过compaction来限制空间放大。合并期间进行的操作需要在合并完成后进行重放，然后将新的操作切换到新产生的树上来执行。

## 🍚研究对象：

高数据密度、 value较大的场景下LSM-tree结构的实现与优化。

设计目标：

1、最小化写放大，为实现高事务写吞吐量，由于存储设备i/o带宽的有限，需要降低写放大。

2、可扩展并发合并，原本存储在触发合并时，开销很高，可能会对期间的操作产生影响，实现多线程更小的并发压缩，可以增量的执行空间回收是必需的。

3、对SSD进行优化，利用顺序读写i/o访问模式来利用SSD的全部贷款，随机i/o仅在点查找时发生，还需要利用i/o并发以利用NVMe提供的更高IOPS。

4、低内存占用，为支持每个节点较大的数据库规模，需要对数据库进行优化来减少内存占用，随着数据密度的增加减少读写缓存。

## 🍟实现方案：

**couchbase架构**：

![Magma](D:\文件库\研究生\Learner\笔记\Magma.png)



为提高写入性能和每个节点的数据密度设计。用户平均文档大小在1-16KB之间。

核心思想是将索引和文档分开存储以最小化写放大，也开发了可扩展的增量compaction方法限制空间放大。

**COW-B+树**：

![Magma2](D:\文件库\研究生\Learner\笔记\Magma2.png)

图中存储是追加形式，每次更改以后根遍历的顺序写入。

写入时是read-modify-write模式，当某个记录需要更改时，通过根页对树进行遍历，导航到非叶子节点页面，从页面中读取记录所在的叶页面。

在内存中对 叶页进行复制，然后修改，最后写入到文件中，由于叶子页面改动，指向叶子页面的非叶子节点也需要改动，自下而上对所有相关的节点页面进行修改重写。

原本的页面变为stale，后面通过compaction进行删除。控制空间放大。

**compaction**：

在后备线程中操作，获取当前的B+树根偏移量，打开一个B+树iterator，打开一个新的DB文件执行装载B+树操作重新构建一个B+树。

在构建时也有新的写入操作，在compaction结束后对这些新的操作进行重放。

重放结束后，将操作都转移到新建的DB文件中，删除旧的文件。

**MAGMA架构**：

![Magama3](D:\文件库\研究生\Learner\笔记\Magama3.png)

**1、写缓存**：

内存组件，用来缓存kv对，向持久存储大量顺序写入数据。也可用于查找，通过无锁跳表实现，大小固定，整体分为两个跳表：可变跳表、不可变跳表。达到限定大小时会flush到SSD上的key index和log-structure object storage。

**2、WAL**：

追加日志，用于对写入kv对的持久化，kv对在写入写缓存的同时也会写入到WAL。

只有预写日志返回fsync时，writeAPI才会返回。在写缓存flush到磁盘后，WAL会进行删除。

**3、LSM Tree Index**：

存储在log-structure object store中的文档的索引以LSMTree的结构进行组织。

以文档的key、seqno、元数据大小为kv对存储。使用准确度为99%的bloomfilter进行筛选。

对于文档的读取操作，首先查找LSMtree得到文档的seqno，用其从log-structure object store中读取文档数据。

**4、log-structure object store**：

通过将文档组织成一个仅追加的分段日志的形式来实现持久化。

维护了一个索引，用于通过seqno来访问文档，也允许进行范围查询。实际上为文档数据库提供了一个变更日志。

**5、Index Block Cache**：

以LRU为读缓存经常使用的LSM index和log-structure object store的索引，只缓存索引，不缓存文档。Couchbase维护vbucket级别的文档缓存，文档级别比块级别的缓存效率更高。理解为块中有许多数据，只有部分数据需要缓存。

**索引与数据分离**：

存储kv索引数据结构像B+树、LSM树，以有序的方式对key进行组织。

为维护数据有序需要对数据进行重组织，重组织的代价在存储的value较大时开销很大，严重浪费SSD带宽。

产生的写放大是维护有序的成本。

在value较大的情况下，如果数据可以单独存储在一个离散的log-structure object store中，可以避免这种写放大。

但是仍然需要维护一个索引来访问数据。

论文在此基础上提出将索引数据结构从文档数据存储中分离出来单独组织。

使用LSM-tree组织索引，索引包括记录的key、文档seqno、文档大小。其中文档seqno用于从log-structure object store获取文档数据。

此外通过前缀压缩进一步减小索引中存储的key大小。

灵感来自Wisckey。

**读写操作**：

通过说明存储引擎中的读和写操作路径，可以了解存储引擎设计中各个组件之间的交互。

查找：

首先查找内存中的write cache--可变、不可变跳表，若不存在时查找LSM Tree Index来获取seqno。

在LSM Tree中进行查找时通过BloomFilter对SSTable进行过滤，对于选择的SSTable从其中的B+树页面来读取key记录（内部索引）。

读取B+树时利用Index Block Cache以减少I/O。

若key存在，获取seqno，从log-structure object store中定位文档数据。

进行定位时也从B+树索引中获取文档所在的block的位置。这个索引也是缓存在Index Block Cache中的，访问时会根据LRU对缓存项进行调整。

写入：

文档的写入、修改操作即对于key的插入、更新、删除。

首先写入write-cache和WAL，当大小超过阈值时触发flush，写入到LSM-tree index和log-structure object store中。

flush时，内存kv被转化为LSM Tree Index的SSTable中的key索引和log-structure object store中的文档，文档被追加到其尾部。

之后，后台进行WAL的删除。

**changelog**：

changelog read操作读取从startSeqno到endSeqno的文档版本流

**日志结构对象存储**：

![Magama4](D:\文件库\研究生\Learner\笔记\Magama4.png)

log-structure object store由多个日志段文件组成，这些日志段被组织为一个连续增长的日志，尾日志段接收传入的写操作。

每个日志段log segment维护一个COW B+树索引，通过seqno定位文档。

在写入触发flush时，write-cache中的数据转化然后通过后台flush线程将修改数据追加到尾日志段。

追加的文档修改数据被组织为4KB的block，每个文档修改数据由唯一seqno标识。

flush也会将每个数据库的offset添加到B+树索引中。

理解为offset=页号，seqno=数据号。组织成B+树的形式。

尾段日志写满时，修改为不可变，新建一个尾日志用于追加数据。

文档版本按照seqno顺序追加，在其上建立的B+树索引也是一直追加写入，避免了read-modify-write操作。叶子节点一直追加，内部节点可能需要改变。在每一层维护一个尾页和block offset，叶子数量足够时会写入。

B+树：

由8byte的 seqno和block offset组成。日志段才内存中维护一个按段起始seqno排序的内存数组，类似于bitmap。查找时通过这个数组确定指定的文档在不在。

理解为用于判断缓存中是否存储。

**垃圾回收**：

文档一直追加写入，文档数据的旧版本以及删除的文档需要通过compaction进行垃圾回收，控制空间放大。

根据追加写入特性，新文档写入到尾部，旧文档在log-structure object store的头部。

通过compaction进行空间回收，所以需要确定触发条件、文档活跃的标准。

触发条件：

定义fragmentation为旧文档占所有数据的比例。即，当fragmentation超过设定阈值触发compaction。

compaction：

选取几个日志段，将活跃的文档复制到新的日志段文件中，替换原本的日志文件。（与COW B+树类似。）。

问题：活跃文档判断标准。设计一种方法估计日志段中的fragmentation，并检查compaction过程中文档版本的有效性。

判断活跃文档：

用文档的key在LSM Tree Index中进行查找，判断查到的seqno是否与当前一致，若一致则活跃。进行compaction时需要保留。

方案：

估计fragmentation key idea：为旧文档的seqno和在对应日志段中占有的大小维护一个逻辑有序的delete list。

如果从这个列表中取所有大小的和，便可以计算出每个日志段中旧文档占据的大小，从而计算出fragmentation。

默认为50%，即放大为2.

进行compaction时可以根据delete list对log segment中对应的文档进行删除（seqno）。

为了最小化写放大，compaction时需要选择fragmentation最高的日志段。

问题：delete list的生成。

解决：文档的活跃由LSM Tree Index判断，而其在进行compaction时，旧版本、删除的kv会被丢弃。

利用这个步骤来生成stale 文档seqno列表。在LSM Tree Index进行compaction时实现一个回调函数，接收被丢弃的seqno和大小，用于填充delete list。

![Magama5](D:\文件库\研究生\Learner\笔记\Magama5.png)

问题：delete list的维护开销较大。

为每一个log segment维护一个bitmap，每个seqno用一个bit标识。但是在数据密集场景下开销仍然过大。

Magma通过LSM来存储delete list，以8-byte seqno为key，以4-byte size为value。

将其排列在log-structure storage 存储的log segment的上方。

![Magama6](D:\文件库\研究生\Learner\笔记\Magama6.png)

在此之上通过设置fragmentation阈值来触发compaction。

**崩溃一致性**：

未提交的或提交未写入的事务的崩溃一致性由WAL保证。

steal/no-force，steal允许未提交事务写入磁盘undo，force强制提交事务写入磁盘redo。

Magma维护一个元数据文件，存储实时SSTable和基于seqno的日志结果对象存储文件的时间点快照。

周期性的运行一个检查点处理，通过当前活跃的LSM Tree Index和 Log-structure store中的文件来创建新的元数据文件。默认10分钟。

崩溃时，通过读取最新的元数据对LSM Tree Index和 Log-structure store进行重建。

LSM Tree Index在compaction时，写delete list时崩溃，需要确保delete list的快照元数据在key索引快照元数据持久化之前进行持久化。否则导致写的未持久的delete list丢失，永远不会回收。

当delete list持久化，但是compaction崩溃时，快照恢复后会再次生成一遍delete list。

## 🍒总结评估：

Referred in [区块链存储优化学习/数据库存储索引](zotero://note/u/2Y9JV3V2/?ignore=1&line=57)