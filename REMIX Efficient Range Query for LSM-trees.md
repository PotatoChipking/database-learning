# 标题：REMIX: Efficient Range Query for LSM-trees

## 🍅摘要：

LSM-tree based key-value (KV) stores organize data in a multi-level structure for high-speed writes. Range queries on traditional LSM-trees must seek and sort-merge data from multiple table ﬁles on the ﬂy, which is expensive and often leads to mediocre read performance. To improve range query efﬁciency on LSM-trees, we introduce a space-efﬁcient KV index data structure, named REMIX, that records a globally sorted view of KV data spanning multiple table ﬁles. A range query on multiple REMIX-indexed data ﬁles can quickly locate the target key using a binary search, and retrieve subsequent keys in sorted order without key comparisons. We build RemixDB, an LSM-tree based KV-store that adopts a write-efﬁcient compaction strategy and employs REMIXes for fast point and range queries. Experimental results show that REMIXes can substantially improve range query performance in a write-optimized LSM-tree based KV-store.

📅学习日期：2023-05-23
🍤作者：Zhong Wenshao,Chen Chen,Wu Xingbo,Jiang Song
🍎出版年份：2021
✉️刊于：
☀️影响因子：

## 🏁研究背景：

在传统LSM-tree结构中的范围查询，需要从多个表中查找、排序数据。这个操作的开销较高，影响读取性能。

bloomfilter对于范围查询的提供的帮助有限，理解为bloomfilter需要知道key才能进行确认，但是范围查询最多只知道区间值。

由于范围的值可能处在多个run中，LSM-tree的compaction会对run进行合并，从而减少run的数量，提高查找效率。

分析leveling：每层一个run，每次run写满时触发合并，写入下一层（重写）。写放大（每层维护有序，合并较多）、读优化（run的数量较少）。

tiering：每层多个run，每次一层的run数量到达上限时合并，写入到下一层的run中（新建）。写优化（一层写满才触发合并，合并次数相对少）、读放大（run的数量更多，需要查询更多次）。

硬件方面：

当今存储设备的随机读性能得到提升，flash ssd的随机读达到顺序读的50%.

英特尔傲腾的3D-XPoint提供的顺序、随机i/o性能相近。

在此背景下，论文针对tiering中的读性能弱点，提出利用硬件优势，使数据逻辑有序（不通过物理上组织。），提高查询性能。

## 🍚研究对象：

研究Tiering策略的读性能较差的问题，提出REMIX（range-query-efficient multi-table index）。实现高效写的同时，对读性能进行优化。并在remix基础之上建立了remixDB。

对合并策略进行了分析：

Leveling：分区的情况 ，每层维护一个run，有多个分区，当分区写满后，触发合并，选择下一层中与该分区的key范围重叠的分区进行合并，重写到下一层。写放大、读优化。

![Remix](D:\文件库\研究生\Learner\笔记\Remix.png)

Tiering：每层有多个run，当run的数量过多时触发合并，该层所有run进行合并，写入到一个新的run，写入下一层，新建不重写。写优化，读放大。

![Remix2](D:\文件库\研究生\Learner\笔记\Remix2.png)

进行范围查询时通过iterator进行，实现跨多个表像对一个有序的run来查询。

使用seek来对iterator进行初始化，使迭代器指向存储中的最小的满足范围的key。称为target key。

next用来将迭代器提前，指向排序的集合中的下一个，知道条件满足（数量、大小）。

在每个run内通过二分查找定位满足大于查找key的最小key，使用cursor进行标识。使用最小堆结构对这些key进行排序。

每次选择一个key时，会将cursor向前移，即指向下一个满足条件的key。理解为，对所有run某一时刻的key组成最小堆，选择时取出堆顶元素，移动cursor，将新的元素放入最小堆进行重新组织，再取堆顶元素。

## 🍟实现方案：

Insight：范围搜索在多个run之上建立了一个有序视图，这个有序视图继承了table的不变性，即直到发生compaction时才会发生变化。

现有LSM-tree架构的存储引擎未利用这个优势，而是在查询时反复构建、丢弃，引起许多计算、i/o开销，从而影响查询性能。

![img](data:image/png;base64,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)

REMIX数据结构：

![Remix3](D:\文件库\研究生\Learner\笔记\Remix3.png)

将排序视图中的key分割为多个segment，每个包含固定数量的key。

内部包括anchor key、cursor offsets、run selectors。

anchor key：该segment中的最小key。

cursor offsets：记录每个run中大于/等于anchor key的最小key的位置。

run selectors：记录该segment中每个key所在的run，按顺序组织。

iterator：

包括一些cursor、一个current 指针。

cursor对应一个run中key的位置。

current指针指向一个run selector。指向一个run，该run对应的cursor决定当前到达的key。

查找步骤：

1、对anchor key使用二分查找确定需要的segment，anchor key<=seek_key.。

2、iterator初始化指向anchor key。通过第一个run selector指向的run，选择对应run的cursor，即可确定anchor key的位置。

3、通过线性查找锁定目标key。提前iterator时，将cursor提前可以实现跳过key（同一个run中的key不记录，通过提前cursor和run selector来确定。）。同时将current 指向的run selector也提前到下一个。

例：查找17.

![Remix4](D:\文件库\研究生\Learner\笔记\Remix4.png)

1、选择小于等于17的最大anchor key对应的segment。选中第二个segment，anchor key为11.

2、iterator初始化。

iterator的cursor被替换为1、2、1.

iterator的指针指向run selector第一个0。

对应R0的cursor offset为1，即现在指向11.

3、线性搜索。

由于11不满足，将cursor提前，现在cursor集为2、2、1.

选择下一个run selector，指向1，R1的cursor为2，找到17，满足，结束查找。

segment中快速查找：

每个segement存放的key越多，对segment的选择越快，但是segment内的查找也越慢。

论文提出在segment内使用二分查找。

为了实现二分查找，必须能够进行随机查找，实现随机查找可以通过run的cursor与run selector计数来实现。

具体：

访问key需要对应run上的cursor。某个run中含有的key数量与其出现次数相同，通过将cursor提前指定次数，可以实现某个run上的随机访问，进而整个segment上的key也可以随机访问。

在segment中，cursor与run selector对应，即出现在run selector上的run，通过对应cursor是指向一个segment中的key的，未出现在run selector中则没有这个关系。

对run selector的计数通过SIMD实现，对其进行二分查找，然后确定对应次数，将该run上的cursor 跳过次数-1次，便可访问该key。

将该key进行比较，选择下一个，使用相同方法找到key，进行比较。

![Remix5](D:\文件库\研究生\Learner\笔记\Remix5.png)

I/O优化：

二分查找可能会导致多个run的访问，提出读取一个run时，对可能访问到的位置进行查找，剪枝查找范围。

REMIXDB：

使用tiering策略，真实场景下的业务通常具有空间局部性，分区策略可以有效的减少在真实场景下的compaction的消耗。

理解为将compaction的粒度变小，结合LSM综述中分区的优势。

REMIXDB采用分区Tiering策略，将key空间划分为不相交的范围。

每个分区的table文件由REMIX索引，提供整个分区的有序视图。（考虑垂直分组。）

RemixDB是一个单层的使用tiering策略的LSM-tree结构的存储引擎。

继承分区tiering的高效写，通过REMIX实现读优化。

点查询通过iterator进行搜索，不使用BloomFilter。

整体结构：

![Remix6](D:\文件库\研究生\Learner\笔记\Remix6.png)

将更新缓冲到Memtable，以及WAL。大小触发阈值时转化为Immutable，并创建一个新的。

分区内的Compaction创建一个包含新旧table文件的混合分区以及一个新的REMIX文件。旧版本进行垃圾回收。

文件结构：

table file：

分为block data和meta data。block data默认4KB，大文件存储为（jumbo block）。

block data：

开头是记录kv对在block中的偏移量的数组，用于随机访问单个kv。

可以存放至多255个kv对。

meta data：

是一个8-bit的数组，每个记录block中的key数量。

![Remix7](D:\文件库\研究生\Learner\笔记\Remix7.png)

REMIX file：

anchor key以B+树的形式组织，并与一个标识cursor offsets 和run selector的segementID关联。

cursor offset：由一个16-bit的block index和一个8-bit的key index组成。即blk-id和key-id。

一个key的多个版本可能存在分区内的多个不同table中（tiering垂直分组）。预留run selector的最高位标识版本。

![Remix8](D:\文件库\研究生\Learner\笔记\Remix8.png)

Compaction：

Abort：

取消一个分区的compaction，将新数据保留在memtable中。

compaction时需要重建remix文件，此时会导致很高的i/o消耗，用以控制分区合并来限制i/o消耗。

Minor Compaction：将新数据写入到一个或多个新建的table而不进行重写。重建REMIX文件。（compaction后table数量小于T时触发。）

Major Compaction：将新数据与一些或全部已有的table进行合并（重写）。（分区文件大于T时触发。）压缩效率为输入文件与输出文件的比值。如原本3个文件合并为一个。效率为3/1.

Split Compaction：将新数据与所有已有的数据进行合并并将分区分为一些新的分区。（当table文件很大时，major效率可能变低，此时需要对分区进行拆分，便可以减少每个分区中table的数量。）

REMIX重建：

REMIXDB利用分区中已有的REMIX文件，并采用高效的合并算法以最小化重建REMIX的I/O开销。

在分区中重建REMIX时，现有的table已被REMIX索引，可以将这些表视为一个有序的run。

将重建过程视为对两个有序的run进行排序合并，一个来自现有数据，一个来自新数据。

使用generalized binary merging algorithm来进行排序合并。

所有的run selector和cursor可以从现有的REMIX和表中取得，不需要I/O。

对于写密集、空间局部性弱的系统，采用多层tiering合并策略或是推迟单个分区内的REMIX重建，可以减少重建成本，代价是有更多层的排序视图。

## 🍒总结评估：

对于范围查询优化提供了思路，利用每次排序建立的iterator。

但是空间成本、重建成本、重建频繁程度较大。

Referred in [区块链存储优化学习/LSM Tree存储优化](zotero://note/u/2Y9JV3V2/?ignore=1&line=44)