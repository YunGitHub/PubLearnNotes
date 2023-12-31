---
layout: post
author: sjf0115
title: Spark 入门 如何使用累加器 Accumulator
date: 2018-06-04 20:28:01
tags:
  - Spark
  - Spark 基础

categories: Spark
permalink: spark-base-how-to-use-accumulator
---

Accumulator 是 Spark 提供的用来实现计数器或者求和的累加器。Spark 本身支持数字类型的累加器，此外还可以添加对新类型的支持。

### 1. 内置累加器

在 Spark 2.0.0 版本之前，我们可以通过调用 `SparkContext.intAccumulator()` 或 `SparkContext.doubleAccumulator()` 来创建一个 Int 或 Double 类型的累加器：
```java
Accumulator<Double> doubleAccumulator = sparkContext.doubleAccumulator(0.0, "Double Accumulator");
Accumulator<Integer> intAccumulator = sparkContext.intAccumulator(0, "Int Accumulator");
Accumulator<Double> doubleAccumulator2 = sparkContext.accumulator(0.0, "Double Accumulator 2");
Accumulator<Integer> intAccumulator2 = sparkContext.accumulator(0, "Int Accumulator 2");
```

在 Spark 2.0.0 之后的版本中，之前的的 Accumulator 已被废除需要用 AccumulatorV2 代替:
```
@deprecated("use AccumulatorV2", "2.0.0")
class Accumulator[T] private[spark] (
    // SI-8813: This must explicitly be a private val, or else scala 2.11 doesn't compile
    @transient private val initialValue: T,
    param: AccumulatorParam[T],
    name: Option[String] = None,
    countFailedValues: Boolean = false)
  extends Accumulable[T, T](initialValue, param, name, countFailedValues)

// Int
@deprecated("use sc().longAccumulator(String)", "2.0.0")
def intAccumulator(initialValue: Int, name: String): Accumulator[java.lang.Integer] =
  sc.accumulator(initialValue, name)(IntAccumulatorParam)
    .asInstanceOf[Accumulator[java.lang.Integer]]

@deprecated("use sc().longAccumulator(String)", "2.0.0")
def accumulator(initialValue: Int, name: String): Accumulator[java.lang.Integer] =
  intAccumulator(initialValue, name)

// Double
@deprecated("use sc().doubleAccumulator(String)", "2.0.0")
def doubleAccumulator(initialValue: Double, name: String): Accumulator[java.lang.Double] =
  sc.accumulator(initialValue, name)(DoubleAccumulatorParam)
    .asInstanceOf[Accumulator[java.lang.Double]]

@deprecated("use sc().doubleAccumulator(String)", "2.0.0")
def accumulator(initialValue: Double, name: String): Accumulator[java.lang.Double] =
  doubleAccumulator(initialValue, name)    
```
通过 AccumulatorV2 方式创建累加器需要调用 `sparkContext.sc().longAccumulator()` 或 `sparkContext.sc().doubleAccumulator()` 方法来创建，如下所示创建一个 Long 和 Double 类型的累加器：
```java
DoubleAccumulator doubleAccumulator = sparkContext.sc().doubleAccumulator("Double Accumulator");
LongAccumulator longAccumulator = sparkContext.sc().longAccumulator("Long Accumulator");
```
上述两个方法的具体实现如下所示：
```java
public LongAccumulator longAccumulator() {
    LongAccumulator acc = new LongAccumulator();
    this.register(acc);
    return acc;
}

public LongAccumulator longAccumulator(final String name) {
    LongAccumulator acc = new LongAccumulator();
    this.register(acc, name);
    return acc;
}

public DoubleAccumulator doubleAccumulator() {
    DoubleAccumulator acc = new DoubleAccumulator();
    this.register(acc);
    return acc;
}

public DoubleAccumulator doubleAccumulator(final String name) {
    DoubleAccumulator acc = new DoubleAccumulator();
    this.register(acc, name);
    return acc;
}
```

通过源码我们知道分别通过创建 `LongAccumulator` 和 `DoubleAccumulator` 对象，然后进行注册来创建一个累加器。所以我们也可以使用如下方式创建一个 Long 类型的累加器：
```java
LongAccumulator longAccumulator = new LongAccumulator();
sparkContext.sc().register(longAccumulator, "Long Accumulator");
```
> LongAccumulator DoubleAccumulator 都继承自 AccumulatorV2

Spark 内置了数值型累加器(例如，Long，Double类型)，我们还可以通过继承 AccumulatorV2 来创建我们自己类型的累加器。

### 2. 自定义累加器

自定义累加器类型的功能在 1.x 版本中就已经提供了，但是使用起来比较麻烦，在 Spark 2.0.0 版本后，累加器的易用性有了较大的改进，而且官方还提供了一个新的抽象类 AccumulatorV2 来提供更加友好的自定义类型累加器的实现方式。官方同时给出了一个实现的示例：CollectionAccumulator，这个类允许以集合的形式收集 Spark 应用执行过程中的一些信息。例如，我们可以用这个类收集 Spark 处理数据过程中的非法数据或者引起异常的异常数据，这对我们处理异常时很有帮助。当然，由于累加器的值最终要汇聚到 Driver 端，为了避免 Driver 端的出现 OOM，需要收集的数据规模不宜过大。

实现自定义类型累加器需要继承 AccumulatorV2 并覆盖下面几个方法：
- reset 将累加器重置为零
- add 将另一个值添加到累加器中
- merge 将另一个相同类型的累加器合并到该累加器中。

下面这个累加器可以用于在程序运行过程中收集一些异常或者非法数据，最终以 `List[String]` 的形式返回：
```java
public static class CollectionAccumulator<T> extends AccumulatorV2<T, List<T>> {

    private List<T> collection = Lists.newArrayList();

    @Override
    public boolean isZero() {
        return collection.isEmpty();
    }

    @Override
    public AccumulatorV2<T, List<T>> copy() {
        CollectionAccumulator<T> accumulator = new CollectionAccumulator<>();
        synchronized (accumulator) {
            accumulator.collection.addAll(collection);
        }
        return accumulator;
    }

    @Override
    public void reset() {
        collection.clear();
    }

    @Override
    public void add(T v) {
        collection.add(v);
    }

    @Override
    public void merge(AccumulatorV2<T, List<T>> other) {
        if(other instanceof CollectionAccumulator){
            collection.addAll(((CollectionAccumulator) other).collection);
        }
        else {
            throw new UnsupportedOperationException("Cannot merge " + this.getClass().getName() + " with " + other.getClass().getName());
        }
    }

    @Override
    public List<T> value() {
        return new ArrayList<>(collection);
    }
}
```
下面我们在数据处理过程中收集非法坐标为例，来看一下我们自定义的累加器如何使用:
```java
public static void main(String[] args) {
    String appName = "CustomAccumulatorExample";
    SparkConf conf = new SparkConf().setAppName(appName).setMaster("local[*]");
    JavaSparkContext sparkContext = new JavaSparkContext(conf);

    List<String> list = Lists.newArrayList();
    list.add("27.34832,111.32135");
    list.add("34.88478,185.17841");
    list.add("39.92378,119.50802");
    list.add("94,119.50802");

    // 创建累加器
    CollectionAccumulator<String> collectionAccumulator = new CollectionAccumulator<>();
    // 注册累加器
    sparkContext.sc().register(collectionAccumulator, "Illegal Coordinates");
    // 过滤非法坐标
    List<String> collect = sparkContext.parallelize(list)
            .filter(new Function<String, Boolean>() {
                @Override
                public Boolean call(String str) throws Exception {
                    String[] coordinate = str.split(",");
                    double lat = Double.parseDouble(coordinate[0]);
                    double lon = Double.parseDouble(coordinate[1]);
                    if (Math.abs(lat) > 90 || Math.abs(lon) > 180) {
                        collectionAccumulator.add(str);
                        return true;
                    }
                    return false;
                }
            }).collect();

    System.out.println("非法坐标采集：" + collect.toString());
    System.out.println("非法坐标累加器记录" + collectionAccumulator.value());
}
```

### 3. 累加器注意事项

累加器不会改变 Spark 的懒加载（Lazy）的执行模型。如果在 RDD 上的某个操作中更新累加器，那么其值只会在 RDD 执行 action 计算时被更新一次。因此，在 transformation （例如， `map()`）中更新累加器时，其值并不能保证一定被更新。

Spark 中的一系列 transformation 操作会构成一个任务链，需要通过 action 操作来触发。累加器也是一样的，也只能通过 action 触发更新，所以在 action 操作之前调用 value 方法查看其数值是没有任何变化的。对于在 action 中更新的累加器，Spark 会保证每个任务对累加器只更新一次，即使重新启动的任务也不会重新更新该值。而如果在 transformation 中更新的累加器，如果任务或作业 stage 被重新执行，那么其对累加器的更新可能会执行多次。

```java
public class AccumulatorTrap {
    public static void main(String[] args) {
        String appName = "AccumulatorTrap";
        SparkConf conf = new SparkConf().setAppName(appName).setMaster("local[*]");
        JavaSparkContext sparkContext = new JavaSparkContext(conf);

        LongAccumulator evenAccumulator = sparkContext.sc().longAccumulator("Even Num Accumulator");
        LongAccumulator oddAccumulator = sparkContext.sc().longAccumulator("Odd Num Accumulator");

        List<Integer> numList = Lists.newArrayList();
        for(int i = 0;i < 10;i++){
            numList.add(i);
        }

        JavaRDD<Integer> resultRDD = sparkContext.parallelize(numList)
                // transform
                .map(new Function<Integer, Integer>() {
                    @Override
                    public Integer call(Integer num) throws Exception {
                        if (num % 2 == 0) {
                            evenAccumulator.add(1L);
                            return 0;
                        } else {
                            oddAccumulator.add(1L);
                            return 1;
                        }
                    }
                });

        // the first action
        resultRDD.count();
        System.out.println("Odd Num Count : " + oddAccumulator.value());
        System.out.println("Even Num Count : " + evenAccumulator.value());

        // the second action
        resultRDD.foreach(new VoidFunction<Integer>() {
            @Override
            public void call(Integer num) throws Exception {
                System.out.println(num);
            }
        });
        System.out.println("Odd Num Count : " + oddAccumulator.value());
        System.out.println("Even Num Count : " + evenAccumulator.value());
    }
}
```

在第一个 action 算子 count 执行之后，累加器输出符合我们预期的结果：
```
Odd Num Count : 5
Even Num Count : 5
```
在第二个 action 算子 foreach 执行之后，累加器输出结果如下：
```
Odd Num Count : 10
Even Num Count : 10
```
其实这个时候又执行了一次 map 操作，所以累加器各自又增加了5，最终获得的结果变成了10。

看了上面的分析以及输出结果，我们知道，那就是使用累加器的过程中只能使用一次 action 操作才能保证结果的准确性。事实上，这种情况是可以解决的，只要将任务之间的依赖关系切断就可以。我们可以调用 cache，persist 等方法将之前的依赖切断，后续的累加器就不会受之前的 transfrom 操作影响了：
```
Odd Num Count : 5
Even Num Count : 5
Odd Num Count : 5
Even Num Count : 5
```
所以在使用累加器时，为了保证准确性，最好只使用一次 action 操作。如果需要使用多次，可以使用 cache 或 persist 操作切断依赖。
