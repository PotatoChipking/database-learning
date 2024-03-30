# 标题：LSM-based Storage Techniques: A Survey

## 🍅摘要：

Recently, the Log-Structured Merge-tree (LSMtree) has been widely adopted for use in the storage layer of modern NoSQL systems. Because of this, there have been a large number of research efforts, from both the database community and the operating systems community, that try to improve various aspects of LSM-trees. In this paper, we provide a survey of recent research efforts on LSM-trees so that readers can learn the state-of-the-art in LSM-based storage techniques. We provide a general taxonomy to classify the literature of LSM-trees, survey the efforts in detail, and discuss their strengths and trade-offs. We further survey several representative LSM-based open-source NoSQL systems and discuss some potential future research directions resulting from the survey.

📅学习日期：2023-05-15
🍤作者：Luo Chen,Carey Michael J.
🍎出版年份：2020
✉️刊于：The VLDB Journal
☀️影响因子：

## 🏁研究背景：

LSM-tree由于其性能优势，更贴合现代硬件、软件需求，被广泛用于NoSQL系统。

因此，对于LSM-tree的研究和实现也有很多，这些研究的侧重点不同，优化性能也不同。

论文对这些优化进行分类，分析了几个具有代表性的实现方案，并根据分析提出了未来的改进方向。

## 🍚研究对象：

1、对LSM-tree的历史工作进行总结，对其原理进行分析。

2、对近来工作进行分析，对优化研究进行分类。

3、对代表性的实现方案进行总结。

4、对未来优化方向进行总结。

## 🍟实现方案：

1、历史工作：

索引结构的两种更新方式如图1.
    1、in-place：原地更新，在更新时使用新值修改旧值--------读优化，牺牲写性能，索引页可能由于更新被分裂，降低空间利用率。
    2、out-of-place：异地更新，将新值写入新的位置而不是进行修改--------写优化（可以利用顺序io来处理写），可以简化恢复处理（不覆盖旧值），但是读性能被牺牲，此外结构需要额外的重组织来提高存储和查询效率。

存在的几个问题：

1、追加型日志记录将相关条目在存储中散发，查询性能较差。

2、一份条目存储多个历史数据，空间利用率低。

96年LSM-tree论文解决了这些问题--------设计了一个合并过程，整合到该结构中。提供了较高的写性能，有限的查询性能与空间利用率。

论文使用leveling合并策略，提出在稳定的工作负载（层数不变）下，写性能在每个相邻层比例相同时，最优。

--------影响了后来的实现和改进。

次年有学者提出了tiering合并策略。

![LSM](D:\文件库\研究生\Learner\笔记\LSM.png)

相邻层的大小由比率T决定。每个层内的组件由其存储的key的范围标识。

Leveling：每层维护一个组件，每层的组件大小由T决定。每层写满之后合并到下层，多次合并之后，下层满了再写入下层。--------组件更少，查询时查找的组件少，是为查询优化。

Tiering：每层维护至多T个组件，该层满时，T个组件合并为一个写入下层。--------减少了合并频率，为了写入优化。



2、近来LSM-tree：

依然使用异地更新来减少随机io。利用磁盘中组件的不变性简化并发控制和恢复。

原本动态合并策略较难实现。所以现在的LSMtree通常选择实际的物理分区。。

1、BloomFilter：

一种空间高效概率数据结构，设计用来概况成员集合存在性。

**支持两种操作**：1、插入一个key。2、测试给定key是否在成员集合中。

**原理**：通过哈希将key映射到bit向量中的位置，1表示存在，0表示不存在。考虑到哈希碰撞的可能性，可能出现误报，但不会瞒报。

用于在磁盘组件之上建立，提高点查询性能，判断所查找的key是否在该组件中，若存在则在该组件中再进行查询。

显然对于需要查找的组件进行了初步筛选，提高查询效率。

**应用实例**：对于内部为B+树结构，非叶子节点足够小可以放入内存中，Bloomfilter建立在叶子节点之上，通过内存中的非叶子节点对叶子节点进行定位，然后先查询BloomFilter，可以减少磁盘io。

实际应用中，考虑其误报概率，采用10bits/key作为默认配置，将误报概率控制在1%，由于BloomFilter很小，可以将其缓存在内存中。

2、Partitioning：

将磁盘组件划分为多个小分区（通常是固定大小的）。称为SSTable（取自LevelDB）。

**优点**：1、打破了大组件的大粒度合并操作，分为细粒度的多个小分区的合并。限制了合并操作的处理时间，同时限制了创建新的组件所需要的临时磁盘空间。

2、分区可以通过仅合并具有重叠的key范围的组件对创建key或是针对性的更新进行优化。

顺序创建key：基本不执行合并，所有的key顺序递增，没有key范围重叠的分区。

针对更新：对于key范围很少更新的组件的合并频率大大降低。

原始LSM结构中动态合并组件原始具备这种分区优势。

**应用**：早期应用是分区指数文件（partitioned exponential file）PE-file。每个PE-file包含多个分区，每个分区可视为一个分离的LSM-tree。当分区过大时可会进行分裂。

但是其严格限制的key范围减少了合并的灵活性。（理解为每个PE-file作为合并的单位）。

**近来的实现策略**：分区与合并策略正交。

leveling和tiering都可以支持分区。

工业中实现分区leveling策略的LSM-tree有LevelDB、RocksDB。

**实现方案**：每层的磁盘组件根据范围划分为多个固定大小的SSTable。level0由于直接存储内存中的数据，所以不进行分区。同时也可以应对突发大量写入。

![LSM2](D:\文件库\研究生\Learner\笔记\LSM2.png)

如图，level1第一个分区写满需要合并，选中level2的与其key重合的分区进行合并，在level2生成新的分区。然后对原本的分区进行垃圾回收。

对下层的分区进行选择时有许多方法，LevelDB采用的是round-robin轮询策略。

Tiering 分区，每层有多个组件，所以可能有多个key范围重叠的SSTable，这是主要问题。

这些SSTable必须根据其时间戳进行排序来确保正确性。有两种对组件中SSTable进行组织的方式--------垂直分组和水平分组。

垂直分组时，key范围重叠的SSTable在一组中。每个组之间的key范围不相交。可以视为leveling分区对tiering的扩展支持。

合并时，组内数据合并到一起然后根据下层的范围重叠的组内的各个SSTable的范围生成新的SSTable并写入对应的下层的位置。SSTable的大小与下层的范围有关，不是固定的。

![LSM3](D:\文件库\研究生\Learner\笔记\LSM3.png)

水平分组时，不重叠的在一组。每个组件被划分为固定大小的SSTable。

每层的第一个组作为活跃组，接收前一次合并的SSTable。

合并时从该层中所有组中选择key范围重叠的SSTable，产生新的SSTable写入下一层的活跃组。

![LSM4](D:\文件库\研究生\Learner\笔记\LSM4.png)

3、并发控制与恢复

对于并发控制，LSM需要处理并发的读写操作，并且注意并发的刷盘和合并操作。

确保并发读写的正确性是数据库访问的一般要求，由于事务隔离性的需求，如今LSM通过锁/多版本并发机制来实现并发控制。

对于LSM自身，并发刷盘和更新是其自有的，为了避免正在使用的组件被删除，每个组件可以维护一个引用计数器。在对组件访问之前，请求可以先获取一个活跃组件的快照，将其引用计数加1。

**恢复**：

由于所有的写先写入内存，由WAL确保写操作的持久性。

对于WAL中的redo log和undo log：

steal/no-steal：
    steal策略代表允许将未提交的事务写入到磁盘，系统此时需要记录undo log，预防事务abort时进行回滚。
    no-steal策略，磁盘不记录未提交事务，事务提交之后才写入磁盘。系统不需要记录undo log。
committed/uncommitted
force/no-force：
    force策略事务提交时必须立即持久化到磁盘，立刻写磁盘会导致许多随机io如B+树原地更新。
    no-force策略事务可以在提交一会内再持久化到磁盘，缓存一些更新批量写入磁盘，如LSM异地更新。需要记录redo log。
  现在数据库大多采用steal no-force。需要记录redo、undo log。

no-steal策略恢复时对redo log进行重放。无需undo log。
理解为未提交的事务写入到磁盘，需要进行undo。
force：
理解为提交的事务未写入到磁盘，需要进行redo。

故障恢复在未分区情况：
   为每个组件维护一对代表存储条目的时间戳范围的时间戳。
   在分区的情况：
对key范围进行了分区，所以时间戳范围不适用（理解为）key被映射到不同SSTable。
LevelDB与RocksDB的解决方案，为对SSTable的操作维护一个日志，恢复时对日志进行重放。

复杂度分析：

写入：

leveling：O(T·L/B)

tiering：O(L/B)

读取：

leveling：O(L)

tiering：O(T·L)

4、LSM优化分类总结：

![LSM5](D:\文件库\研究生\Learner\笔记\LSM5.png)

1、写放大：

tiering比leveling有更低的写放大，但是查询性能和空间利用率差一些。

对于tiering的优化策略可视为垂直分区和水平分区的一些变体。

2、合并操作：

提高合并性能、最小化缓存未命中、消除写阻塞。

对合并的sstable的范围进行检测、并行合并减少io。

合并时，原本的组件在内存中的缓存会失效。

对于未分区的leveling合并策略，大量的写可能会导致flush与merge阻塞。

3、硬件：

大内存，在flush前对数据进行整合，提高空间利用率。

多核，对多线程访问的并发控制进行优化。

改变部分特性以充分利用硬件性能SSD/NVM。

原生存储，理解为持久化结构，实现在硬盘中，自行管理底层存储设备，充分利用顺序读写、io。

4、特殊负载：

对某些场景做特殊优化，比如牺牲范围查询性能，获取单点查询性能。

5、自动调参：

开发针对LSM - tree的自动调优技术，以减轻终端用户的调优负担。

对合并策略、布隆过滤器进行参数调优。

6、二级索引：

二级索引可以支持高效的查询处理。一般来说，基于LSM的系统会维护一个一级索引和多个二级索引。

理解为二级索引单独存储，增强查询处理。

5、经典实现：

1、LevelDB：

提供了简单的kv接口：put、get、scan。是一个嵌入式的存储引擎，旨在为更高层的应用提供支持。

贡献：

开创了leveling分区合并策略的设计与实现。

2、RocksDB：

是LevelDB的一个分支，增加了许多新特征。在默认比率为10的情况下，leveling实现90%的空间利用率。

其在合并策略、合并操作上有改进，额外还添加了新功能。

合并策略：

基于leveling分区，进行了部分改进。

在level0使用tiering，可以应对大量突然的写入，不会被合并阻塞。

支持动态调节每层的大小，考虑理想情况下在最下层存满时空间放大取最优值，现实中往往不会满足这个条件。

所以根据最高层的当前大小动态调节低层的大小以得到最优空间放大。

cold first：

选择冷的SSTable进行合并，可以让热的SSTable保持在低层中，减少写入开销。

delete first：

选择含有较多删除条目的SSTable，快速释放磁盘空间。

此外还提供merge filter API，用户用其指定策略，对条目进行过滤，部分条目被加入到SSTable中。

合并操作：

合并操作会消耗大量CPU和磁盘资源，会影响查询性能，且合并不可预测。

RocksDB采用leaky bucket机制解决，维护一个桶存储令牌，由令牌填充速度控制。每次指向写之前，所有的刷新和合并都必须请求一定数量的令牌。

新特性：

常规更新时，需要先读取然后再修改，RocksDB允许直接写入修改部分，避免的读取原始记录过程。

3、HBase：

基于基础的tiering合并策略，同时对合并策略进行了探索。

为管理时间序列数据设计了了日期tiering合并策略，根据组件的时间范围来合并组件、划分组件。

4、Cassandra：

支持未分区tiering策略、分区leveling策略和date-tiered策略，其每个数据分区都由一个基于LSM的引擎提供支持。

5、AsterixDB：

是一个开源大数据管理系统，旨在管理大量半结构化数据json-like；每个数据集由哈希映射，每个分区有一个基于LSM的存储引擎管理。

6、未来研究方向：

1、全面性能评估：分析查询、写性能之外，分析空间利用率。

2、分区tiering结构：目前的两种tiering分区策略：垂直/水平几乎涵盖了所有与tiering相关的优化，但是这两种方案的特点和取舍尚不清楚。一般来说，垂直分区选择合并SSTable时自由度更高，水平分区SSTable的大小是固定的。

3、混合合并策略：leveling和tiering策略可以混合使用，对点查询、空间放大影响较小。

4、最小化性能方差：理解为，合并导致刷盘停顿导致写停顿，以及内存写与磁盘写的性能差异。

5、面向数据库存储引擎：当时LSM改进局限于单个LSM的kv存储，随着LSM在数据库领域的广泛使用，应该针对更通用的多索引开发新的查询处理和读数据技术。

给出可能方向：1、辅助结果的自适应维护以促进查询处理。2、支持LSM的查询优化。3、LSM的维护与查询执行的协同（理解为，合并/flush对资源的占用影响查询性能）。

## 🍒总结评估：

Referred in [区块链存储优化学习/数据库存储索引](zotero://note/u/2Y9JV3V2/?ignore=1&line=49)