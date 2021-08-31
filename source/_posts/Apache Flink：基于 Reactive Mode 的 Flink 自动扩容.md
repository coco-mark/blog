---
title: Apache Flink：基于 Reactive Mode 的 Flink 自动扩容
category: Flink
date: 2021-08-28
---

## 简介

流式作业长时间运行过程中常常会经历不同流量负载的情况。流量负载会出现周期性的变化，如：白天与晚上、周末与工作日、节假日与非节假日，这些波动可能是突发事件或是业务的自然增长。虽然这些波动有些是可预见的，但是如果想要在所有场景下保证相同的服务质量，那么就需要解决如何让作业资源随着需求的变化而动态调整。

<!--more-->

一个简单的衡量当前所需资源与可用资源是否匹配的方法是：计算当前负载与可用的 workers 数之间的面积。如下图所示，左图中分配了固定的资源量，可用看到：实际负载与可用的 workers 之间有很大的差距 —— 因此造成了资源的浪费。右图中展示了弹性资源分配的情况，红线与黑线之间的距离在负载的变化中不断的努力减小。

![静态资源分配 vs 弹性资源分配](/img/intro.svg)


多亏了 Flink 1.2 引入的[可扩展状态（rescalable state）](https://flink.apache.org/features/2017/07/04/flink-rescalable-state.html)，我们可以**手动**对 Flink 作业扩/缩容，即可以通过调整并行度、重启任务的方式来调整资源。例如，如果你的 Flink Job 当前的并行度是 100，当负载升高时可以上调并行度到 200 并重启应用来应对负载的升高。

这种方式的问题在于你需要手动的借助一些自研工具来进行资源的计算与评估，并重新部署来进行合理的扩/缩容，不仅如此，这其中还可能包括一些异常的处理，以及对其他有相似情况的任务做同样的工作。

Flink 1.13 引入的[响应模式（reactive mode）](https://ci.apache.org/projects/flink/flink-docs-master/docs/deployment/elastic_scaling/)给你提供了另一种选择：这种模式下你只需要监控你的 Flink 集群，然后根据一些监控指标添加/移除相应的资源，剩下的事情 Flink 会帮你完成。响应式模式下 JobManager 会尝试引入所有可用的 TaskManager 资源用于当前的流数据处理。

响应式模式的一个巨大优势在于你不再需要详细的去了解 Flink 扩容相关知识就可以达到适应性扩容的目的。基本上可以把 Flink 看做一个服务器集群（如同 web 服务器、缓存、批处理），你可以根据所需进行扩/缩容。当前自动扩容在业界已经有非常成熟的方案，众多基础设施都提供相应的支持：主流的云服务都提供相关的指标监控组件，并适应性进行资源调整。例如，AWS 基于 [Auto Scaling groups](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html) 提供支持、Google Cloud 的 [Managed Instance groups](https://cloud.google.com/compute/docs/instance-groups)。相应的，Kubernetes 提供了 [Horizontal Pod Autoscalers](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)。

不同于其他支持自动扩容“服务器集群”，Flink 是一个包含状态的系统，通常需要处理重要的数据并保证强一致性（类似于数据库）。但并不像传统数据库那样，Flink 可以弹性的调整资源（基于 checkpoint 和状态后端）来优化当前的集群负载，而且没有太多的要求（如一个简单的 blob 存储用于状态备份即可）。

## 开始使用

通过以下步骤你可以在本地的 Flink 1.13.0 版本中体验一下响应模式：

```bash
# These instructions assume you are in the root directory of a Flink distribution.
# Put Job into usrlib/ directory
mkdir usrlib
cp ./examples/streaming/TopSpeedWindowing.jar usrlib/

# Submit Job in Reactive Mode
./bin/standalone-job.sh start -Dscheduler-mode=reactive -Dexecution.checkpointing.interval="10s" -j org.apache.flink.streaming.examples.windowing.TopSpeedWindowing

# Start first TaskManager
./bin/taskmanager.sh start
```

你已经开启了一个基于响应式模式的 Flink 任务。你可以通过 [Flink Web UI](http://localhost:8081/) 查看到刚刚启动的一个 TaskManager。如果想要扩容只需要简单的添加另外一个 TaskManager 即可：

```bash
# Start additional TaskManager
./bin/taskmanager.sh start
```

如果想缩容，停止一个 TaskManager 实例即可：

```bash
# Remove a TaskManager
./bin/taskmanager.sh stop
```

基于 [Docker](https://ci.apache.org/projects/flink/flink-docs-master/docs/deployment/resource-providers/standalone/docker/) 或 [standalone Kubernetes](https://ci.apache.org/projects/flink/flink-docs-master/docs/deployment/resource-providers/standalone/kubernetes/) 部署的 Flink 集群都可以在响应模式下部署任务（以上都需要基于 application 部署模式）

## 基于 Kubernetes 的示例

此章节，我们会演示一个真实场景中基于响应模式部署的示例。你可以将本示例作为自己部署自动扩容式集群的起点或模板。

### 操作流程

本示例的核心思路是基于 Kubernetes 的 [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)，该组件会监控所有 TaskManager pods 的 CPU 负载来相应的调整**副本因子（replication factor）**。当 CPU 负载升高时，autoscaler 会增加 TaskManager 资源来平摊压力；当负载降低时，autoscaler 会减少 TaskManager 资源。

整体的部署情况如下图所示：

![Kubernetes 部署图](/img/arch.png)

我们来逐一介绍一下：

**Flink**

*   **JobManager** 是基于 [Kubernetes job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) 部署。提交的 container 是基于官方的 Flink Docker 镜像，其中还包含了一个 Flink 任务的 jar 包。该 Flink 任务会从 Kafka  topic 中读取数据，然后对读取的事件进行复杂的数学运算。通过复杂的数学运算来使 CPU 负载升高。这种方式下，我们不需要部署大型的 Kafka 集群就可以模拟高负载的场景。

*   **TaskManager** 也基于 Kubernetes 部署，并通过 [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 进行扩容。本示例中，autoscaler 将会监控 pods 的 CPU 负载。pods 的数量会在 1～15 之间调整。

**其他的组件：**

*   我们部署了 **Zookeeper** 和 **Kafka**（各占用一个 pod），并创建了一个 topic 作为 Flink 任务的读数据源。

*   还有一个**数据生成器（Data Generator）** 的 pod 来周期性的向 Kafka topic 中写入 string 类型数据。在本示例中，写入速率的周期遵循正弦函数。

*   我们还部署了 **Prometheus** 和 **Grafana** 来用于监控。

如果你想自己尝试一下，以上都可以[从 Github 中获取](https://github.com/rmetzger/flink-reactive-mode-k8s-demo)。

### 结果

我们将以上组件全部部署在了仅包含一台主机的 Kubernetes 集群中，并运行了几天。以下的 Grafana 看板截图中展示了这几天运行的成果：

![响应模式实验结果](/img/result.png)

让我们更仔细的观察一下这个监控看板：

*   左上角图中是 **Kafka 消费延迟监控**，基于 Flink Kafka consumer（source 算子）上报的指标。该看板用于监控消费延迟的消息数。指标升高表示 Flink消费速度低于 Kafka producer 生产速度，此时需要扩容。该看板也反映了 Kafka 的吞吐量，最高的吞吐量约 75k，最小时为 0。 

*   右上角看板表示 **Flink 每秒吞吐量监控**，基于 Flink 的 reports per second 指标上报。该指标走向与正弦曲线大致相同，峰值约 6k/s，峰谷接近 0。

*   左下角的看板中展示了每个 TaskManager 的 **CPU 负载监控**。Kubernetes pod autoscaler 会基于该指标调整 TaskManager 的副本数量。可以看到每当 CPU 负载到达某个值时 TaskManager 的数量就会随之增加。

*   右下角图中展示了 **TaskManager 数量**。当吞吐（或 CPU 负载）升高时，我们可以看到 TaskManager 数量增大到 5（部分峰值下涨到了 8 个），最小时为 1 个。该图很好的展示了响应模式的工作过程：TaskManager 数量随着负载的变化而变化。

### 经验总结：将心跳超时配置降低能够让缩容更平顺

在我们刚刚开启实验时，我们从图标中注意到一些 Flink 反常的表现：

![响应模式未能很好的缩容](/img/high-timeout.png)

上面所有图中，我们可以看到会有毛刺出现：消费延迟曲线会突然增大到 600k（是平时 75k 正常峰值的 8 倍）。在“TaskManager 数量”监控看板中我们发现 TaskManager 数量某些情况下并没有很好的追随吞吐量曲线的变化。导致我们浪费了大量配置的 TaskManager 资源。

我们还发现这种情况只有在负载降低时才会发生，但是响应模式也是支持缩容场景的。那到底是什么原因导致毛刺出现以及 TaskManager 缩容不及时的呢？

在 Flink 中，JobManager 会定期发送心跳信息给 TaskManager 来确定 TaskManager 是否还存活。默认心跳的发送频率是 50s 一次。这个默认值看上去像是很高，但是在高负载情况下可能出现网络波动、gc 停滞或其他情况导致心跳数据发送延迟。我们不希望将短暂的中断判断成 TaskManager 彻底失联。

然而，这个默认值在本次实验中带来了问题：当 Kubernetes autoscaler 监控到 CPU 负载降低时会降低 TaskManager 数量，停止 TaskManager 实例。随之 Flink 会因为**数据传输层（data transport layer）**与这些 TaskManager 失联而立即停止数据处理，而且 JobMaster 将会等待 50s 后才会认定 TaskManager 真正被关闭了。

在 JobManager 等待期间吞吐量会降到 0，数据也会因此积压在 Kafka 中（消费延迟看板出现毛刺的原因）。当 Flink 重新运行起来后会对积压数据进行消费，从而造成了 CPU 负载的升高。autoscaler 监控到负载变化后会分配更多的 TaskManager，因此造成了 TaskManager 的浪费。

我们观察发现这种情况只会发生在缩容的场景中，因为缩容更加容易引起不稳定的情况的发生相比于扩容。扩容时，TaskManager 资源增加，数据停止处理仅发生在任务重启阶段（重启动作很快，仅会造成 Kafka 少量数据积压）；然而缩容时，数据停止处理的时间大约 50s。

我们通过调整 `heartbeat.timeout` 为 8s 来缓解了以上问题的发生。另外，我们期望后续社区能够优化 JobMaster 判断 TaskManager 失联的策略，能够更好、更快的的处理失联的场景。

## 总结

本文中我们介绍了 Flink 响应模式，这是 Flink 向动态资源规划、提升资源利用率方向迈进的重要一步。本文还演示了响应模式在 Kubernetes 中的实践，以及一些实践经验的总结与学习。

响应模式是 Flink 1.13 中的新特性被记录在产品开发文档的[ MVP（Minimal Viable Product）章节](https://flink.apache.org/roadmap.html#feature-stages)。在使用之前你还需要认真查看[官方文档](https://ci.apache.org/projects/flink/flink-docs-master/docs/deployment/elastic_scaling)中相关的使用限制。里面提到的[最大限制](https://ci.apache.org/projects/flink/flink-docs-master/docs/deployment/elastic_scaling/#limitations)是：只有在**独立应用部署模式（standalone application mode）**下才支持响应模式（即，不能在 active resource managers 部署模式或 session 部署模式中的集群使用）

社区非常期待大家针对这一特性的反馈，从而提升 Flink 弹性化资源管理能力。如何你有任何反馈请通过 [Flink 开发者邮件列表](https://flink.apache.org/community.html#mailing-lists)告诉我们，或在 [Twitter](https://twitter.com/rmetzger_) 里面 @ 我。

<img src="/img/wx_pub.png" width="400" />

*翻译自 [Apache Flink: Scaling Flink automatically with Reactive Mode](https://flink.apache.org/2021/05/06/reactive-mode.html)*

*译者：[可可](https://coco-mark.github.io/) @ [欢迎邮件联系我](mailto:cherry.picker2018@icloud.com.)*