---
title: Flink Time
date: "2020-04-11 11:51:00"
categories:
- 大数据
- Flink
tags:
- 大数据
- Flink
toc: true
typora-root-url: ..\..\..
---

## 时间（Time）

### 时间类型

- Flink中的时间与现实世界中的时间是不一致的，在flink中被划分为**事件时间，摄入时间，处理时间**三种。
- 如果以EventTime为基准来定义时间窗口将形成EventTimeWindow,要求消息本身就应该携带EventTime
- 如果以IngesingtTime为基准来定义时间窗口将形成IngestingTimeWindow,以source的systemTime为准。
- 如果以ProcessingTime基准来定义时间窗口将形成ProcessingTimeWindow，以operator的systemTime为准。

![1586577084774](/img/1586577084774.png)

### 时间详解

**Processing Time**

Processing Time 是指事件被处理时机器的系统时间。

当流程序在 Processing Time 上运行时，所有基于时间的操作(如时间窗口)将使用当时机器的系统时间。每小时 Processing Time 窗口将包括在系统时钟指示整个小时之间到达特定操作的所有事件。

例如，如果应用程序在上午 9:15 开始运行，则第一个每小时 Processing Time 窗口将包括在上午 9:15 到上午 10:00 之间处理的事件，下一个窗口将包括在上午 10:00 到 11:00 之间处理的事件。

Processing Time 是最简单的 “Time” 概念，不需要流和机器之间的协调，它提供了最好的性能和最低的延迟。但是，在分布式和异步的环境下，Processing Time 不能提供确定性，因为它容易受到事件到达系统的速度（例如从消息队列）、事件在系统内操作流动的速度以及中断的影响。

**Event Time**

Event Time 是事件发生的时间，一般就是数据本身携带的时间。这个时间通常是在事件到达 Flink 之前就确定的，并且可以从每个事件中获取到事件时间戳。在 Event Time 中，时间取决于数据，而跟其他没什么关系。Event Time 程序必须指定如何生成 Event Time 水印，这是表示 Event Time 进度的机制。

完美的说，无论事件什么时候到达或者其怎么排序，最后处理 Event Time 将产生完全一致和确定的结果。但是，除非事件按照已知顺序（按照事件的时间）到达，否则处理 Event Time 时将会因为要等待一些无序事件而产生一些延迟。由于只能等待一段有限的时间，因此就难以保证处理 Event Time 将产生完全一致和确定的结果。

假设所有数据都已到达， Event Time 操作将按照预期运行，即使在处理无序事件、延迟事件、重新处理历史数据时也会产生正确且一致的结果。 例如，每小时事件时间窗口将包含带有落入该小时的事件时间戳的所有记录，无论它们到达的顺序如何。

请注意，有时当 Event Time 程序实时处理实时数据时，它们将使用一些 Processing Time 操作，以确保它们及时进行。

**Ingestion Time**

Ingestion Time 是事件进入 Flink 的时间。 在源操作处，每个事件将源的当前时间作为时间戳，并且基于时间的操作（如时间窗口）会利用这个时间戳。

Ingestion Time 在概念上位于 Event Time 和 Processing Time 之间。 与 Processing Time 相比，它稍微贵一些，但结果更可预测。因为 Ingestion Time 使用稳定的时间戳（在源处分配一次），所以对事件的不同窗口操作将引用相同的时间戳，而在 Processing Time 中，每个窗口操作符可以将事件分配给不同的窗口（基于机器系统时间和到达延迟）。

与 Event Time 相比，Ingestion Time 程序无法处理任何无序事件或延迟数据，但程序不必指定如何生成水印。

在 Flink 中，，Ingestion Time 与 Event Time 非常相似，但 Ingestion Time 具有自动分配时间戳和自动生成水印功能。

## 时间戳和水印

在介绍Watermark相关内容之前我们先抛出一个具体的问题，在实际的流式计算中数据到来的顺序对计算结果的正确性有至关重要的影响，比如：某数据源中的某些数据由于某种原因(如：网络原因，外部存储自身原因)会有5秒的延时，也就是在实际时间的第1秒产生的数据有可能在第5秒中产生的数据之后到来(比如到Window处理节点).选具体某个delay的元素来说，假设在一个5秒的Tumble窗口(详见Window介绍章节)，有一个EventTime是 11秒的数据，在第16秒时候到来了。图示第11秒的数据，在16秒到来了，如下图：

![1586589440002](/img/1586589440002.png)

那么对于一个Count聚合的Tumble(5s)的window，上面的情况如何处理才能window2=4，window3=2 呢？Apache Flink的时间类型
开篇我们描述的问题是一个很常见的TimeWindow中数据乱序的问题，乱序是相对于事件产生时间和到达Apache Flink 实际处理算子的顺序而言的，关于时间在Apache Flink中有如下三种时间类型，如下图：

![1586589447529](/img/1586589447529.png)

那么对于一个Count聚合的Tumble(5s)的window，上面的情况如何处理才能window2=4，window3=2 呢？

### Apache Flink的时间类型

开篇我们描述的问题是一个很常见的TimeWindow中数据乱序的问题，乱序是相对于事件产生时间和到达Apache Flink 实际处理算子的顺序而言的，关于时间在Apache Flink中有如下三种时间类型，如下图：

![1586589454924](/img/1586589454924.png)

- ProcessingTime 

  是数据流入到具体某个算子时候相应的系统时间。ProcessingTime 有最好的性能和最低的延迟。但在分布式计算环境中ProcessingTime具有不确定性，相同数据流多次运行有可能产生不同的计算结果。

- IngestionTime

  IngestionTime是数据进入Apache Flink框架的时间，是在Source Operator中设置的。与ProcessingTime相比可以提供更可预测的结果，因为IngestionTime的时间戳比较稳定(在源处只记录一次)，同一数据在流经不同窗口操作时将使用相同的时间戳，而对于ProcessingTime同一数据在流经不同窗口算子会有不同的处理时间戳。

- EventTime

  EventTime是事件在设备上产生时候携带的。在进入Apache Flink框架之前EventTime通常要嵌入到记录中，并且EventTime也可以从记录中提取出来。在实际的网上购物订单等业务场景中，大多会使用EventTime来进行数据计算。

  开篇描述的问题和本篇要介绍的Watermark所涉及的时间类型均是指EventTime类型。

### 什么是Watermark

Watermark是Apache Flink为了处理EventTime 窗口计算提出的一种机制,本质上也是一种时间戳，由Apache Flink Source或者自定义的Watermark生成器按照需求Punctuated或者Periodic两种方式生成的一种系统Event，与普通数据流Event一样流转到对应的下游算子，接收到Watermark Event的算子以此不断调整自己管理的EventTime clock。 Apache Flink 框架保证Watermark单调递增，算子接收到一个Watermark时候，框架知道不会再有任何小于该Watermark的时间戳的数据元素到来了，所以Watermark可以看做是告诉Apache Flink框架数据流已经处理到什么位置(时间维度)的方式。 Watermark的产生和Apache Flink内部处理逻辑如下图所示: 

![1586589616212](/img/1586589616212.png)

## Watermark的产生方式

目前Apache Flink 有两种生产Watermark的方式，如下：

- Punctuated - 数据流中每一个递增的EventTime都会产生一个Watermark。 

在实际的生产中Punctuated方式在TPS很高的场景下会产生大量的Watermark在一定程度上对下游算子造成压力，所以只有在实时性要求非常高的场景才会选择Punctuated的方式进行Watermark的生成。

- Periodic - 周期性的（一定时间间隔或者达到一定的记录条数）产生一个Watermark。在实际的生产中Periodic的方式必须结合时间和积累条数两个维度继续周期性产生Watermark，否则在极端情况下会有很大的延时。

所以Watermark的生成方式需要根据业务场景的不同进行不同的选择。

### Watermark的接口定义

对应Apache Flink Watermark两种不同的生成方式，我们了解一下对应的接口定义，如下：

- Periodic Watermarks - AssignerWithPeriodicWatermarks

```
/**
* Returns the current watermark. This method is periodically called by the
* system to retrieve the current watermark. The method may return {@code null} to
* indicate that no new Watermark is available.
*
* The returned watermark will be emitted only if it is non-null and itsTimestamp
* is larger than that of the previously emitted watermark (to preserve the contract of
* ascending watermarks). If the current watermark is still
* identical to the previous one, no progress in EventTime has happened since
* the previous call to this method. If a null value is returned, or theTimestamp
* of the returned watermark is smaller than that of the last emitted one, then no
* new watermark will be generated.
*
* The interval in which this method is called and Watermarks are generated
* depends on {@link ExecutionConfig#getAutoWatermarkInterval()}.
*
* @see org.Apache.flink.streaming.api.watermark.Watermark
* @see ExecutionConfig#getAutoWatermarkInterval()
*
* @return {@code Null}, if no watermark should be emitted, or the next watermark to emit.
*/
@Nullable
Watermark getCurrentWatermark();
```

- Punctuated Watermarks - AssignerWithPunctuatedWatermarks 

```
public interface AssignerWithPunctuatedWatermarks<T> extends TimestampAssigner<T> {

/**
* Asks this implementation if it wants to emit a watermark. This method is called right after
* the {@link #extractTimestamp(Object, long)} method.
*
* The returned watermark will be emitted only if it is non-null and itsTimestamp
* is larger than that of the previously emitted watermark (to preserve the contract of
* ascending watermarks). If a null value is returned, or theTimestamp of the returned
* watermark is smaller than that of the last emitted one, then no new watermark will
* be generated.
*
* For an example how to use this method, see the documentation of
* {@link AssignerWithPunctuatedWatermarks this class}.
*
* @return {@code Null}, if no watermark should be emitted, or the next watermark to emit.
*/
@Nullable
Watermark checkAndGetNextWatermark(T lastElement, long extractedTimestamp);
}
```

AssignerWithPunctuatedWatermarks 继承了TimestampAssigner接口 -TimestampAssigner

```
public interfaceTimestampAssigner<T> extends Function {

/**
* Assigns aTimestamp to an element, in milliseconds since the Epoch.
*
* The method is passed the previously assignedTimestamp of the element.
* That previousTimestamp may have been assigned from a previous assigner,
* by ingestionTime. If the element did not carry aTimestamp before, this value is
* {@code Long.MIN_VALUE}.
*
* @param element The element that theTimestamp is wil be assigned to.
* @param previousElementTimestamp The previous internalTimestamp of the element,
* or a negative value, if noTimestamp has been assigned, yet.
* @return The newTimestamp.
*/
long extractTimestamp(T element, long previousElementTimestamp);
}
```

从接口定义可以看出，Watermark可以在Event(Element)中提取EventTime，进而定义一定的计算逻辑产生Watermark的时间戳。

## Watermark解决如上问题

从上面的Watermark生成接口和Apache Flink内部对Periodic Watermark的实现来看，Watermark的时间戳可以和Event中的EventTime 一致，也可以自己定义任何合理的逻辑使得Watermark的时间戳不等于Event中的EventTime，Event中的EventTime自产生那一刻起就不可以改变了，不受Apache Flink框架控制，而Watermark的产生是在Apache Flink的Source节点或实现的Watermark生成器计算产生(如上Apache Flink内置的 Periodic Watermark实现), Apache Flink内部对单流或多流的场景有统一的Watermark处理。

回过头来我们在看看Watermark机制如何解决上面的问题，上面的问题在于如何将迟来的EventTime 位11的元素正确处理。要解决这个问题我们还需要先了解一下EventTime window是如何触发的？ EventTime window 计算条件是当Window计算的Timer时间戳 小于等于 当前系统的Watermak的时间戳时候进行计算。 

- 当Watermark的时间戳等于Event中携带的EventTime时候，上面场景（Watermark=EventTime)的计算结果如下：

![1586590446371](/img/1586590446371.png)

 上面对应的DDL(Alibaba 企业版的Flink分支)定义如下：

```
CREATE TABLE source(
...,
Event_timeTimeStamp,
WATERMARK wk1 FOR Event_time as withOffset(Event_time, 0) 
) with (
...
);
```

- 如果想正确处理迟来的数据可以定义Watermark生成策略为 Watermark = EventTime -5s， 如下：

![1586590568191](/img/1586590568191.png)

上面对应的DDL(Alibaba 内部的DDL语法，目前正在和社区讨论)定义如下： 

```
CREATE TABLE source(
...,
Event_timeTimeStamp,
WATERMARK wk1 FOR Event_time as withOffset(Event_time, 5000) 
) with (
...
);
```

上面正确处理的根源是我们采取了 延迟触发 window 计算 的方式正确处理了 Late Event. 与此同时，我们发现window的延时触发计算，也导致了下游的LATENCY变大，本例子中下游得到window的结果就延迟了5s.

## 多流的Watermark处理

在实际的流计算中往往一个job中会处理多个Source的数据，对Source的数据进行GroupBy分组，那么来自不同Source的相同key值会shuffle到同一个处理节点，并携带各自的Watermark，Apache Flink内部要保证Watermark要保持单调递增，多个Source的Watermark汇聚到一起时候可能不是单调自增的，这样的情况Apache Flink内部是如何处理的呢？如下图所示：

![1586591065285](/img/1586591065285.png)

Apache Flink内部实现每一个边上只能有一个递增的Watermark， 当出现多流携带Eventtime汇聚到一起(GroupBy or Union)时候，Apache Flink会选择所有流入的Eventtime中最小的一个向下游流出。从而保证watermark的单调递增和保证数据的完整性.如下图: 

![1586591258097](/img/1586591258097.png)

本节以一个流计算常见的乱序问题介绍了Apache Flink如何利用Watermark机制来处理乱序问题. 本篇内容在一定程度上也体现了EventTime Window中的Trigger机制依赖了Watermark(后续Window篇章会介绍)。Watermark机制是流计算中处理乱序，正确处理Late Event的核心手段。

## 使用Event Time 处理实时数据

> 如下是一段 log 日志，我们根据该日志格式，来分析客户的下单量情况。
>
> 日志格式：
> 1581490623000,James,5
> 1581490624150,John,2
> …
> 接下来，我们从并行Source 和 非并行Source 两个方向，来使用 EventTime 处理实时数据。(接下来示例，设置延迟为0s，即不延迟)

### 非并行Source
非并行Source，以 socketTextStream为例来介绍 Flink使用 EventTime 处理实时数据。

**代码**
```java
/**
* TODO 非并行Source EventTime 
*

* @author liuzebiao
 
* @Date 2020-2-12 15:25
*/
public class EventTimeDemo {

    public static void main(String[] args) throws Exception {

    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    
    //设置EventTime作为时间标准
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
    
    //读取source，并指定(1581490623000,Mary,3)中哪个字段为EventTime时间
    //WaterMarks:是Flink中窗口延迟触发的机制。Time.seconds(0)表示无延迟。
    SingleOutputStreamOperator<String> source = env.socketTextStream("localhost", 8888).assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<String>(Time.seconds(0)) {
        @Override
        public long extractTimestamp(String line) {
            String[] split = line.split(",");
            return Long.parseLong(split[0]);
        }
    });
    
    SingleOutputStreamOperator<Tuple2<String, Integer>> mapOperator = source.map(line -> {
        String[] split = line.split(",");
        return Tuple2.of(split[1], Integer.parseInt(split[2]));
    }).returns(Types.TUPLE(Types.STRING,Types.INT));
    
    KeyedStream<Tuple2<String, Integer>, Tuple> keyedStream = mapOperator.keyBy(0);
    //EventTime滚动窗口
    WindowedStream<Tuple2<String, Integer>, Tuple, TimeWindow> windowedStream = keyedStream.window(TumblingEventTimeWindows.of(Time.seconds(5)));
    
    SingleOutputStreamOperator<Tuple2<String, Integer>> sum = windowedStream.sum(1);
    
    sum.print();
    
    env.execute("EventTimeDemo");
    }
}
```

**测试结果**

![1586583564019](/img/1586583564019.png)

> **结果分析**
> 备注： (1581490623000转换后为：2020-02-12 14:57:03
> ​             1581490624000转换后为：2020-02-12 14:57:04)
>
> 当我们在Socket中输入如下数据：
> 1581490623000,Mary,2
> 1581490624000,John,3
> 1581490624500,Clerk,1
> 1581490624998,Maria,4
> 1581490624999,Mary,3
> 1581490626000,Mary,3
> 1581490630800,Steve,3     (2020-02-12 14:57:10.800)
>
> 窗口定义的时间是：含头不含尾。即：[0,5)，
> 图片解析：(我们定义滚动窗口为5s，我们分析图片发现到4998时,并没有输出内容。因为4998还没超过5s，窗口规定是>=临界值时触发，所以当我们输入4999临界时，我们发现输出内容了，说明一个窗口滚动完成，输出内容包含4999这个时间的值；当输入6000时，6000在[5,10)之间没有>10，所以不输出。输入30800【2020-02-12 14:57:10.800)】，已经超过10s，所以结果只输出1个 (Mary,3)，因为Steve已经被分到另一个窗口了)
>
> 还有一个问题，就是：当输入到 4999 时，只是Mary这个分组满足5s这个条件，但是其它分组John，Clerk 等也同步输出结果了。显然这不符合逻辑。为什么会出现这种情况呢？是因为SocketStream 是非并行数据流，所以才会出现这样子的结果。(接下来我们就是用并行数据流KafkaSource来分析)
>

### 并行Source
并行Source，以 KafkaSouce 为例来介绍 Flink使用 EventTime 处理实时数据。

**代码**

并行KafkaSource EventTime示例(读取 topic为 window_demo中的消息)，代码如下所示：

```java
/**
* TODO 并行KafkaSource EventTime示例(读取 topic为 window_demo中的消息)
*
* @author liuzebiao
* @Date 2020-2-12 15:25
*/
public class EventTimeDemo {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        
        //设置EventTime作为时间标准
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        
        //Kafka props
        Properties properties = new Properties();
        //指定Kafka的Broker地址
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.204.210:9092,192.168.204.211:9092,192.168.204.212:9092");
        //指定组ID
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "flinkDemoGroup");
        //如果没有记录偏移量，第一次从最开始消费
        properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        
        FlinkKafkaConsumer<String> kafkaSource = new FlinkKafkaConsumer("window_demo", new SimpleStringSchema(), properties);
        
        //2.通过addSource()方式，创建 Kafka DataStream
        //读取source，并指定(1581490623000,Mary,3)中哪个字段为EventTime时间
        SingleOutputStreamOperator<String> source = env.addSource(kafkaSource).assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<String>(Time.seconds(0)) {
            @Override
            public long extractTimestamp(String line) {
                String[] split = line.split(",");
                return Long.parseLong(split[0]);
            }
        });
        
        SingleOutputStreamOperator<Tuple2<String, Integer>> mapOperator = source.map(line -> {
            String[] split = line.split(",");
            return Tuple2.of(split[1], Integer.parseInt(split[2]));
        }).returns(Types.TUPLE(Types.STRING,Types.INT));
        
        KeyedStream<Tuple2<String, Integer>, Tuple> keyedStream = mapOperator.keyBy(0);
        //EventTime滚动窗口
        WindowedStream<Tuple2<String, Integer>, Tuple, TimeWindow> windowedStream = keyedStream.window(TumblingEventTimeWindows.of(Time.seconds(5)));
        
        SingleOutputStreamOperator<Tuple2<String, Integer>> sum = windowedStream.sum(1);
        
        sum.print();
        
        env.execute("EventTimeDemo");
    }
}
```

**测试结果**

1. 创建 Topic 命令如下：

   ```shell
   bin/kafka-topics.sh --create --zookeeper 192.168.204.210:2181,192.168.204.211:2181,192.168.204.212:2181 --replication-factor 1 --partitions 3 --topic window_demo
   # (特别注意一下：此处创建了3个分区)
   ```

2. 创建 Topic 成功截图(点击放大查看)：

   ![1586583578404](/img/1586583578404.png)

3. 使用命令，写入数据到Kafka：

   ```shell
   bin/kafka-console-producer.sh --broker-list 192.168.204.210:9092 --topic window_demo
   ```

   使用命令写入以下数据：

   ```shell
   1581490623000,Mary,2
   1581490624000,John,3
   1581490624500,Clerk,1
   1581490624998,Maria,4
   1581490624999,Mary,3
   ```

4. 测试结果：

   ![1586583590046](/img/1586583590046.png)

### 结果分析
​	在并行Source一例中，当我们输入1581490624999,Mary,3时，我们看到控制台会直接帮我们输出计算结果。

​	但是，在使用 KafkaSource 时，我们连续输入了 3次1581490624999,Mary,3，我们才看到控制台帮我们输出计算了结果。

​	那这是为什么呢？这是 并行Source 和 非并行Source 的原因导致的（这里涉及到 KafkaSource 创建的 topic，有 3 个分区的原因，如下图所示）

![1586584999171](/img/1586584999171.png)