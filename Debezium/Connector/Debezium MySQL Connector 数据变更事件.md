---
layout: post
author: wy
title: Debezium MySQL Connector 数据变更事件
date: 2022-02-01 15:40:21
tags:
  - Debezium

categories: Debezium
permalink: debezium-mysql-connector-data-change-events
---

Debezium MySQL Connector 会为 INSERT、UPDATE 以及 DELETE 行级操作生成一个数据变更事件(Data Change Events)。每个数据变更事件都会包含一个 Key 和一个 Value，具体结构取决于所要变更的表。

Debezium 和 Kafka Connect 都是为连续事件消息流设计的。然而，随着时间的推移这些事件的结构也可能会发生变化，这无疑给消费者处理事件增加了难度。为了解决这个问题，每个事件都包含一个 schema，或者可以使用 schema 注册表，包含了一个 schema ID，消费者可以使用它从注册表中获取 schema。这样我们就可以单独的处理每个事件了。

下面的 JSON 展示了变更事件的四个基本组成部分。需要注意的是，应用程序中 Kafka Connect Converter 的配置决定了变更事件中这四个部分的表示样式。当配置 Converter 产生 Schema 时，Schema 才会出现在变更事件中。同样，当配置 Converter 产生事件 Key 和事件 Payload 时，Key 和 Payload 才会出现在变更事件中。如果我们使用 JSON Converter 并配置生成变更事件所有四个基本部分时，变更事件具有如下结构：

![](1)

第一个 schema 字段是事件 key 的一部分。指定了一个 Kafka Connect schema，描述了事件 key 中 payload 部分的内容。换句话说，第一个 schema 字段描述了变更表的主键或唯一键（如果表没有主键）的结构。可以通过设置 message.key.columns Connector 参数来覆盖表的主键。在这种情况下，schema 字段描述了由该属性标识的 key 的结构。第一个 payload 字段是事件 key 的一部分。与第一个 schema 字段描述的结构一致，并且包含变更行的键。

第二个 schema 字段是事件 value 的一部分。指定了一个 Kafka Connect schema，描述了事件 value 中 payload 部分的内容。换句话说，第二个 schema 字段描述了被更改行的结构。通常，此 schema 字段包含嵌套 schema。第二个 payload 字段是事件 value 的一部分。与第二个 schema 字段描述的结构一致，并且包含变更行的实际数据。

### 1. 变更事件的 Key

变更事件的 Key 包含变更表的键和变更行的实际键的 schema。在 Connector 创建事件时，schema 及其相应的 payload 字段都会为变更表的 PRIMARY KEY（或唯一约束）中的每一列包含一个字段。假设有如下 customers 表：
```sql
CREATE TABLE customers (
  id INTEGER NOT NULL AUTO_INCREMENT PRIMARY KEY,
  first_name VARCHAR(255) NOT NULL,
  last_name VARCHAR(255) NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE KEY
) AUTO_INCREMENT=1001;
```
捕获的 customers 表的变更事件都具会有相同 schema 的事件的 Key。只要 customers 表定义好，捕获的 customers 表的每个变更事件都具有如下 Key 结构：

![](2)

事件 key 的 schema 字段指定了一个 Kafka Connect schema，描述了事件的 key 中 payload 部分的内容；name 字段定义了 schema 名称。schema 名称的格式为 connector-name.database-name.table-name.Key。在这个例子中：
- mysql-server-1 是生成此事件的 Connector 名称。
- inventory 是包含变更表的数据库。
- customer 变更表。

schema 下的 optional 字段指示事件 key 是否必须在 payload 字段中包含一个值。在此示例中，需要在 payload 中包含一个值。当表没有主键时，事件 Key 的 payload 字段中的值是可选的。fields 字段指定 payload 中预期的每个字段，包括每个字段的名称、类型以及是否需要。payload 字段包含生成此变更事件的行的键。在此示例中，包含一个值为 1001 的 id 字段。

### 2. 变更事件的 Value

变更事件中的 Value 内容要比 Key 的复杂一些。与 key 一样，Value 也包含一个 schema 和 payload。创建、更新或删除数据的操作的更改事件都具有带有信封结构的值负载。

假设有如下 customers 表，跟变更事件中的 Key 中的示例表一样：
```sql
CREATE TABLE customers (
  id INTEGER NOT NULL AUTO_INCREMENT PRIMARY KEY,
  first_name VARCHAR(255) NOT NULL,
  last_name VARCHAR(255) NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE KEY
) AUTO_INCREMENT=1001;
```

值的架构，它描述了值的有效负载的结构。 更改事件的值模式在连接器为特定表生成的每个更改事件中都是相同的。

### 3. Create 事件



### 4. Update 事件

## 5. 主键更新

## 6. Delete 事件

## 7. Tombstone 事件

原文:[Data change events](https://debezium.io/documentation/reference/1.8/connectors/mysql.html#mysql-events)
