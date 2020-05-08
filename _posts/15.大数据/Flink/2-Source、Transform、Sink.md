---
title: Source、Transform、Sink
date: "2020-04-09 11:00:00"
categories:
- 大数据
- Flink
tags:
- 大数据
- Flink
toc: true
typora-root-url: ..\..\..
---

# Data Source

## 一、内置 Data Source

Flink Data Source 用于定义 Flink 程序的数据来源，Flink 官方提供了多种数据获取方法，用于帮助开发者简单快速地构建输入流，具体如下：

### 1.1 基于文件构建

**1. readTextFile(path)**：按照 TextInputFormat 格式读取文本文件，并将其内容以字符串的形式返回。示例如下：

```java
env.readTextFile(filePath).print();
```

**2. readFile(fileInputFormat, path)** ：按照指定格式读取文件。

**3. readFile(inputFormat, filePath, watchType, interval, typeInformation)**：按照指定格式周期性的读取文件。其中各个参数的含义如下：

- **inputFormat**：数据流的输入格式。
- **filePath**：文件路径，可以是本地文件系统上的路径，也可以是 HDFS 上的文件路径。
- **watchType**：读取方式，它有两个可选值，分别是 `FileProcessingMode.PROCESS_ONCE` 和 `FileProcessingMode.PROCESS_CONTINUOUSLY`：前者表示对指定路径上的数据只读取一次，然后退出；后者表示对路径进行定期地扫描和读取。需要注意的是如果 watchType 被设置为 `PROCESS_CONTINUOUSLY`，那么当文件被修改时，其所有的内容 (包含原有的内容和新增的内容) 都将被重新处理，因此这会打破 Flink 的 *exactly-once* 语义。
- **interval**：定期扫描的时间间隔。
- **typeInformation**：输入流中元素的类型。

使用示例如下：

```java
final String filePath = "D:\\log4j.properties";
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.readFile(new TextInputFormat(new Path(filePath)),
             filePath,
             FileProcessingMode.PROCESS_ONCE,
             1,
             BasicTypeInfo.STRING_TYPE_INFO).print();
env.execute();
```

### 1.2 基于集合构建

**1. fromCollection(Collection)**：基于集合构建，集合中的所有元素必须是同一类型。示例如下：

```java
env.fromCollection(Arrays.asList(1,2,3,4,5)).print();
```

**2. fromElements(T ...)**： 基于元素构建，所有元素必须是同一类型。示例如下：

```java
env.fromElements(1,2,3,4,5).print();
```

**3. generateSequence(from, to)**：基于给定的序列区间进行构建。示例如下：

```java
env.generateSequence(0,100);
```

**4. fromCollection(Iterator, Class)**：基于迭代器进行构建。第一个参数用于定义迭代器，第二个参数用于定义输出元素的类型。使用示例如下：

```java
env.fromCollection(new CustomIterator(), BasicTypeInfo.INT_TYPE_INFO).print();
```

其中 CustomIterator 为自定义的迭代器，这里以产生 1 到 100 区间内的数据为例，源码如下。需要注意的是自定义迭代器除了要实现 Iterator 接口外，还必须要实现序列化接口 Serializable ，否则会抛出序列化失败的异常：

```java
import java.io.Serializable;
import java.util.Iterator;

public class CustomIterator implements Iterator<Integer>, Serializable {
    private Integer i = 0;

    @Override
    public boolean hasNext() {
        return i < 100;
    }

    @Override
    public Integer next() {
        i++;
        return i;
    }
}
```

**5. fromParallelCollection(SplittableIterator, Class)**：方法接收两个参数，第二个参数用于定义输出元素的类型，第一个参数 SplittableIterator 是迭代器的抽象基类，它用于将原始迭代器的值拆分到多个不相交的迭代器中。

### 1.3  基于 Socket 构建

Flink 提供了 socketTextStream 方法用于构建基于 Socket 的数据流，socketTextStream 方法有以下四个主要参数：

- **hostname**：主机名；
- **port**：端口号，设置为 0 时，表示端口号自动分配；
- **delimiter**：用于分隔每条记录的分隔符；
- **maxRetry**：当 Socket 临时关闭时，程序的最大重试间隔，单位为秒。设置为 0 时表示不进行重试；设置为负值则表示一直重试。示例如下：

```shell
 env.socketTextStream("192.168.0.229", 9999, "\n", 3).print();
```

## 二、自定义 Data Source

### 2.1 SourceFunction

除了内置的数据源外，用户还可以使用 `addSource` 方法来添加自定义的数据源。自定义的数据源必须要实现 SourceFunction 接口，这里以产生 [0 , 1000) 区间内的数据为例，代码如下：

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.addSource(new SourceFunction<Long>() {
    
    private long count = 0L;
    private volatile boolean isRunning = true;

    public void run(SourceContext<Long> ctx) {
        while (isRunning && count < 1000) {
            // 通过collect将输入发送出去 
            ctx.collect(count);
            count++;
        }
    }

    public void cancel() {
        isRunning = false;
    }

}).print();
env.execute();
```

### 2.2 ParallelSourceFunction 和 RichParallelSourceFunction

上面通过 SourceFunction 实现的数据源是不具有并行度的，即不支持在得到的 DataStream 上调用 `setParallelism(n)` 方法，此时会抛出如下的异常：

```shell
Exception in thread "main" java.lang.IllegalArgumentException: Source: 1 is not a parallel source
```

如果你想要实现具有并行度的输入流，则需要实现 ParallelSourceFunction 或 RichParallelSourceFunction 接口，其与 SourceFunction 的关系如下图： 

![1586415778743](/img/1586415778743.png)

ParallelSourceFunction 直接继承自 ParallelSourceFunction，具有并行度的功能。RichParallelSourceFunction 则继承自 AbstractRichFunction，同时实现了 ParallelSourceFunction 接口，所以其除了具有并行度的功能外，还提供了额外的与生命周期相关的方法，如 open() ，closen() 。

## 三、Streaming Connectors

### 3.1 内置连接器

除了自定义数据源外， Flink 还内置了多种连接器，用于满足大多数的数据收集场景。当前内置连接器的支持情况如下：

- Apache Kafka (支持 source 和 sink)
- Apache Cassandra (sink)
- Amazon Kinesis Streams (source/sink)
- Elasticsearch (sink)
- Hadoop FileSystem (sink)
- RabbitMQ (source/sink)
- Apache NiFi (source/sink)
- Twitter Streaming API (source)
- Google PubSub (source/sink)

除了上述的连接器外，你还可以通过 Apache Bahir 的连接器扩展 Flink。Apache Bahir 旨在为分布式数据分析系统 (如 Spark，Flink) 等提供功能上的扩展，当前其支持的与 Flink 相关的连接器如下：

- Apache ActiveMQ (source/sink)
- Apache Flume (sink)
- Redis (sink)
- Akka (sink)
- Netty (source)

随着 Flink 的不断发展，可以预见到其会支持越来越多类型的连接器，关于连接器的后续发展情况，可以查看其官方文档：[Streaming Connectors]( https://ci.apache.org/projects/flink/flink-docs-release-1.9/dev/connectors/index.html) 。在所有 DataSource 连接器中，使用的广泛的就是 Kafka，所以这里我们以其为例，来介绍 Connectors 的整合步骤。

### 3.2 整合 Kakfa

#### 1. 导入依赖

整合 Kafka 时，一定要注意所使用的 Kafka 的版本，不同版本间所需的 Maven 依赖和开发时所调用的类均不相同，具体如下：

| Maven 依赖                      | Flink 版本 | Consumer and Producer 类的名称                   | Kafka 版本 |
| :------------------------------ | :--------- | :----------------------------------------------- | :--------- |
| flink-connector-kafka-0.8_2.11  | 1.0.0 +    | FlinkKafkaConsumer08 <br/>FlinkKafkaProducer08   | 0.8.x      |
| flink-connector-kafka-0.9_2.11  | 1.0.0 +    | FlinkKafkaConsumer09<br/> FlinkKafkaProducer09   | 0.9.x      |
| flink-connector-kafka-0.10_2.11 | 1.2.0 +    | FlinkKafkaConsumer010 <br/>FlinkKafkaProducer010 | 0.10.x     |
| flink-connector-kafka-0.11_2.11 | 1.4.0 +    | FlinkKafkaConsumer011 <br/>FlinkKafkaProducer011 | 0.11.x     |
| flink-connector-kafka_2.11      | 1.7.0 +    | FlinkKafkaConsumer <br/>FlinkKafkaProducer       | >= 1.0.0   |

这里我使用的 Kafka 版本为 kafka_2.12-2.2.0，添加的依赖如下： 

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka_2.11</artifactId>
    <version>1.9.0</version>
</dependency>
```

#### 2. 代码开发

这里以最简单的场景为例，接收 Kafka 上的数据并打印，代码如下：

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
Properties properties = new Properties();
// 指定Kafka的连接位置
properties.setProperty("bootstrap.servers", "hadoop001:9092");
// 指定监听的主题，并定义Kafka字节消息到Flink对象之间的转换规则
DataStream<String> stream = env
    .addSource(new FlinkKafkaConsumer<>("flink-stream-in-topic", new SimpleStringSchema(), properties));
stream.print();
env.execute("Flink Streaming");
```

### 3.3 整合测试

#### 1. 启动 Kakfa

Kafka 的运行依赖于 zookeeper，需要预先启动，可以启动 Kafka 内置的 zookeeper，也可以启动自己安装的：

```shell
# zookeeper启动命令
bin/zkServer.sh start

# 内置zookeeper启动命令
bin/zookeeper-server-start.sh config/zookeeper.properties
```

启动单节点 kafka 用于测试：

```shell
# bin/kafka-server-start.sh config/server.properties
```

#### 2. 创建 Topic

```shell
# 创建用于测试主题
bin/kafka-topics.sh --create \
                    --bootstrap-server hadoop001:9092 \
                    --replication-factor 1 \
                    --partitions 1  \
                    --topic flink-stream-in-topic

# 查看所有主题
 bin/kafka-topics.sh --list --bootstrap-server hadoop001:9092
```

#### 3. 启动 Producer

这里 启动一个 Kafka 生产者，用于发送测试数据：

```shell
bin/kafka-console-producer.sh --broker-list hadoop001:9092 --topic flink-stream-in-topic
```

#### 4. 测试结果

在 Producer 上输入任意测试数据，之后观察程序控制台的输出：

![1586416665872](/img/1586416665872.png)

程序控制台的输出如下：

![1586416679333](/img/1586416679333.png)

可以看到已经成功接收并打印出相关的数据。

# Transform

## 一、Transformations 分类

Flink 的 Transformations 操作主要用于将一个和多个 DataStream 按需转换成新的 DataStream。它主要分为以下三类：

- **DataStream Transformations**：进行数据流相关转换操作；
- **Physical partitioning**：物理分区。Flink 提供的底层 API ，允许用户定义数据的分区规则；
- **Task chaining and resource groups**：任务链和资源组。允许用户进行任务链和资源组的细粒度的控制。

以下分别对其主要 API 进行介绍：

## 二、DataStream Transformations

### 2.1 Map [DataStream → DataStream] 

对一个 DataStream 中的每个元素都执行特定的转换操作：

```java
DataStream<Integer> integerDataStream = env.fromElements(1, 2, 3, 4, 5);
integerDataStream.map((MapFunction<Integer, Object>) value -> value * 2).print();
// 输出 2,4,6,8,10
```

### 2.2 FlatMap [DataStream → DataStream]

FlatMap 与 Map 类似，但是 FlatMap 中的一个输入元素可以被映射成一个或者多个输出元素，示例如下：

```java
String string01 = "one one one two two";
String string02 = "third third third four";
DataStream<String> stringDataStream = env.fromElements(string01, string02);
stringDataStream.flatMap(new FlatMapFunction<String, String>() {
    @Override
    public void flatMap(String value, Collector<String> out) throws Exception {
        for (String s : value.split(" ")) {
            out.collect(s);
        }
    }
}).print();
// 输出每一个独立的单词，为节省排版，这里去掉换行，后文亦同
one one one two two third third third four
```

### 2.3 Filter [DataStream → DataStream]

用于过滤符合条件的数据：

```java
env.fromElements(1, 2, 3, 4, 5).filter(x -> x > 3).print();
```

### 2.4 KeyBy 和 Reduce

- **KeyBy [DataStream → KeyedStream]** ：用于将相同 Key 值的数据分到相同的分区中；
- **Reduce [KeyedStream → DataStream]** ：用于对数据执行归约计算。

如下例子将数据按照 key 值分区后，滚动进行求和计算：

```java
DataStream<Tuple2<String, Integer>> tuple2DataStream = env.fromElements(new Tuple2<>("a", 1),
                                                                        new Tuple2<>("a", 2), 
                                                                        new Tuple2<>("b", 3), 
                                                                        new Tuple2<>("b", 5));
KeyedStream<Tuple2<String, Integer>, Tuple> keyedStream = tuple2DataStream.keyBy(0);
keyedStream.reduce((ReduceFunction<Tuple2<String, Integer>>) (value1, value2) ->
                   new Tuple2<>(value1.f0, value1.f1 + value2.f1)).print();

// 持续进行求和计算，输出：
(a,1)
(a,3)
(b,3)
(b,8)
```

KeyBy 操作存在以下两个限制：

- KeyBy 操作用于用户自定义的 POJOs 类型时，该自定义类型必须重写 hashCode 方法；
- KeyBy 操作不能用于数组类型。

### 2.5 Aggregations [KeyedStream → DataStream]

Aggregations 是官方提供的聚合算子，封装了常用的聚合操作，如上利用 Reduce 进行求和的操作也可以利用 Aggregations 中的 sum 算子重写为下面的形式：

```java
tuple2DataStream.keyBy(0).sum(1).print();
```

除了 sum 外，Flink 还提供了 min , max , minBy，maxBy 等常用聚合算子：

```java
// 滚动计算指定key的最小值，可以通过index或者fieldName来指定key
keyedStream.min(0);
keyedStream.min("key");
// 滚动计算指定key的最大值
keyedStream.max(0);
keyedStream.max("key");
// 滚动计算指定key的最小值，并返回其对应的元素
keyedStream.minBy(0);
keyedStream.minBy("key");
// 滚动计算指定key的最大值，并返回其对应的元素
keyedStream.maxBy(0);
keyedStream.maxBy("key");

```

### 2.6 Union [DataStream → DataStream]

用于连接两个或者多个元素类型相同的 DataStream 。当然一个 DataStream 也可以与其本生进行连接，此时该 DataStream 中的每个元素都会被获取两次：

```shell
DataStreamSource<Tuple2<String, Integer>> streamSource01 = env.fromElements(new Tuple2<>("a", 1), 
                                                                            new Tuple2<>("a", 2));
DataStreamSource<Tuple2<String, Integer>> streamSource02 = env.fromElements(new Tuple2<>("b", 1), 
                                                                            new Tuple2<>("b", 2));
streamSource01.union(streamSource02);
streamSource01.union(streamSource01,streamSource02);
```

### 2.7 Connect [DataStream,DataStream → ConnectedStreams]

Connect 操作用于连接两个或者多个类型不同的 DataStream ，其返回的类型是 ConnectedStreams ，此时被连接的多个 DataStreams 可以共享彼此之间的数据状态。但是需要注意的是由于不同 DataStream 之间的数据类型是不同的，如果想要进行后续的计算操作，还需要通过 CoMap 或 CoFlatMap 将 ConnectedStreams  转换回 DataStream：

```java
DataStreamSource<Tuple2<String, Integer>> streamSource01 = env.fromElements(new Tuple2<>("a", 3), 
                                                                            new Tuple2<>("b", 5));
DataStreamSource<Integer> streamSource02 = env.fromElements(2, 3, 9);
// 使用connect进行连接
ConnectedStreams<Tuple2<String, Integer>, Integer> connect = streamSource01.connect(streamSource02);
connect.map(new CoMapFunction<Tuple2<String, Integer>, Integer, Integer>() {
    @Override
    public Integer map1(Tuple2<String, Integer> value) throws Exception {
        return value.f1;
    }

    @Override
    public Integer map2(Integer value) throws Exception {
        return value;
    }
}).map(x -> x * 100).print();

// 输出：
300 500 200 900 300
```

### 2.8 Split 和 Select

- **Split [DataStream → SplitStream]**：用于将一个 DataStream 按照指定规则进行拆分为多个 DataStream，需要注意的是这里进行的是逻辑拆分，即 Split 只是将数据贴上不同的类型标签，但最终返回的仍然只是一个 SplitStream；
- **Select [SplitStream → DataStream]**：想要从逻辑拆分的 SplitStream 中获取真实的不同类型的 DataStream，需要使用 Select 算子，示例如下：

```java
DataStreamSource<Integer> streamSource = env.fromElements(1, 2, 3, 4, 5, 6, 7, 8);
// 标记
SplitStream<Integer> split = streamSource.split(new OutputSelector<Integer>() {
    @Override
    public Iterable<String> select(Integer value) {
        List<String> output = new ArrayList<String>();
        output.add(value % 2 == 0 ? "even" : "odd");
        return output;
    }
});
// 获取偶数数据集
split.select("even").print();
// 输出 2,4,6,8
```

### 2.9 project [DataStream → DataStream]

project 主要用于获取 tuples 中的指定字段集，示例如下：

```java
DataStreamSource<Tuple3<String, Integer, String>> streamSource = env.fromElements(
                                                                         new Tuple3<>("li", 22, "2018-09-23"),
                                                                         new Tuple3<>("ming", 33, "2020-09-23"));
streamSource.project(0,2).print();

// 输出
(li,2018-09-23)
(ming,2020-09-23)
```

## 三、物理分区

物理分区 (Physical partitioning) 是 Flink 提供的底层的 API，允许用户采用内置的分区规则或者自定义的分区规则来对数据进行分区，从而避免数据在某些分区上过于倾斜，常用的分区规则如下：

### 3.1 Random partitioning [DataStream → DataStream]

随机分区 (Random partitioning) 用于随机的将数据分布到所有下游分区中，通过 shuffle 方法来进行实现：

```java
dataStream.shuffle();
```

### 3.2 Rebalancing [DataStream → DataStream]

Rebalancing 采用轮询的方式将数据进行分区，其适合于存在数据倾斜的场景下，通过 rebalance 方法进行实现：  

```java
dataStream.rebalance();
```

### 3.3 Rescaling [DataStream → DataStream]

当采用 Rebalancing 进行分区平衡时，其实现的是全局性的负载均衡，数据会通过网络传输到其他节点上并完成分区数据的均衡。 而 Rescaling 则是低配版本的 rebalance，它不需要额外的网络开销，它只会对上下游的算子之间进行重新均衡，通过 rescale 方法进行实现：

```java
dataStream.rescale();
```

ReScale 这个单词具有重新缩放的意义，其对应的操作也是如此，具体如下：如果上游 operation 并行度为 2，而下游的 operation 并行度为 6，则其中 1 个上游的 operation 会将元素分发到 3 个下游 operation，另 1 个上游 operation 则会将元素分发到另外 3 个下游 operation。反之亦然，如果上游的 operation 并行度为 6，而下游 operation 并行度为 2，则其中 3 个上游 operation 会将元素分发到 1 个下游 operation，另 3 个上游 operation 会将元素分发到另外 1 个下游operation：

![1586420666492](/img/1586420666492.png)

### 3.4 Broadcasting [DataStream → DataStream]

将数据分发到所有分区上。通常用于小数据集与大数据集进行关联的情况下，此时可以将小数据集广播到所有分区上，避免频繁的跨分区关联，通过 broadcast 方法进行实现：

```java
dataStream.broadcast();
```

### 3.5 Custom partitioning [DataStream → DataStream]

Flink 运行用户采用自定义的分区规则来实现分区，此时需要通过实现 Partitioner 接口来自定义分区规则，并指定对应的分区键，示例如下：

```java
 DataStreamSource<Tuple2<String, Integer>> streamSource = env.fromElements(new Tuple2<>("Hadoop", 1),
                new Tuple2<>("Spark", 1),
                new Tuple2<>("Flink-streaming", 2),
                new Tuple2<>("Flink-batch", 4),
                new Tuple2<>("Storm", 4),
                new Tuple2<>("HBase", 3));
streamSource.partitionCustom(new Partitioner<String>() {
    @Override
    public int partition(String key, int numPartitions) {
        // 将第一个字段包含flink的Tuple2分配到同一个分区
        return key.toLowerCase().contains("flink") ? 0 : 1;
    }
}, 0).print();


// 输出如下：
1> (Flink-streaming,2)
1> (Flink-batch,4)
2> (Hadoop,1)
2> (Spark,1)
2> (Storm,4)
2> (HBase,3)
```

## 四、任务链和资源组

任务链和资源组 ( Task chaining and resource groups ) 也是 Flink 提供的底层 API，用于控制任务链和资源分配。默认情况下，如果操作允许 (例如相邻的两次 map 操作) ，则 Flink 会尝试将它们在同一个线程内进行，从而可以获取更好的性能。但是 Flink 也允许用户自己来控制这些行为，这就是任务链和资源组 API：

### 4.1 startNewChain

startNewChain 用于基于当前 operation 开启一个新的任务链。如下所示，基于第一个 map 开启一个新的任务链，此时前一个 map 和 后一个 map 将处于同一个新的任务链中，但它们与 filter 操作则分别处于不同的任务链中：

```java
someStream.filter(...).map(...).startNewChain().map(...);
```

### 4.2 disableChaining

 disableChaining 操作用于禁止将其他操作与当前操作放置于同一个任务链中，示例如下：

```java
someStream.map(...).disableChaining();
```

### 4.3 slotSharingGroup

slot 是任务管理器  (TaskManager) 所拥有资源的固定子集，每个操作 (operation) 的子任务 (sub task) 都需要获取 slot 来执行计算，但每个操作所需要资源的大小都是不相同的，为了更好地利用资源，Flink 允许不同操作的子任务被部署到同一 slot 中。slotSharingGroup 用于设置操作的 slot 共享组 (slot sharing group) ，Flink 会将具有相同 slot 共享组的操作放到同一个 slot 中 。示例如下：

```java
someStream.filter(...).slotSharingGroup("slotSharingGroupName");
```

# Sink

## 一、Data Sinks

在使用 Flink 进行数据处理时，数据经 Data Source 流入，然后通过系列 Transformations 的转化，最终可以通过 Sink 将计算结果进行输出，Flink Data Sinks 就是用于定义数据流最终的输出位置。Flink 提供了几个较为简单的 Sink API 用于日常的开发，具体如下：

### 1.1 writeAsText

`writeAsText` 用于将计算结果以文本的方式并行地写入到指定文件夹下，除了路径参数是必选外，该方法还可以通过指定第二个参数来定义输出模式，它有以下两个可选值：

- **WriteMode.NO_OVERWRITE**：当指定路径上不存在任何文件时，才执行写出操作；
- **WriteMode.OVERWRITE**：不论指定路径上是否存在文件，都执行写出操作；如果原来已有文件，则进行覆盖。

使用示例如下：

```java
 streamSource.writeAsText("D:\\out", FileSystem.WriteMode.OVERWRITE);
```

以上写出是以并行的方式写出到多个文件，如果想要将输出结果全部写出到一个文件，需要设置其并行度为 1：

```java
streamSource.writeAsText("D:\\out", FileSystem.WriteMode.OVERWRITE).setParallelism(1);
```

### 1.2 writeAsCsv

`writeAsCsv` 用于将计算结果以 CSV 的文件格式写出到指定目录，除了路径参数是必选外，该方法还支持传入输出模式，行分隔符，和字段分隔符三个额外的参数，其方法定义如下：

```java
writeAsCsv(String path, WriteMode writeMode, String rowDelimiter, String fieldDelimiter) 
```

### 1.3 print / printToErr

`print / printToErr` 是测试当中最常用的方式，用于将计算结果以标准输出流或错误输出流的方式打印到控制台上。

### 1.4 writeUsingOutputFormat

采用自定义的输出格式将计算结果写出，上面介绍的 `writeAsText` 和 `writeAsCsv` 其底层调用的都是该方法，源码如下：

```java
public DataStreamSink<T> writeAsText(String path, WriteMode writeMode) {
    TextOutputFormat<T> tof = new TextOutputFormat<>(new Path(path));
    tof.setWriteMode(writeMode);
    return writeUsingOutputFormat(tof);
}
```

### 1.5 writeToSocket

`writeToSocket` 用于将计算结果以指定的格式写出到 Socket 中，使用示例如下：

```shell
streamSource.writeToSocket("192.168.0.226", 9999, new SimpleStringSchema());
```

## 二、整合 Kafka Sink

### 3.1 addSink

Flink 提供了 addSink 方法用来调用自定义的 Sink 或者第三方的连接器，想要将计算结果写出到 Kafka，需要使用该方法来调用 Kafka 的生产者 FlinkKafkaProducer，具体代码如下：

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 1.指定Kafka的相关配置属性
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "192.168.200.0:9092");

// 2.接收Kafka上的数据
DataStream<String> stream = env
    .addSource(new FlinkKafkaConsumer<>("flink-stream-in-topic", new SimpleStringSchema(), properties));

// 3.定义计算结果到 Kafka ProducerRecord 的转换
KafkaSerializationSchema<String> kafkaSerializationSchema = new KafkaSerializationSchema<String>() {
    @Override
    public ProducerRecord<byte[], byte[]> serialize(String element, @Nullable Long timestamp) {
        return new ProducerRecord<>("flink-stream-out-topic", element.getBytes());
    }
};
// 4. 定义Flink Kafka生产者
FlinkKafkaProducer<String> kafkaProducer = new FlinkKafkaProducer<>("flink-stream-out-topic",
                                                                    kafkaSerializationSchema,
                                                                    properties,
                                               FlinkKafkaProducer.Semantic.AT_LEAST_ONCE, 5);
// 5. 将接收到输入元素*2后写出到Kafka
stream.map((MapFunction<String, String>) value -> value + value).addSink(kafkaProducer);
env.execute("Flink Streaming");
```

### 3.2 创建输出主题

创建用于输出测试的主题：

```shell
bin/kafka-topics.sh --create \
                    --bootstrap-server hadoop001:9092 \
                    --replication-factor 1 \
                    --partitions 1  \
                    --topic flink-stream-out-topic

# 查看所有主题
 bin/kafka-topics.sh --list --bootstrap-server hadoop001:9092
```

### 3.3 启动消费者

启动一个 Kafka 消费者，用于查看 Flink 程序的输出情况：

```java
bin/kafka-console-consumer.sh --bootstrap-server hadoop001:9092 --topic flink-stream-out-topic
```

### 3.4 测试结果

在 Kafka 生产者上发送消息到 Flink 程序，观察 Flink 程序转换后的输出情况，具体如下：

![1586421679558](/img/1586421679558.png)

可以看到 Kafka 生成者发出的数据已经被 Flink 程序正常接收到，并经过转换后又输出到 Kafka 对应的 Topic 上。

## 三、自定义 Sink

除了使用内置的第三方连接器外，Flink 还支持使用自定义的 Sink 来满足多样化的输出需求。想要实现自定义的 Sink ，需要直接或者间接实现 SinkFunction 接口。通常情况下，我们都是实现其抽象类 RichSinkFunction，相比于 SinkFunction ，其提供了更多的与生命周期相关的方法。两者间的关系如下：

 ![1586421691428](/img/1586421691428.png)

这里我们以自定义一个 FlinkToMySQLSink 为例，将计算结果写出到 MySQL 数据库中，具体步骤如下：

### 4.1 导入依赖

首先需要导入 MySQL 相关的依赖：

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.16</version>
</dependency>
```

### 4.2 自定义 Sink

继承自 RichSinkFunction，实现自定义的 Sink ：

```java
public class FlinkToMySQLSink extends RichSinkFunction<Employee> {

    private PreparedStatement stmt;
    private Connection conn;

    @Override
    public void open(Configuration parameters) throws Exception {
        Class.forName("com.mysql.cj.jdbc.Driver");
        conn = DriverManager.getConnection("jdbc:mysql://192.168.0.229:3306/employees" +
                                           "?characterEncoding=UTF-8&serverTimezone=UTC&useSSL=false", 
                                           "root", 
                                           "123456");
        String sql = "insert into emp(name, age, birthday) values(?, ?, ?)";
        stmt = conn.prepareStatement(sql);
    }

    @Override
    public void invoke(Employee value, Context context) throws Exception {
        stmt.setString(1, value.getName());
        stmt.setInt(2, value.getAge());
        stmt.setDate(3, value.getBirthday());
        stmt.executeUpdate();
    }

    @Override
    public void close() throws Exception {
        super.close();
        if (stmt != null) {
            stmt.close();
        }
        if (conn != null) {
            conn.close();
        }
    }

}
```

### 4.3 使用自定义 Sink

想要使用自定义的 Sink，同样是需要调用 addSink 方法，具体如下：

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
Date date = new Date(System.currentTimeMillis());
DataStreamSource<Employee> streamSource = env.fromElements(
    new Employee("hei", 10, date),
    new Employee("bai", 20, date),
    new Employee("ying", 30, date));
streamSource.addSink(new FlinkToMySQLSink());
env.execute();
```

### 4.4 测试结果

启动程序，观察数据库写入情况：

![1586421708524](/img/1586421708524.png)

数据库成功写入，代表自定义 Sink 整合成功。

## Streaming Connectors

除了上述 API 外，Flink 中还内置了系列的 Connectors 连接器，用于将计算结果输入到常用的存储系统或者消息中间件中，具体如下：

- Apache Kafka (支持 source 和 sink)
- Apache Cassandra (sink)
- Amazon Kinesis Streams (source/sink)
- Elasticsearch (sink)
- Hadoop FileSystem (sink)
- RabbitMQ (source/sink)
- Apache NiFi (source/sink)
- Google PubSub (source/sink)

除了内置的连接器外，你还可以通过 Apache Bahir 的连接器扩展 Flink。Apache Bahir 旨在为分布式数据分析系统 (如 Spark，Flink) 等提供功能上的扩展，当前其支持的与 Flink Sink 相关的连接器如下：

- Apache ActiveMQ (source/sink)
- Apache Flume (sink)
- Redis (sink)
- Akka (sink)

这里接着在 Data Sources 章节介绍的整合 Kafka Source 的基础上，将 Kafka Sink 也一并进行整合，具体步骤如下。