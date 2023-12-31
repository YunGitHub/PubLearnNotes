---
layout: post
author: sjf0115
title: 有状态流处理:Flink状态后端
date: 2019-04-19 20:38:21
tags:
  - Flink

categories: Flink
permalink: stateful-stream-processing-apache-flink-state-backends
---

> Flink 1.12.0

这篇文章我们将深入探讨有状态流处理，更确切地说是 Flink 中可用的不同状态后端。在以下部分，我们将介绍 Flink 的3个状态后端，看一下它们的局限性以及如何根据具体案例需求选择最合适的状态后端。

在有状态的流处理中，当启用了 Flink 的检查点功能时，状态会持久化存储以防止数据的丢失并确保发生故障时能够完全恢复。为应用程序选择何种状态后端，取决于状态持久化的方式和位置。

Flink 提供了三种可用的状态后端：MemoryStateBackend，FsStateBackend 以及 RocksDBStateBackend。

![](https://github.com/sjf0115/ImageBucket/blob/main/Flink/stateful-stream-processing-apache-flink-state-backends-1.png?raw=true)

### 1. MemoryStateBackend

MemoryStateBackend 是将状态维护在 Java 堆上的一个内部状态后端。KeyedState 以及窗口算子都使用哈希表来存储数据值以及定时器。当应用程序进行 Checkpoint 时，状态后端会在将状态发给 JobManager 之前对状态进行快照，JobManager 会将状态存储在 Java 堆上。MemoryStateBackend 默认支持异步快照。异步快照可以避免阻塞数据流处理，从而避免发生反压。

使用 MemoryStateBackend 时的注意点：
- 默认情况下，每一个单独状态最大为 5 MB。可以通过 MemoryStateBackend 的构造函数修改最大值。
- 状态大小受到 Akka 帧大小的限制，所以无论在配置中怎么配置状态大小，都不能大于 Akka 的帧大小。
- 状态的总大小不能超过 JobManager 的内存。

什么时候使用 MemoryStateBackend：
- 本地开发或调试时建议使用 MemoryStateBackend，因为这种场景的状态不会很大。
- MemoryStateBackend 非常适合状态比较小的用例和流处理程序。例如一次仅处理一条记录的函数（Map, FlatMap，或 Filter）或者 [Kafka consumer](https://smartsi.blog.csdn.net/article/details/126475307?spm=1001.2014.3001.5502)。

### 2. FsStateBackend

FsStateBackend 配置需要文件系统的 URL（类型，地址，路径等）来配置。举个例子，比如可以是：
- hdfs://namenode:40010/flink/checkpoints
- s3://flink/checkpoints

当选择 FsStateBackend 时，正在处理的数据会保存在 TaskManager 的内存中。在 checkpoint 时，状态后端会将状态快照写入配置的文件系统中，同时会在 JobManager 或者 Zookeeper（在高可用场景下）的内存中存储一些元数据。

FsStateBackend 默认提供异步快照，以避免在 checkpoint 时阻塞数据流处理。可以通过在实例化 FsStateBackend 时将布尔标志设置为 false 来关闭异步快照：
```java
new FsStateBackend(path, false);
```
> 当前的状态仍然会先存在 TaskManager 中，所以状态的大小同样不能超过 TaskManager 的内存。

什么时候使用 FsStateBackend：
- FsStateBackend 非常适合处理大状态，大窗口，或大键值状态的有状态流处理作业。
- FsStateBackend 非常适合高可用方案。

### 3. RocksDBStateBackend

RocksDBStateBackend 的配置同样需要文件系统的 URL（类型，地址，路径等）来配置，如下所示：
- hdfs://namenode:40010/flink/checkpoints
- s3://flink/checkpoints

RocksDBStateBackend 将正在处理的数据使用 RocksDB 存储在本地磁盘上。在 checkpoint 时，要么将整个 RocksDB 数据库存储到配置的文件系统中，要么在状态比较大时将增量差异数据存储到配置的文件系统中。状态后端也会在 JobManager 或者 Zookeeper（在高可用场景下）的内存中存储一些元数据。。RocksDB 默认也是配置成异步快照。

使用 RocksDBStateBackend 时的注意点：
- RocksDB 的每个 key 和 value 的最大为 2^31 字节。这是因为 RocksDB 的 JNI API 是基于 byte[] 的。
- 我们需要在此强调，对于使用合并操作的有状态流处理应用程序，例如 ListState，随着时间的推移可能会累积超过 2^31 字节大小，这也将会导致后续检索的失败。

何时使用 RocksDBStateBackend：
- RocksDBStateBackend 非常适合处理大状态，大窗口，或大键值状态的有状态流处理作业。
- RocksDBStateBackend 非常适合高可用方案。
- RocksDBStateBackend 是目前唯一支持在有状态流处理应用程序中增量 Checkpoint 的状态后端。

在使用 RocksDB 时，状态大小只受限于磁盘可用空间的大小。这也使得 RocksDBStateBackend 成为管理超大状态的比较好的选择。使用 RocksDB 的权衡点在于所有状态的访问和检索都需要序列化（或反序列化）才能跨越 JNI 边界。与上面提到的堆上后端相比，这可能会影响应用程序的吞吐量。

原文:[Stateful Stream Processing: Apache Flink State Backends](https://www.ververica.com/blog/stateful-stream-processing-apache-flink-state-backends)
