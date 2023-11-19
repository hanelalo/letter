---
title: Flink 入门：Flink 架构
date: 2023-11-18 12:55:33
tags: 大数据
categories: Flink
description: 在大数据时代下，为了普通的 OLTP 系统结合离线分析已经无法满足越来越复杂的业务，为了构建一个具备扩展性、高性能的性能，像 Flink 这一类流处理引擎日渐壮大。
---

在学习过程中，我一直在寻找一个答案，为什么要使用 Flink？如果没有 Flink，现在的软件系统就构建不起来了吗？

## Flink 简介

正如官网所说，Flink 是一个支持在有界、无界数据流上进行有状态计算的实时流式处理引擎。

* 有界：可以理解为数据流是有尽头的，比如要处理去年一整年的数据，那肯定是有界，是能知道什么时候处理完所有数据的。
* 无界：可以理解为数据流是没有尽头的，比如要处理某个线上系统的时时刻刻发生的事件，事件时刻都在发生，所以是不知道什么时候能处理完成的。
* 状态：比如 http 协议，我们知道它是一个无状态的协议，所以多个请求之间没法互相关联，每个请求无法知道上一个请求的内容，所以才有了 Cookie、Session 等解决方案。同样，在流处理中，如果我们想要做一些窗口处理的话，相当于是要将多个接收到的消息放到一起处理，就像是 http 中发出的每个请求，而状态，就是 Cookie、Session 了，它能存储多个消息处理后的结果，在后续消息到达时还能被更新。状态对于很多需要做一些聚合操作的场景是非常有用的。

基于以上的特性，Flink 能够在一些实时告警系统，比如一些反欺诈告警，还可以在一些事件驱动系统中使用，因为其实所谓事件，对 Flink 来说其实就是一个消息而已。

## 为什么需要 Flink？

前面简单介绍了 Flink，好像知道了它能干啥，但是为什么就必须用 Flink 呢？这里我其实是有两个疑问：

1. 不用这些流处理引擎，我自己按现在的传统的后端系统架构好像也能做这个事，为啥非要用这些流处理引擎呢？
2. 除了 Flink，就没有其他的工具或者中间件能做这个事？

### 为什么需要流式引擎？

诚然，当我们遇见一个需要进行实时的数据处理的需求时，完全可以新建设一个满足这个需求的系统，但是当这种要求实时处理的需求越来越多时，这个系统将会面临修改，面临性能问题，这也就意味着需要有足够的扩展性，需要有足够的性能，而当你开发出这样一个系统的时候，有没有一种可能，其实你开发出的是一个类似 Flink 的一个系统？那为何不直接用 Flink，偏偏要自己造轮子呢。

其实我的意思是，如果你只是有那么几个场景，几个需求需要进行像 Flink 这样的计算，那可以考虑不用 Flink，自己简单实现一下就行了，也不用过多考虑什么设计上的扩展性，但是如果这样的需求特别多的话，就需要考虑是否需要一个专业的流式处理引擎来做这件事了。

### 为什么选择 Flink？

首先我们需要知道两个概念：批处理、流处理。

批处理，其实就是对一个既定的数据集合做处理，比如处理上个月的交易数据，上个月的交易数据肯定是已经稳定不变的了，完全可以一个定时任务就完成所有事。

流处理，流处理和批处理不一样，批处理其实是完全知道会处理一些什么样的数据的，而且数据都已经准备好了，但是流处理是不知道数据什么时候到达，不知道数据是否已经全部到达的，甚至大部分时候要处理的数据一直在源源不断的产生。

如果听过了 Flink，那肯定会知道还有一个开源项目叫做 Spark，现在我们从数据处理的历史发展说起，探寻选择 Flink 的原因。

最开始的大批量数据处理，都是采用 MapReduce 进行处理，而且也只能处理存在 HDFS 上的存量数据，也就是只能进行批处理，不仅如此，MapReduce 的处理每个环节都会将处理后的数据写到 HDFS，而下一个环节又将这些数据读出来，处理完成后，又写入到 HDFS 中，但是实际上，中间过程产生的数据，其实是对业务无用的，一方面占用了磁盘，另一方面还因为数据的 IO 导致拉低了性能。

后来，又有了新一代的处理引擎 Spark，一方面，每个处理环节不在讲结果写入磁盘，而是放在内存中交给下一个环节进行处理，这弥补了 MapReduce 的缺陷；另一方面，它还通过微批的方式，宣称支持流处理，微批其实也是一种批处理，只不过是将一小部分到达的事件作为一个批次进行处理，所以叫做微批（micro batch），但是对于真正的流处理来说，Spark 的微批因为本质上还是批处理，所以还是存在一定的延迟，且当吞吐量持续上涨时，延迟也会一直增长，这是不能接受的。

在 Spark 之后，终于又迎来了 Flink 的时代，它和 Spark 一样是一个流批一体的处理引擎，且真正支持了流处理，而不是通过微批支持，所以哪怕吞吐量越来越大，也不会因为吞吐量的增长而影响延迟。

另外，Flink 作为一个现代的流处理引擎，在生态上，数据输入支持了主流的中间件，可以从 MySQL 等存储中获取数据，还可以直接从 kafka 消费实时数据，而数据输出也是支持了很多主流的存储、中间件，官方基本都有相应的开源工具做支持；在可扩展性上，得益于 Flink 的架构设计，它可以直接进行横向的扩缩容。

总结一下，选择 Flink 的原因如下：

* 活跃的开源社区，完整的生态。
* 高可扩展性。
* 真正的流批一体。

## Flink 架构

上文说到，Flink 的架构设计保证了 Flink 能够直接进行扩缩容，可能是因为它和一些传统应用的中间的场景不同，所以架构也是有些别致。

Flink 的部署架构上，为 Job Manager、Task Manager、Client 3 个组件，所以 Flink 采用的是类似 Master-Slave 的模式。

* Client

  因为 Flink 的使用方式一般是将写好的 Flink 程序打包后，上传到 Flink 集群，由 Client 解析成逻辑执行图，将这个应用程序当成一个 Job 提交到 Job Manager。

* Job Manager

  Job Manager 将逻辑执行图解析成物理执行图，一个 Job 就会变成多个 Task，然后将这些 Task 交给 Task Manager 执行，Job Manager 主要职责是协调 checkpoint、协调 Task Manager 故障转移、Task 调度等。
  
  Job Manager 实例上还分为 Dispatcher、Job Master、Resource Manager 3 个组件。
  
  * Dispatcher
  
    提供了一系列 REST 接口，用于接收客户端提交的 Job，并启动一个新的 Job Master 给新的 Job，当然还有一些其他的各种查询状态的接口也在这里。
  
  * Job Master
  
    用于管理有个 Job，Job Manager 上可以同时运行多个 Job，但是每个 Job 都有自己的 Job Master。
  
  * Resource Manager
  
    用于管理、分配 Flink 集群的计算资源。主要就是管理 task slots，这个在 Task Manager 中详细说明。Flink 实现了多个 Resource Manager，使得 Flink 集群可以部署在 Kubernetes、Yarn 上，还可以直接单机部署，但是因为单机部署的话，task slots 其实在启动时就确定了，这种模式下，Resource Manager 是没办法新启动一个 Task Manager 的。
  
* Task Manager

  Task Manager 主要是执行由 Job Manager 分配的 Task。一个 Flink 集群中至少要有一个 Task Manager。每个 Task Manager 上可以有多个 task slots，task slots 代表着这个 Task Manager 可以并行处理的任务数。如果 task slots 是 3，那代表这个 Task Manager 实例可以同时处理 3 个任务。也代表着每个 task 最多占用 Task Manager 1/3 的内存，同一个 Task Manager 中，task 之间只会隔离内存，而不会隔离 CPU、网络等资源。

### 部署架构

Flink 应用程序会衍生出一个或多个 Job，而 Job 可以在本地环境或者一个远程环境运行，远程环境还可以是一个集群环境。根据任务的生命周期不同，可以选择不同的部署方式。

#### session 模式

* 集群生命周期

  session 模式下的 Flink 集群，是在运行 Job 之前就已经准备好了 Job Manager、Task Manager，当集群上的 Job 运行完成后，集群依然还是保持运行，等待新的 Job 到来。换句话说，集群的生命周期和 Job 的生命周期并不相关。

* 资源隔离

  所有 Job 共享同一个集群，也就意味着会有一些资源竞争，比如网络带宽。同样也是因为共享集群，所以当某个 Task Manager 实例故障时，会影响在上面运行的所有 task；但是如果是集群的 Job Manager 故障，那么将会影响所有在集群上运行的 Job。

基于 session 模式的特性，如果很多任务本身执行的时间是特别短的，甚至相对于集群的启动时长来说都比较短，那么可以考虑一 session 方式部署。

在这个模式下，还是需要由 Client 执行 main() 方法，解析出执行图，准备好依赖，然后将执行图和依赖上传到 Job Manager，但是如果提交的 Job 比较多的话，首先在 Client 这里就会阻塞住，因为上传依赖是比较耗时的。

#### application 模式

* 集群生命周期

  application 模式的集群是给某个 Flink 应用程序专用的集群，换句话说，每一个提交的 Flink 应用程序，都会单独分配一套完整的 Flink 集群，包括 Job Manager、Task Manager 等。这意味着得将 Flink 部署在类似 k8s 这样的环境下。

  application 模式下的集群的生命周期和 Flink 应用程序关联，当 Flink 应用程序执行完成后，集群资源也将会释放。

  需要知道的是，一个 Flink 应用程序是可以提交多个 Job 的，也就意味着，同一个 Flink 应用程序提交的所有 Job 会共享一个集群。

  另外，application 模式下，main() 方法的调用是在 Job Manager 上执行。

* 资源隔离

  站在 Job 的角度看，集群的资源隔离粒度比 session 模式下更细腻，做到了在每个 Flink 应用级别的资源隔离，其他的区别不大。

  故障也只影响当前这个 Flink 应用所提交的 Job。

#### per-job 模式

也叫 job 模式。

在 Flink 1.15 时，这种模式被置为过时，官方建议以 application 模式代替。

* 集群生命周期

  在 per-job 模式中，会为每个提交的 job 启动一个集群。首先客户端会请求集群管理器创建 Job Manager，然后将 Job 提交到 Job Manager，Job Manager 再惰性分配运行 Job 需要的计算资源。

  当 Job 运行完之后，集群将被销毁，资源被回收。

* 资源隔离

  因为是为每个 Job 单独启动的集群，所以集群故障也只会影响一个 Job。

因为需要等待分配 Task Manager 资源，所以 per-job 模式更适合运行长时间运行着的任务，即对启动时间并不敏感的任务。

## 总结

首先我们介绍了 Flink 用途，然后解释了为什么需要流式引擎，为什么选择 Flink，最后我们介绍了 Flink 的部署架构。

这里再下节一下 3 种部署架构的区别：

* per-job 模式的资源隔离是在 Job 维度，而 application 模式的资源隔离则是在 Flink 应用程序级别，session 则是基本算是没有资源隔离，application 模式算是一种另类的 session 模式。
* 在 session、per-job 模式下，Client 需要下载 Flink 程序的依赖、执行 main() 方法解析执行图、上传执行图和依赖到集群。
* 在 application 模式下，Client 则是直接将 Flink 程序提交到集群中，由 Job Manager 做依赖下载、调用 main()方法解析执行图，因为已经在 Job Manager 上运行，所以不存在上传执行图和依赖这一步。
* 相比之下，application 节省了 Client 上传依赖、执行图的带宽，使 Client 变得轻量级。
* application 模式下，因为是在 Job Manager 上下载依赖，所以需要保证依赖所在服务和 Flink 集群的网络是通的。

## 参考

* [Apache Spark Vs Flink](https://www.macrometa.com/event-stream-processing/spark-vs-flink)
* [Application Deployment in Flink: Current State and the new Application Mode](https://flink.apache.org/2020/07/14/application-deployment-in-flink-current-state-and-the-new-application-mode/)

