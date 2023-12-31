---
layout: post
author: sjf0115
title: 图解 CAP 理论
date: 2019-09-21 12:35:17
tags:
  - 分布式

categories: 分布式
permalink: an-illustrated-proof-of-the-cap-theorem
---

CAP 定理是分布式系统中的一个重要的基本定理，指出任何分布式系统最多只能具有以下三个属性中的其中两个：
- Consistency(一致性)
- Availability(可用性)
- Partition tolerance(分区容错)

> 1998年，加州大学的计算机科学家 Eric Brewer 提出，分布式系统有上述三个指标。它们的第一个字母分别是 C、A、P。Eric Brewer 说，这三个指标不可能同时做到，因此这个结论就叫做 CAP 定理。

### 1. 什么是CAP定理

CAP定理指出任何分布式系统不可能同时保持一致，可用性以及分区容错性。听起来很简单，但是保持一致性意味着什么？保持可用性呢？保持分区容忍呢？分布式系统到底意味着什么？

在这篇文章中，我们将介绍一个简单的分布式系统，并说明如果系统保持一致性、可用性以及分区容错性意味着什么。

### 2. 分布式系统

让我们来考虑一个非常简单的分布式系统。我们的分布式系统由两台服务器G1和G2组成；这两台服务器都追踪同一个变量v，变量v的初始值为v0；G1和G2之间可以相互通信，同样也可以与外部的客户端通信；我们的分布式系统的架构如下图所示：

![](https://github.com/sjf0115/ImageBucket/blob/main/Distributed/an-illustrated-proof-of-the-cap-theorem-1.jpg?raw=true)

客户端可以向任何服务器发送读写请求。当服务器接收到请求之后，将根据请求执行一些计算，然后把请求结果返回给客户端。一个写请求过程如下所示：

![](https://github.com/sjf0115/ImageBucket/blob/main/Distributed/an-illustrated-proof-of-the-cap-theorem-2.jpg?raw=true)

下面是一个读请求过程：

![](https://github.com/sjf0115/ImageBucket/blob/main/Distributed/an-illustrated-proof-of-the-cap-theorem-3.jpg?raw=true)

现在我们已经建立好我们的分布式系统，下面我们一起探讨一下分布式系统的一致性、可用性以及分区容错性的含义。

### 3. 一致性

在 Gilbert 和 Lynch 论文中一致性的描述为：
```
在写操作完成之后开始的任何读操作必须返回写操作的值，或者更后续写操作的结果值。
```
> any read operation that begins after a write operation completes must return that value, or the result of a later write operation.

这就意味着在一个一致性的分布式系统中，客户端向任何服务器发起一个写请求，将一个值写入服务器，那么之后向任何服务器（不一定是修改值的服务器）发起的读请求，都必须读取到这个值（或者更新的值）。

下面是一个非一致性的分布式系统的例子:

![](https://github.com/sjf0115/ImageBucket/blob/main/Distributed/an-illustrated-proof-of-the-cap-theorem-4.jpg?raw=true)

上图中客户端向G1服务器发起写请求，将变量v的值从v0更新为v1，并得到G1服务器的确认响应。但当向G2服务器读取变量v的值时，读取到的却是旧的值v0，与期待的v1不一致。

下面是一个一致性的分布式系统的例子:

![](https://github.com/sjf0115/ImageBucket/blob/main/Distributed/an-illustrated-proof-of-the-cap-theorem-5.jpg?raw=true)

在这个系统中，G1在给客户端发送确认之前，会先把v的新值复制给G2，这样，当客户端从G2读取v的值时就能读取到最新的值v1。

### 4. 可用性

在 Gilbert 和 Lynch 论文中可用性描述为：
```
系统中非故障节点收到的每个请求都必须返回响应。
```
> every request received by a non-failing node in the system must result in a response.

在一个可用性的分布式系统中，如果我们的客户端向服务器发送请求并且服务器没有崩溃，那么服务器最终必须返回响应给客户端。不允许服务器忽略客户端的请求。

### 5. 分区容错性

在 Gilbert 和 Lynch 论文中分区容错性描述为：
```
从一个节点发送到另一节点的过程中网络允许任意消息的丢失。
```
> the network will be allowed to lose arbitrarily many messages sent from one node to another.

这就意味着服务器G1和G2之间互相发送的任意消息都可能丢失。如果所有的消息都丢失了，那么我们的系统就变成了如下所示：

![](https://github.com/sjf0115/ImageBucket/blob/main/Distributed/an-illustrated-proof-of-the-cap-theorem-6.jpg?raw=true)

> 与第一个图相比，两个节点之间缺少了联系。

为了满足分区容错性，我们的系统需要能在网络分区情况下也能正常的工作。

> Network Partition: 网络分区（网络分裂）

### 6. 证明

现在我们已经了解了一致性、可用性和分区容错性的含义，现在我们来证明一个系统不可能同时满足这三个属性。

假设存在一个同时满足这三个属性的系统，我们要做的第一件就是让系统发生网络分区，就像下图的情况一样：

![](https://github.com/sjf0115/ImageBucket/blob/main/Distributed/an-illustrated-proof-of-the-cap-theorem-7.jpg?raw=true)

接下来，我们有一个客户端向G1发起写请求，将v的值更新为v1。因为系统是可用的，所以G1必须给客户端发送响应，但是由于网络分区，G1无法将其数据复制到G2。

![](https://github.com/sjf0115/ImageBucket/blob/main/Distributed/an-illustrated-proof-of-the-cap-theorem-8.jpg?raw=true)

接着，客户端向G2发起一个读请求，因为系统是可用的，所以G2必须给客户端返回响应。又由于网络分区的，G2无法从G1更新v的值，所以G2返回给客户端的是旧的值v0。

![](https://github.com/sjf0115/ImageBucket/blob/main/Distributed/an-illustrated-proof-of-the-cap-theorem-9.jpg?raw=true)

客户端已经将G1上v的值修改为v1，但是从G2上读取到的值仍然是v0，这违背了一致性。

我们假设存在一个满足一致性、可用性、分区容错性的分布式系统，但是在出现网络分区等情况下，系统会表现出不一致的行为，因此证明不存在这样一个同时满足一致性、可用性、分区容错性的系统。

### 7. A还是C

如果保证G2的一致性，那么G1必须在写操作时，锁定G2的读操作和写操作。只有数据同步后，才能重新开放读写。锁定期间，G2不能读写，所以不能提供可用性；如果保证G2的可用性，那么势必不能锁定G2，所以一致性不成立。

综上所述，G2无法同时做到一致性和可用性。系统设计时只能选择一个。如果追求一致性，那么无法保证所有节点的可用性；如果追求所有节点的可用性，那就没法做到一致性。

所以，对于一个分布式系统来说，P是一个基本要求，CAP三者中，只能根据系统要求在A和C两者之间做权衡。

英译对照:
- Consistency：一致性
- Availability：可用性
- Partition tolerance：分区容错

原文:[An Illustrated Proof of the CAP Theorem](https://mwhittaker.github.io/blog/an_illustrated_proof_of_the_cap_theorem/)
