---
layout: post
author: smartsi
title: Flink 如何解析与传递参数
date: 2020-10-31 18:53:01
tags:
  - Flink

categories: Flink
permalink: parse-and-pass-arguments-in-flink
---

几乎所有的 Flink 应用程序（包括批处理与流处理程序）都需要依赖外部配置参数。例如，可以用来指定输入和输出源(如路径或者地址)，系统参数(并发数，运行时配置)以及应用程序特定参数（通常用在自定义函数中）。

从 0.9 版本开始，Flink 提供了一个叫 ParameterTool 的简单程序，提供一些基础的工具来解决上述问题，当然你也可以不用这里描述的 ParameterTool，你可以使用其他框架，例如，[Commons CLI](https://commons.apache.org/proper/commons-cli/)、[argparse4j](http://argparse4j.sourceforge.net/) 在 Flink 中也是支持的。

### 1. 解析参数

下面我们看一下如何获取配置并导入 ParameterTool 中。ParameterTool 提供了一系列预定义的静态方法来读取配置信息，ParameterTool 内部存储是一个 Map<String, String> 数据结构，因此很容易与你自己的配置相集成。

#### 1.1 读取.properties文件参数

下面方法将去读取一个本地 Properties 文件，并返回 key/value 键值对：
```java
String propertiesFile = "/Users/wy/study/dev.properties";
ParameterTool parameter = ParameterTool.fromPropertiesFile(propertiesFile);
```

#### 1.2 读取命令行参数

下面会从命令行中获取像 `--input hdfs:///mydata --elements 42` 这种形式的参数：
```java
public static void main(String[] args) {
    ParameterTool parameter = ParameterTool.fromArgs(args);
}
```

#### 1.3 从系统属性中获取参数

当启动一个 JVM 时，你可以将系统属性传递给它：`-Dinput=hdfs:///mydata`，你还可以用这些系统属性来初始化 ParameterTool：
```java
ParameterTool parameter = ParameterTool.fromSystemProperties();
```

#### 1.4 使用参数

我们已经将参数放在了 ParameterTool 对象中，那现在我们如何从 ParameterTool 对象中获取参数呢？ParameterTool 提供了一些内置的方法以供我们访问：
```java
ParameterTool parameters = // ...
parameter.getRequired("input");
parameter.get("output", "myDefaultValue");
parameter.getLong("expectedCount", -1L);
parameter.getNumberOfParameters()
```
我们可以直接在 main() 方法中使用这些方法的返回值。例如，我们可以像如下设置算子的并行度：
```java
ParameterTool parameters = ParameterTool.fromArgs(args);
int parallelism = parameters.get("mapParallelism", 2);
DataSet<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer()).setParallelism(parallelism);
```
### 2. 传递参数

在数据处理的过程中，往往需要给函数传递一些参数，那下面看看有哪些方法可以进行参数的传递？

#### 2.1 使用Configuration

我们可以通过 Configuration 对象为函数传递参数。如下显示将 Configuration 作为配置对象传递给用户定义的函数：
```java
ParameterTool parameters = ParameterTool.fromArgs(args);
Configuration conf = parameters.getConfiguration();
DataSet<Tuple2<String, Integer>> counts = text
        .flatMap(new Tokenizer())
        .withParameters(conf);
```
在 Tokenizer 中，我们可以通过 open(Configuration conf) 方法访问传递过来的参数：
```java
public static final class Tokenizer extends RichFlatMapFunction<String, Tuple2<String, Integer>> {
    @Override
    public void open(Configuration conf) throws Exception {
      // 获取参数
	    conf.getInteger('xxx', -1);
    }
}
```
如果我们不需要使用 ParameterTool 来解析参数，我们可以直接定义 Configuration：
```java
Configuration conf = new Configuration();
conf.setString("xxx", "xxx");
DataSet<Tuple2<String, Integer>> counts = text
        .flatMap(new Tokenizer())
        .withParameters(conf);
```

#### 2.2 注册全局参数

除了上述方法之外，我们还可以在 ExecutionConfig 中将参数注册为全局作业参数，可以在 JobManager 的 WEB 界面或者用户自定义函数中访问配置值。如下代码所示注册全局参数：
```java
ParameterTool parameters = ParameterTool.fromArgs(args);
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setGlobalJobParameters(parameters);
```
可以在任意 Rich 用户函数中访问参数：
```java
public static final class Tokenizer extends RichFlatMapFunction<String, Tuple2<String, Integer>> {
    @Override
    public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
      ParameterTool parameters = (ParameterTool) getRuntimeContext().getExecutionConfig().getGlobalJobParameters();
	    parameters.getRequired('xxx');
    }
}
```
