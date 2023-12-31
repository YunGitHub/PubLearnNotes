---
layout: post
author: sjf0115
title: Hadoop 剖析 HDFS 文件读写
date: 2017-11-21 20:29:01
tags:
  - Hadoop

categories: Hadoop
permalink: hadoop-internal-anatomy-of-hdfs-file-wr
---

为了了解客户端及与之交互的HDFS、 NameNode 和 DataNode 之间的数据流是什么样的，下图显示了在读取文件时事件的发生顺序。

![](https://github.com/sjf0115/ImageBucket/blob/main/Hadoop/hadoop-internal-anatomy-of-hdfs-file-wr-1.png?raw=true)

客户端通过调用 FileSystem 对象的 `open()` 方法来打开希望读取的文件，对于 HDFS 来说，这个对象是分布式文件系统(步骤1)的一个实例。DistributedFileSystem 通过使用 RPC 来调用 NameNode ，以确定文件起始块的位置(步骤2)。对于每一个块， NameNode 返回存有该块副本的 DataNode 地址。此外，这些 DataNode 根据它们与客户端的距离来排序(根据集群的网络拓扑)。如果该客户端本身就是一个 DataNode (比如，在一个 MapReduce 任务中)，并保存有相应数据块的一个副本时，该节点就会从本地 DataNode 读取数据。

DistributedFileSystem 类返回一个 FSDataInputStream 对象(一个支持文件定位的输入流)给客户端并读取数据。FSDataInputStream 类转而封装 DFSInputStream 对象，该对象管理着 DataNode 和 NameNode 的I/O。

接着，客户端对这个输入流调用 `read()` 方法(步骤3)。存储着文件起始几个块的 DataNode 地址的 DFSInputStream 随即连接距离最近的 DataNode。通过对数据流反复调用 `read()` 方法，可以将数据从 DataNode 传输到客户端(步骤4)。到达块的末端时，DFSInputStream 关闭与该 DataNode 的连接，然后寻找下一个块的最佳 DataNode (步骤5)。客户端只需要读取连续的流，并且对于客户端都是透明的。

客户端从流中读取数据时，块是按照打开 DFSInputStream 与 DataNode 新建连接的顺序读取的。它也会根据需要询问 NameNode 来检索下一批数据块的 DataNode 的位置。一旦客户端完成读取，就对 FSDataInputStream 调用 `close()` 方法(步骤6)。

在读取数据的时候，如果 DFSInputStream 在与 DataNode 通信时遇到错误，会尝试从这个块的另外一个最邻近 DataNode 读取数据。它也记住那个故障 DataNode ，以保证以后不会反复读取该节点上后续的块。DFSInputStream 也会通过校验和确认从 DataNode 发来的数据是否完整。如果发现有损坏的块，就在 DFSInputStream 试图从其他 DataNode 读取其复本之前通知 NameNode 。

这个设计的一个重点是， NameNode 告知客户端每个块中最佳的 DataNode ，并让客户端直接连接到该 DataNode 检索数据。由于数据流分散在集群中的所有 DataNode ，所以这种设计能使HDFS可扩展到大量的并发客户端。同时， NameNode 只需要响应块位置的请求(这些信息存储在内存中，因而非常高效)，无需响应数据请求，否则随着客户端数量的增长， NameNode 会很快成为瓶颈。

来源：Hadoop权威指南
