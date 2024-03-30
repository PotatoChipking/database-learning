# 标题：Real-Time LSM-Trees for HTAP Workloads

## 🍅摘要：

Real-time analytics systems employ hybrid data layouts in which data are stored in different formats throughout their lifecycle. Recent data are stored in a row-oriented format to serve OLTP workloads and support high insert rates, while older data are transformed to a column-oriented format for OLAP access patterns. We observe that a Log-Structured Merge (LSM) Tree is a natural fit for a lifecycle-aware storage engine due to its high write throughput and level-oriented structure, in which records propagate from one level to the next over time. To build a lifecycle-aware storage engine using an LSM-Tree, we make a crucial modification to allow different data layouts in different levels, ranging from purely row-oriented to purely column-oriented, leading to a Real-Time LSM-Tree. We give a cost model and an algorithm to design a Real-Time LSM-Tree that is suitable for a given workload, followed by an experimental evaluation of LASER - a prototype implementation of our idea built on top of the RocksDB key-value store.

📅学习日期：2023-08-03
🍤作者：Saxena Hemant,Golab Lukasz,Idreos Stratos,Ilyas Ihab F.
🍎出版年份：2022
✉️刊于：
☀️影响因子：

## 🏁研究背景：

很多实时分析系统采用混合数据布局，其中数据在其整个生命周期中以不同格式存储。

较新的数据以行格式存储，以服务于OLTP工作负载--insert操作占比较高。

较旧的数据以列格式存储，用于OLAP负载。

而 LSM 设计是一个高吞吐量和多层结构，数据在层与层之间移动，基于这样的结构使得 LSM 本身对数据具有较强的生命周期感知能力。

新插入的数据常常以 TP 形式访问，即点查和更新；而历史数据一般以 AP 形式访问，如生成每小时或每月的报告，通常是扫描某些列。一些系统，如 SAP HANA、MemSQL、IBM Wildfire 将数据从行格式逐渐向列格式转换，这类系统具备生命周期感知数据布局。

LSM 常常应用于 KV 存储、关系型存储、时序存储、流存储等，但通常实现是单纯的行存或列族形式（RocksDB 的 CF），现有实现都没有注意到 LSM 可以在数据生命周期中更改数据布局。而这篇论文填补了这一空缺思想。

论文总结LSM tree的特点有：

1. 写优化，数据写以及 compaction 都是批处理，大大提高写吞吐量。
2. LSM 数据在各个 level 中自然的过渡。
3. 不同的 level 可以以不同的数据布局存储。这允许实现一个灵活的可配置的存储引擎，支持上层行存，下层列存，以适应不同的负载。
4. LSM 中核心动作 compaction 可以用于无缝用于改变数据布局。



## 🍚研究对象：

分析了LSM tree的写入放大、点查询、范围查询、空间放大的复杂度。

定义了生命周期驱动的混合负载概念。并关于生命周期与LSM tree结构的结合进行研究。

分析了LSM tree结构与生命周期感知结合的可行性。提出，利用LSM tree不同level以不同布局存储数据的方式实现生命周期感知。

提出column group对设计空间进行描述，针对不同布局引起的问题，重新设计了Compaction。

## 🍟实现方案：

**生命周期驱动的混合负载：**

在HTAP负载之外，具有较高的数据摄取速率、数据规模、数据持久化、随着数据生命周期访问模式会发生变化。

（生命周期即数据生成、修改、删除的版本变化周期）

该工作负载包括读写混合，较新的数据访问模式为OLTP风格，较久的数据由OLAP风格访问。

将上述负载标识为以下操作的混合：

1. insert(key, row)：插入一个新的条目
2. read(key, )：读 key 的 对应的列值
3. scan()：扫描范围 Key 在 low 和 high 之间的 对应的列值
4. update(key, )：更新 key 中指定的 列的值。
5. delete(key)：删除指定 key。

“Π” 作为投影列的集合，代表只访问部分列的数据。

**Column Group定义：**

混合存储布局由作为行存储在一起的列组（CG）定义。（row格式存储数据。）

例：row：A,B,C,D混合存储布局为两个CG：

CG1：<A,B,C>. CG2:<D>.

**通过CG表示的设计空间：**

![HTAP1](D:\文件库\研究生\Learner\笔记\HTAP1.png)

指的是 LSM 每一个 level 都可以使用特定的 CG 形式。例如上图，最左边是面向 TP 的行存形式；最右边是面向 AP 的列存格式；而中间是一种混合设计模式。

通过 CG 的配置改变数据生命周期不同阶段的布局来支持 HTAP 负载。在设计中会保持 memTable 和 level0 与原始的 LSM 一致，以此保持 LSM 的高吞吐量能力。在其他 level 中就会将数据分为 CG 布局，且 CG 的存储文件中仍然保留索引和 Bloom 过滤器。

**论文假设：**

1、level自顶向下，下层level的CG是上层Level的CG的子集。即不允许，Level 1的CG为<A,B>,<C>.Level2的CG为<A,B,C>的情况。

2、考虑某些负载，在某个Level中同时有OLAP和OLTP访问需求，此时没有一个CG能够满足负载需求，可以通过复制的方式实现，但是开销较大。

**Laser 存储引擎设计：**

The HTAP storage engine based on Real-Time LSM-Trees。

引入了列存系统中的一些概念：

1、存储CG的数据模型。2、列更新。3、stitching 单独列以重构元组。

考虑LSM tree结构查询数据时，某个元组的列值可能连续，因此对于CG数据模型，将其与key一起存储。引入的额外开销由于列存模型较优的压缩性能可以忽略。

![HTAP2](D:\文件库\研究生\Learner\笔记\HTAP2.png)

写入操作：

**insert** 执行方式与常规 LSM 一致，即将 KV 放入 memtable 中再随着时间和数据积累刷入磁盘以及在 level 中移动。

**update** 单独的列可能有两种实现方式，其一是获取完整元组，更新后再插入整个元组，这是面向行存的更新方式。在面向列存中应该支持更新特定的列，那么就会允许插入只包含更新后的列值部分。则在 compaction 过程允许部分行就会与完整行或其他部分列进行合并操作，以及旧值丢弃等。

读取操作：

**Point queries (with projections)** 中为了支持投影查找，在每个了level 中只会查找与投影列重叠的 CG，且在特定 CG 列中找到值即可返回查询结果。由于允许部分列的更新，就会导致同一个元组最新值跨不同的 level。例如上图中 key=108 AB 在 level_0，CD值在 level_2。

**Range queries (with projections)** 在每个 level 中进行遍历处理，然后按顺序返回值，且过滤掉旧版本值。每一层迭代器都是对指定的投影列来查找。与点查一样会在不同的 level 中找到列值。通过使用 **LevelMergingIterators** 合并不同 level 的值，通过 **ColumnMergingIterators** 来处理同一Level中各个CG中的列的值。

![HTAP3](D:\文件库\研究生\Learner\笔记\HTAP3.png)

问题：

由于更新时可能更新部分列，其他列的数据不再次写入，因此导致不同CG的填充速率不一。

进而对于Compaction，若是对每个CG一起处理，对于未写满的CG，会被合并到下层中，这中情况不符合生命周期感知设计。

解决：

重新设计Compaction-CG local compaction strategy。

如Figure 6

Compaction1：将Level1的CG<A,B>,合并到Level2的CG<A>,CG<B>中。

Compaction2：将Level2的CG<C>合并到Level3的CG<C>中。

![HTAP4](D:\文件库\研究生\Learner\笔记\HTAP4.png)

**LevelMergingIterators** 在范围查找和 compaction 时，从每个 level 获取和合并对应的元组，同时删除属性对应的旧版本值。

对于范围查询key[50-108],从level0-level2中搜索。key107和108具有多个版本，此时只返回最新版本的值。

**ColumnMergingIterators** 在相同的 level 中将不同的 CG 组合成一个元组。每个 LevelMergingIterators 会产生多个 ColumnmergingIterators。因为一个 level 只会有一个 key 的一个版本，所以 ColumnMergingIterators 不会过滤旧版本值。但可能面临同一 level 中列值为空的情况。

只更新部分列时，如图中key107、108，此时ColumnMergeingIterators只返回部分值。

**Compaction 的过程：**首先检测出溢出的 level，以及该 level 中最溢出的 CG。然后在下一个 level 中寻找重叠的 CGS。为每一个 level 开启一个或多个 ColumnMergingIterators，然后启动 LevelMergingIterators。最后以顺序写入下一个 level 中。

## 🍒总结评估：