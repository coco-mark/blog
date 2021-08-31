---
title: Apache Flink 大状态管理：增量 checkpointing 介绍
category: Flink
date: 2021-08-21
---

Apache Flink 专门为**状态流（stateful stream）**计算而搭建。那什么是流计算中的**状态（state）**？我在[上一篇博客](http://flink.apache.org/features/2017/07/04/flink-rescalable-state.html)中定义了状态和状态流计算，简单回顾一下：状态是被定义在**算子（operator）**中的一块内存空间，通过存储过去事件中的信息来影响未来事件的处理。

<!--more-->

在一些常见的复杂流计算场景中状态是一个基本且必要的概念。以下是[Flink 官方文档](https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/stream/state.html)中提到的几个经典场景：

* 当需要在应用中寻找特定格式的事件时，需要依赖状态来存储输入的事件序列

* 当需要按分钟进行数据聚合时，需要依赖状态来存储聚合过程的中间结果

* 当需要通过数据集训练机器学习模型时，需要依赖状态存储当前版本的模型参数

然而，状态流只有支持状态**容错（fault tolerant）**才能在实际生产过程中发挥其真正价值。“容错”意味着即使在软件或硬件层面出现了异常也不会影响最终结果的准确性，即：没有数据丢失和重复数据情况发生。

容错机制一直是 Flink 强大且著名的特性。该特性能够将软件或硬件异常带来的影响降到最低，并且 Flink 应用内保证了**精确一次（exactly-once）**语义的结果。

Flink 实现容错的核心机制是 [checkpointing](https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/stream/checkpointing.html)。checkpinting 是一次全局的、异步的应用状态快照生成的过程，该过程被周期性的触发，最终写入到可靠的存储中（通常为分布式文件系统）。当异常发生时，Flink 使用最近一次完成的 checkpoint 作为状态的初始点来重启应用。一些开发者的 Flink 应用会产生 GB 甚至 TB 级别的状态，这些开发者反馈，在大状态下制作 checkpoint 是一个缓慢且资源密集型的操作，正因如此我们在 Flink 1.3 中提出了**“增量快照（incremental checkpoint）”**。

在没有引入增量 checkpointing 之前，每次 checkpoint 不得不包含 Flink 应用的全部状态。我们注意到从上个 checkpoint 到下个 checkpoint 期间状态的改变通常比较小，往往没有必要对全部状态制作 checkpoint，为此我们引入了增量 checkpointing。不同于全量 checkpointing，增量 checkpointing 仅包含与上次 checkpoint 不同（或增量）的部分，并存储这些不同（或增量）的内容。

大状态场景下，增量 checkpointing 给性能带来了巨大的提升。之前生产环境的测试结果中，TB 级别的状态在增量 checkpointing 使用前后 checkpoint 时间从原来的 3mins 下降到 30s。这得益于 checkpoint 不再需要将全部的状态上传到持久化的存储中。

# 开始使用

当前 Flink 仅支持在 RocksDB 状态后端中使用增量 checkpointing，Flink 借助了 RocksDB 内部备份机制来定期对 checkpoint 进行整理。正因如此，增量 checkpoint 的历史记录不会无限制的增长下去，Flink 会自动的消耗和修剪旧的 checkpoints。

关于增量 checkpointing 的开启，[Apache Flink 的 checkpointing 官方文档](https://ci.apache.org/projects/flink/flink-docs-release-1.4/ops/state/large_state_tuning.html#tuning-rocksdb)中有更详细的介绍。简单来说，你需要正常开启 checkpointing，并在创建 RocksDB 状态后端实例时将构造器的第二个参数设置为`true`。

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStateBackend(newRocksDBStateBackend(filebackend, true));
```

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment()

env.setStateBackend(newRocksDBStateBackend(filebackend, true))
```

默认情况下，Flink 仅保留 1 个已完成的 checkpoint，如果需要保留更多，[可以通过以下配置](https://ci.apache.org/projects/flink/flink-docs-master/docs/dev/datastream/fault-tolerance/checkpointing/)：

```
state.checkpoints.num-retained
```

# 内部机制

Flink 增量 checkpointing 是以 [RocksDB checkpoints](https://github.com/facebook/rocksdb/wiki/Checkpoints) 为基础实现的。RocksDB 底层基于“[log-structured-merge（LMS）](https://en.wikipedia.org/wiki/Log-structured_merge-tree)”树形结构实现，它在内存中维护了一个被称作“memtable”的**可变的内存缓冲区（changeable in-memory buffer）**，key 相同的数据被更新时原 value 会被覆盖。当 memtable 写满时会将数据按照 key 顺序写入到磁盘中，写入过程中还会对数据进行轻量级的压缩。一旦数据写入到磁盘中，该数据将不可改变，此时的数据结构被称为**“sorted-string-table（sstable）”**

RocksDB 会启动一个后台任务来将重复的 key 进行整理、合并成新的 sstables，旧的 sstables 将被删除，合并后的 sstable 会包含之前所有数据。

在此之上，Flink 会追踪自上次 checkpoint 之后哪些 sstables 被创建或被删除。得益于 sstables 的不可变性，Flink 得以掌握 state 的变化。在 checkpoint 触发时，Flink 会将所有的 memtables 数据强制写入磁盘中，并在本地临时目录中创建一个**硬链接（hard-link）**。这个过程会发生短暂的同步阻塞，而后 checkpoint 的其他制作阶段都是异步完成的，不再有阻塞发生。

接着 Flink 将新生成的 sstables 拷贝到持久化的存储中（如 HDFS、S3），新 checkpoint 将为它们创建引用。Flink 并不是将现有所有的 sstables 全部拷贝到持久化存储，而是让新 checkpoint 再次引用它们。新 checkpoint 不会引用已删除的文件，因为 RocksDB 的合并过程最终会生成新的 sstable 来替代旧 sstables，旧文件最终会随着压缩合并过程被删除。增量 checkpoints 历史记录也因此得以被修剪。

为了追踪 checkpoints 之间的变化，Flink 需要将 RocksDB 压缩合并后的 sstables 上传到持久化存储中，这其中包含了部分冗余的内容。但由于 Flink 是增量式处理，所以这部分工作带来的开销很小，而且压缩后更少的 checkpoint 历史记录也利于 checkpoint 恢复，正因如此我们认为这样做是值得的。

# 举个例子
![增量 checkpoint 示例图](/img/incremental_cp_impl_example.svg)

示例中包含了一个**算子（Operator）**中的某个 subtask*（假定 operator id 为 2，subtask id 为 1，对应 op-2-1，译者注）*，这个算子中包含了 keyed state，另外 checkpoint 保留数配置为 **2**。上图中每一列分别表示：

1. 每次 checkpoint 时本地 RocksDB 包含的 sstables 文件情况；

2. checkpoint 对持久化存储中备份文件的引用记录；

3. 状态**共享注册表（shared state registry）**用于当 checkpoint 完成时统计文件引用的次数；

4. checkpoints 保留情况。

在制作 checkpoint “CP 1”时，本地 RocksDB 目录下包含了 2 个 sstables 新文件，这两个新文件将被上传到持久化存储中，目录名称与 checkpoint 名称相匹配。当 checkpoint 完成后，Flink 会在状态共享注册表中新增两条记录，并将引用数量设为 1。注册表中的 key 是通过 operator、subtask、sstables 名称共同组合而成。注册表中同时还保存了 key 到持久化存储路径的映射*（即图中第二列，译者注）*。

“CP 2”阶段，RocksDB 又新生成了 2 个 sstables 文件，原来的 2 个旧文件仍然存在。制作 “CP 2”时，Flink 会先将新 sstables 文件上传到持久化存储中，然后再将 2 个旧文件引用到“CP 2”*（旧文件已经在“CP 1”阶段上传到持久化存储中，因此这里只需要添加引用，译者注）*。本次 checkpoint 完成后， Flink 将注册表中的引用数加 1。

“CP 3”阶段，RocksDB 完成了 `sstable-(1)`、 `sstable-(2)` 和 `sstable-(3)` 到 `sstable-(1,2,3)` 的压缩合并，3 个旧的 sstables 文件被删除。此时，`sstable-(1,2,3)` 包含了之前所有的状态，重复的 entries 已经随着压缩合并过程被覆盖。此外，`sstable-(4)` 保留了下来，并且生成了新的 `sstable-(5)`。Flink将新的 `sstable-(1,2,3)` 和 `sstable-(5)` 上传，旧的 `sstable-(4)` 被再次引用。此时 checkpoint 数量达到了设置的上限（上限为 2），Flink 将 “CP 1” 删除，注册表中 “CP 1” 涉及到的引用数减 1（即 `sstable-(1)` 和 `sstable-(2)`）。

在 “CP 4”阶段，RocksDB 把 `sstable-(4)`、`sstable-(5)` 和新的 `sstable-(6)` 合并成 `sstable-(4,5,6)`。Flink 将新的 `sstable-(4,5,6)` 上传，并与旧的 `sstable-(1,2,3)` 一起引用到“CP 4”，`sstable-(1,2,3)` 和 `sstable-(4,5,6) ` 引用数加 1。此时 checkpoint 数量再次达到了上限，将 “CP 2” 删除，同时 `sstable-(1)`、`sstable-(2)` 和 `sstable-(3)` 引用数降到了 0，Flink 将它们从持久化存储中删除。

# 竞争状态与并发 checkpoints

Flink 支持并发执行多个 checkpoints，即会存在上个 checkpoint 未完成下个 checkpoint 已开始的情况。因此需要考虑到哪些地方新旧 checkpoint 会同时涉及。Flink 仅在**checkpoint 协调器（checkpoint coordinator）**确认后才会与持久化存储中的文件建立引用关系，因此避免了出现引用已删除文件的情况发生。

# checkpoint 恢复与性能考量

增量 checkpoint 的开启后不需要对状态恢复过程进行额外配置。故障发生时，Flink `JobManager` 会通知所有 tasks 从上次完成的 checkpoint 中恢复，无论全量 checkpoint 还是增量 checkpoint，每一个 `TaskManager` 都要从分布式文件系统中下载属于自己的那份状态文件。

尽管增量 checkpoint 特性在大状态场景下极大减少了 checkpoint 的制作时间，但这背后存在着一些权衡。总的来说，增量 checkpoint 减少了一般场景下  checkpoint 制作时间，但同时也带来了更长的恢复时间，具体恢复的时间取决于状态的大小。当集群出现严重异常时，Flink `TaskManagers` 不得不从多个 checkpoints 中进行恢复，恢复时间会比全量 checkpoint 更长。旧 checkpoints 仍然需要保留，因为新 checkpoint 需要引用它们，这可能导致 checkpoint 变更的历史记录无限的增长下去。你需要准备更多的分布式存储资源来保存它们，同时也需要准备更大的网络带宽来保证读取效率。

有些策略可以帮助你在简便性与性能之间找到更好的权衡点，推荐阅读[Flink 官方文档](https://ci.apache.org/projects/flink/flink-docs-release-1.4/ops/state/checkpoints.html#basics-of-incremental-checkpoints)了解更多的细节。

*这篇文章[最初](https://data-artisans.com/blog/managing-large-state-apache-flink-incremental-checkpointing-overview)[发表在 Data Artisans 博客](https://data-artisans.com/blog/managing-large-state-apache-flink-incremental-checkpointing-overview)中，是由 Stefan Richter 和 Chris Ward 贡献的。*

<img src="/img/wx_pub.png" width="400" />

*翻译自 [Managing Large State in Apache Flink: An Intro to Incremental Checkpointing](https://flink.apache.org/features/2018/01/30/incremental-checkpointing.html)*

*译者：[可可](https://coco-mark.github.io/) @ [欢迎邮件联系我](mailto:cherry.picker2018@icloud.com.)*