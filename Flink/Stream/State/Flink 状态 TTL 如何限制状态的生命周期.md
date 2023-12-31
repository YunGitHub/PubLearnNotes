---
layout: post
author: smartsi
title: Flink 状态 TTL 如何限制状态的生命周期
date: 2021-06-26 10:56:01
tags:
  - Flink

categories: Flink
permalink: state-ttl-for-apache-flink-how-to-limit-the-lifetime-of-state
---

> Flink 1.6 版本

很多有状态流应用程序的常见需求是能够控制应用程序状态的访问时长以及何时删除它。这篇文章介绍了在 1.6.0 版本添加到 Flink 的状态生命周期时间（TTL）功能。

下面我们会介绍这个新的状态 TTL 功能的动机并讨论其用例。此外，我们还会展示如何使用和配置它，以及解释 Flink 如何使用 TTL 管理内部状态。文章最后还展望了对未来的改进和扩展。

### 1. Flink有状态流处理

任何实时流应用程序都会包含有状态操作。Flink 为容错状态流处理提供了许多强大的功能。用户可以选择维护状态的不同状态原语（原子值，列表，映射）和状态后端（堆内存，RocksDB）。处理函数中的应用程序逻辑可以访问和修改状态。通常，状态会与 Key 相关联，允许类似于 Key/Value 存储的可伸缩处理和存储。Apache Flink 透明地管理状态分布（包括对扩容和缩容的支持），并定期执行 Checkpoint，以便在出现故障时恢复作业，并提供状态 Exactly-Once 一致性语义的保证。

在文章的其余部分中，我们会介绍一个有状态应用程序示例，该应用程序提取用户登录事件，保存每个用户的最后一次登陆时间，以改善高频访问用户的的体验。

### 2. 状态的瞬态性质

状态应仅在有限时间内保存的主要原因有两个。

#### 2.1 遵守数据保护法规

围绕数据隐私法规的最新发展，例如，欧盟推出的新的通用数据保护条例 (GDPR)，遵守此类数据要求成为 IT 行业的一个重要话题。对于为客户提供短期服务并处理其个人数据的公司而言，特别是要求只保留有限的时间并在此后不能访问是一个共同的挑战。在我们存储上次登录时间的应用程序中，为防止对用户隐私进行不必要洞察，永久存储信息是不可接受的。因此，应用程序需要在一段时间后删除该信息。

#### 2.2 更有效地管理存储状态的大小

另一个问题是存储状态的规模不断增长。通常，当用户活跃时数据需要临时持久化，例如网络会话。当活跃结束时，数据不在用用处，而它仍然占用存储空间。应用程序必须采取额外的操作并明确删除无用状态以清理存储。按照我们之前存储上次登录时间的示例，一段时间后状态可能就没有必要了，因为稍后用户可能会被视为'不频繁'用户。

这两个要求都可以通过一个功能来解决：一旦不能再访问或一旦其价值不足以将其保存在存储中时，就会'神奇地'删除 Key 对应的状态。

### 3. 可以做些什么？

Apache Flink 1.6.0 版本开始引入了状态 TTL 功能。流处理应用的开发者可以将算子的状态配置为在一定时间内没有被使用下自动过期。过期状态稍后由惰性清理策略进行垃圾收集。

在 Flink 的 DataStream API 中，状态由状态描述符定义。状态 TTL 通过将 StateTtlConfiguration 传递给状态描述符来配置。以下 Java 示例展示了如何创建状态 TTL 配置并将其提供给状态描述符，该描述符将用户的上次登录时间作为 Long 值保存：
```java
import org.apache.flink.api.common.state.StateTtlConfig;
import org.apache.flink.api.common.time.Time;
import org.apache.flink.api.common.state.ValueStateDescriptor;

StateTtlConfig ttlConfig = StateTtlConfig
   .newBuilder(Time.days(7))
   .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
   .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
   .build();

ValueStateDescriptor<Long> lastUserLogin = new
ValueStateDescriptor<>("lastUserLogin", Long.class);lastUserLogin.enableTimeToLive(ttlConfig);
```

Flink 提供了多个选项来配置状态 TTL 功能：
- 什么时候重置 Time-to-Live ？默认情况下，当状态修改时会更新状态的到期时间。或者，也可以在读取时更新，但需要额外的写操作来更新时间戳。
- 哪些时间语义用于 Time-to-Live 计时器？在 Flink 1.6.0 中，用户只能在处理时间方面定义状态 TTL。计划在未来的 Apache Flink 版本中支持事件时间。
- 过期状态可以最后一次访问吗？假设某个状态已经过期，但它仍然在存储中并且没有被删除。如果可以读取此状态，那么用户可以为其值设置不同的可见性类型。在这两种情况下，状态随后都会被移除：
  - 第一个是永不返回过期状态。通过这种方式，对用户隐藏过期状态，这会阻止过期后访问任何个人数据。
  - 第二个是返回已过期但还没有垃圾回收的状态。此替代方案解决了最终存储清理很重要但应用程序仍可以充分利用仍然可用但已过期的状态的应用程序。

内部实现上，状态 TTL 功能是通过存储上次修改的时间戳以及实际状态值实现。虽然这种方法增加了一些存储开销，但它可以允许 Flink 在状态访问、Checkpoint、恢复以及存储清理过程中检查过期状态。

### 4. 垃圾回收

当一个状态在读操作中被访问时，Flink 会检查它的时间戳，如果过期则清除状态（取决于配置的状态可见性，是否返回过期状态）。由于这种惰性删除方式，永远不会再次访问的过期状态将永远占用存储空间，除非它被垃圾回收。

如果应用程序逻辑没有明确的处理，那么如何删除过期状态呢？一般来说，有不同的策略可以在后台进行删除。

Flink 1.6.0 仅在检查点或保存点生成完整快照时才支持自动驱逐过期状态。请注意，状态驱逐不适用于增量检查点。必须明确启用完整快照上的状态驱逐，如以下示例所示：
```java
StateTtlConfig ttlConfig = StateTtlConfig
   .newBuilder(Time.days(7))
   .cleanupFullSnapshot()
   .build();
```
本地存储大小保持不变，但存储的快照会减少。只有当算子从快照重新加载其状态时，即在恢复或从保存点启动时，算子的本地状态才会被清除。由于这些限制，应用程序在 Flink 1.6.0 过期后仍然需要主动删除状态。一种常见的方法是基于计时器在一定时间后手动清理状态。想法是为每个状态值和访问的 TTL 注册一个计时器。当定时器结束时，如果自定时器注册以来没有发生状态访问，则可以清除状态。这种方法引入了额外的成本，因为计时器会随着原始状态一起消耗存储空间。然而，Flink 1.6 对定时器处理进行了重大改进，例如高效的定时器删除（FLINK-9423）和 RocksDB 支持的定时器服务。

Apache Flink 的开源社区目前正在研究针对过期状态的额外垃圾收集策略。不同的想法仍在进行中，并计划在未来发布。一种方法基于 Flink 计时器，其工作方式类似于上述手动清理。 但是，用户不需要自己实现清理逻辑，状态会自动为他们清理。更复杂的想法取决于所选的状态后端：
- 堆内存状态后端中的增量部分清理在状态访问或记录处理时触发。
- RocksDB 特定的过滤器会在常规压缩过程中过滤掉过期的值。

### 5. 总结

基于时间的状态访问限制和自动状态清理是有状态流处理领域的常见挑战。随着 1.6.0 版本发布，Apache Flink 引入了第一个 State TTL 实现来解决这些问题。在当前版本中，状态 TTL 保证在配置超时后状态不可访问，以符合 GDPR 或任何其他数据合规性规则。Flink 社区正在开发多个扩展，以在未来版本中改进和扩展 State TTL 功能。

原文:[State TTL for Apache Flink: How to Limit the Lifetime of State](https://www.ververica.com/blog/state-ttl-for-apache-flink-how-to-limit-the-lifetime-of-state)
