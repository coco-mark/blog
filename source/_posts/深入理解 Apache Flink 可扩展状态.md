---
title: 深入理解 Apache Flink 可扩展状态
category: Flink
date: 2021-09-05
---

在 2017 年 5 月发布的 Apache Flink 1.2.0 中我们介绍了新特性**可扩展状态（rescalable state）**。接下来本篇文章会详细介绍 Flink 的**状态流处理（stateful stream processing）**与可扩展状态。

## 状态流处理简介

总的来说，我们可以认为**状态（state）**存储在**算子（operator）**的内存中，通过存储历史事件的信息来作用于未来事件的处理。

<!--more-->

相比而言，无状态算子在处理时仅仅关注当前的事件，不需要关注当前上下文以及过去的事件。举个例子：假设有个 source 算子发送事件 `e = {event_id:int, event_value:int}`，我们要对事件进行解析并输出 `event_value`。这一需求可以简单的通过 source-map-sink 来实现，map 算子用于解析 `event_value `，然后输出到 sink 算子。以上就是一个典型的无状态处理的示例。

但需求如果是只有当前 `event_value` 大于历史的 value 时才输出呢？这种情况下，很显然 map 算子需要记录历史中最大的 `event_value`。这就是一个状态流处理的示例。

这个示例说明：一些重要的场景下状态是流处理能够正确运行的基础。

## Apache Flink 中的状态

Apache Flink 是一个大型分布式系统，支持大规模状态流的处理。为了扩展的灵活性，Flink 会把任务逻辑拆解为以算子为顶点的**有向无环图（DAG）**，执行阶段会再将算子拆解为多个并行的实例。理论上，每个**算子并行实例（parallel operator instance）**都是一个独立的、可调度的**任务（task）**，tasks 运行在独立的机器上并通过网络连接在一起。

为了达到高吞吐、低延迟的目标，我们需要把 tasks 之间的网络传输降到最低。在 Flink 任务中，网络传输仅仅发生在 DAG 图的**边（edges）**中（即纵向来看，见“图 1”）。通过“边”流式数据得以从上游流向下游。

然而，并行实例之间（即横向来看，见“图 1”）不会进行数据传输。为了实现这种设计原则，**数据局部性（data locality）**显得尤为重要，这种设计原则也深深的影响着 Flink 状态的存储与访问方式。

为实现数据的局部性，Flink 将每个并行算子中的状态数据与对应 task 绑定，并存储在 task 所部署的那台机器中。

这种设计下，所有状态都存储在 task 本地，不用跨 task 进行状态数据传输。对于 Flink 这样的大型分布式系统，省去这种网络开销对于提升系统的扩展性意义重大。

在 Flink 中，我们将状态分成了两类：operator state 和 keyed state。operator state 用于算子的每个并行实例中（即 sub-task）；[keyed state 可以理解为被分区或被分片的 operator state，每个 key 对应特定的一个**状态分区（state-partition）**](https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/stream/state.html#keyed-state)。现在我们可以使用 operator state 轻松的实现前面的那个示例：operator state 存储历史最大值，从而决定 map 算子输出的 value。

## 状态流任务的扩容

对于无状态流而言，并行度（即，一个算子下并行 subtasks 的数量）调整非常简单。并行度调高时，只要开启新的算子并行实例，并将新实例与上下游连接即可；调低的过程与之相反。如**图 1** 所示。

相比之下，调整状态流并行度则更加复杂，因为我们必须 **1）重新分配**之前算子中的状态，并且保证分配前后状态的 **2）一致性**，以及 **3）有效性**。我们知道在 Flink 这种**无共享架构（no-shared architecture）**中，所有状态都存储在 task 本地，并且任务运行期间算子并行之间不会有消息的传递。

不过，Flink 已经有了一个 tasks 间传递状态数据的机制，而且还能够满足一致性，且保证了**精确一次（exactly-once）**语义——它就是 checkpointing！

更多 checkpointation 的细节可以从 Flink[官方文档](https://ci.apache.org/projects/flink/flink-docs-release-1.3/internals/stream_checkpointing.html)中查看。简单来说，**checkpoint 协调器（checkpoint coordinator）**会向流数据中插入特殊的事件（该事件被称为 checkpoint barrier）从而来触发 checkpoint。

checkpoint barriers 会随着流数据从 source 流向 sink，当算子实例接收到 barrier 之后会立即开展状态快照的创建，并将快照上传到分布式存储系统中（例如，HDFS）。

在状态恢复阶段，新 tasks（新 task 很有可能已经运行在另外一台机器中）会从分布式存储中寻找自己的状态数据并下载到本地。

![图 1](/img/stateless-stateful-streaming.svg)

如**图 1B** 所示，我们借助 checkpointing 实现状态的扩/缩容。首先，checkpoint 触发并将状态数据上传到分布式存储中。接着，并行度发生了改变并重启，任务会从分布式存储中拉取之前的状态快照进行恢复。这种方式解决了 **1）状态重新分配**和 **2）状态一致性**的问题，然而还存另一个问题：因并行度的改变，之前的状态数据与当前的算子并行实例无法一一对应，那么我们该如何 **3）正确的进行状态分配**呢？

当然，我们可以简单的将 map_1*（map 表示上图中 map 算子，1 表示并行实例的 id，译者注）*和 map_2 的状态分配给新的 map_1 和 map_2。这种方式会导致 map_3 没有分配到状态数据。这种幼稚的想法不但低效，而且在某些场景下还可能会导致错误的结果。

接下来的章节中，我们会介绍 Flink 时如何高效且正确的进行状态重分配。Flink 状态分为 operator state 和 keyed state，它们的分配方式会有所不同。

## Operator State 在扩/缩容场景下的重分配

首先，我们介绍一下 operator state 如何在扩/缩容时如何进行状态重分配。在 Flink 中一个最常见 operator state 使用场景是 Kafka sources 需要对当前每个 Kafka partitions 消费的 offsets 进行保存。每个 Kafka source 并行实例都需要使用 operator state 来保存 `<PartitionID, Offset>` 键值对，来存储每个 partition 当前消费的 offset。那么我们应该如何在扩/缩容时进行 operator state 的再分配呢？理想情况下，在扩/缩容后，我们可能希望将 checkpoint 中保存的所有 `<PartitionID, Offset>` 键值以借助轮训的方式再分配给所有的并行实例。

从用户角度而言，我们理解 Kafka partition offsets 的含义，我们清楚它们是独立的、可重新分配最小单元。但是问题是如何才能让 Flink 理解这一**特定的领域知识（domain-specific knowledge）**呢？

**图 2A** 中展示了 Flink 之前版本中 operator state 的 checkpoint 接口。在创建快照时，每个并行实例返回一个**对象（Object）**来表示当前状态。在 Kafka source 的场景中，这个对象内包含了 partition offsets 列表。

这个对象会上传到分布式存储中。在恢复阶段，这个对象将被并行实例从分布式存储中下载下来，以参数的方式传入到状态恢复的方法中。

这种方式在扩/缩容时会出现问题：因为 Flink 无法感知对象内的数据结构，也不知道如何将这个对象分解成有效的、可重分配的 partitions。尽管 Kafka source 中的 operator state 始终是以 list 的方式存储 partition offsets，但是由于之前版本中返回的是 `Object` 类型，对于 Flink 而言这是一个黑盒，所以无法将其重分配。

为了解决黑盒的问题，我们对 checkpoint 接口做了一个小调整，将接口类型改为了 `ListCheckpointed`。如**图 2B** 所示，新 checkpoint 接口接收并返回一个 list 类型的 state。使用 List 类型来替代 Object 使状态的分区边界更加明确：尽管 List 中的元素对于 Flink 而言仍然是一个黑盒，但是完全可以把它们当作是 operator state 中原子、可独立分配的单元。

![图 2](/img/list-checkpointed.svg)

Flink 提供了一组简单的 API，通过它们可以实现状态的拆分或合并。新 checkpoint 接口也使得 Kafka source 中存储的 partition offsets 状态分区更加明确，状态重分配过程也因此可以简单视为是 Lists 的拆分或合并过程。

```java
public class FlinkKafkaConsumer<T> extends RichParallelSourceFunction<T> implements CheckpointedFunction {
	 // ...

   private transient ListState<Tuple2<KafkaTopicPartition, Long>> offsetsOperatorState;

   @Override
   public void initializeState(FunctionInitializationContext context) throws Exception {

      OperatorStateStore stateStore = context.getOperatorStateStore();
      // register the state with the backend
      this.offsetsOperatorState = stateStore.getSerializableListState("kafka-offsets");

      // if the job was restarted, we set the restored offsets
      if (context.isRestored()) {
         for (Tuple2<KafkaTopicPartition, Long> kafkaOffset : offsetsOperatorState.get()) {
            // ... restore logic
         }
      }
   }

   @Override
   public void snapshotState(FunctionSnapshotContext context) throws Exception {

      this.offsetsOperatorState.clear();

      // write the partition offsets to the list of operator states
      for (Map.Entry<KafkaTopicPartition, Long> partition : this.subscribedPartitionOffsets.entrySet()) {
         this.offsetsOperatorState.add(Tuple2.of(partition.getKey(), partition.getValue()));
      }
   }

   // ...

}
```

## Keyed State 扩/缩容场景下的重新分配

keyed state 是 Flink 另外一种状态类型。相比于 operator state，keyed state 是基于 key 进行分区*（operator state 基于 subtasks 进行分区，译者注*），key 是通过流数据中 event 解析得来。

通过下面的例子来说明一下 keyed state 与 operator state 之间的不同。假设我们有个流数据格式为 `{customer_id:int, value:int}`。通过前面的介绍我们知道：可以通过 operator state 计算 value 的累加值。

现在我们对需求做一下变动：计算每一个 `customer_id` 下 value 的累加值。这就需要用到 keyed state，因为我们必须在每个 key 对应的数据流进行聚合统计，通过 keyed state 存储对应 key stream 下的累加值。

注意，在 Flink 中 keyed state 只能在 keyed streams 中使用，keyed stream 基于 keyBy() 方法创建。`keyBy()` 方法会 **1）让用户指定从事件中提取 key 的方法**，并且 **2）保证相同 key 的事件始终被同一并行实例处理**。正因如此，keyed state 也会绑定到这一并行实例上，每个 key state 也因此对应唯一一个并行实例。key 到 operator 的映射关系是通过 key 的 hash 值计算得来。

我们可以看到 keyed state 相比于 operator state 有个明显的优势：当进行扩/缩容时，可以很容易找出状态与并行实例之间的映射关系。状态的重分配只需要和 keyed stream 分区保持一致即可。在扩/缩容之后，key state 需要分配到对应并行实例中，keyed stream 的 hash 策略确定了 keyed state 与新并行实例之间的映射关系。

尽管 keyed state 顺利的解决了在扩/缩容时状态与 sub-tasks 之间映射的问题，但还留下了另一个问题：如何能高效的将状态传送到 subtasks 本地？

如果我们没有进行扩/缩容，那么每一个 subtask 可以简单的顺序拉取对应的状态。

但是在扩/缩容时，我们无法再进行简单的顺序读取：新 subtask 所属的状态数据可能会散布在所有的状态文件中（可以考虑一下，当你改变并行度之后 `hash(key) mod parallelism` 会发生什么样的变化）。通过**图 3A**我们可以清楚的看到，在并行度改变后状态数据读取过程发生巨大变化。这个示例中，我们展示了并行度从 3 变到 4 时，key（key 的范围：0～20） 的 shuffler 过程。

一个幼稚解决方法是：让新 subtasks 读取全部状态，然后从中再过滤出属于自己 key 的状态数据。尽管这种方式可以充分利用顺序读取带来的效率，但每个 subtask 将会读取大量与自身无关的状态数据，并且还会给分布式存储系统带来大量的读取请求。

另一种方法是在 checkpoint 时构建一个 keyed state 存储索引。这种方法可以让 subtasks 精准的读取到自己 key 所属的状态数据，从而避免读取大量无关数据。但这种方法也存在 2 个不足。

1. 所有 key 的索引信息（即 key 到存储位置和偏移量的映射）数据量可能会非常大；
2. 另外，这种方法会导致大量的**随机读写（random I/O）**（如**图 3A** 所示，在寻找对应 key 的状态数据时，随机读写大大降低了分布式文件系统的性能）。

Flink 的最终的方法是介于以上这两个极端方法之间：引入 key-group 作为状态分配的最小单元。那么它是如何工作的呢？key-group 的数量必须在任务启动之前指定，且一旦指定后将无法再做更改。由于 key-group 是状态分配的最小单元，这也意味着 key-group 的数量就是并行度的上限。简单来说，key-groups 是并行度的灵活性（它限制了并行度的上限）与索引和状态恢复开销之间的一种权衡。

我们将 key-groups 作为 key 范围分配给 subtasks。因此 subtasks 是顺序读取，通常这会涉及到多个 key-groups 的顺序读取过程。这种方式的另一个优势在于：减少了索引的元数据，元数据中只需要记录 key-group 到 subtask 映射关系。我们再不需要存储 key-to-subtask 明细列表，因为 key-group 的范围信息足够说明 key 的边界以应对扩/缩容。

在**图 3B** 中展示了 10 个 key-groups 在并行度从 3 到 4 的过程。我们可以看到，通过引入key-groups 以及范围重分配的方式极大的提升了访问效率。**图 3B** 中的计算公式 2 和计算公式 3 还详细的说明了 key-groups 的计算和分配过程。

![图 3](/img/key-groups.svg)

## 结束语

感谢你读到这里，我们希望你现在对 Apache Flink 状态的扩/缩容以及如何在现实场景中使用扩/缩容有了一个清晰的认识。

这个月初发布的 Flink 1.3.0 为状态管理和容错机制新增了很多特性，这其中包括增量 checkpoints。并且社区正在探索以下特性…

* 状态副本
* 不受 Flink 任务生命周期约束的状态
* 自动扩/缩容（不依赖 savepoints）

…这些会发布在 Flink 1.4.0 或更高版本中

如果你想了解更多，我们推荐阅读Apache Flink[官方](https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/stream/state.html)[文档](https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/stream/state.html)。

*这篇文章摘录自 Data Artisans 博客。如果你希望阅读最初的文章可以点击[这里](https://data-artisans.com/blog/apache-flink-at-mediamath-rescaling-stateful-applications)（外部链接）。*

<img src="/img/wx_pub.png" width="400" />

*翻译自 [A Deep Dive into Rescalable State in Apache Flink](https://flink.apache.org/features/2017/07/04/flink-rescalable-state.html)*

*译者：[可可](https://coco-mark.github.io/) @ [欢迎邮件联系我](mailto:cherry.picker2018@icloud.com.)*