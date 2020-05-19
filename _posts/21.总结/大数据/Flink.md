---
title: Flink 总结
date: "2020-04-28 13:43:11"
categories:
- 总结
tags:
- 总结
toc: true
typora-root-url: ..\..\..
---

### Flink组件栈

Flink 采用分层的架构设计，从而保证各层在功能和职责上的清晰。如下图所示，由上而下分别是 API & Libraries 层、Runtime 核心层以及物理部署层：

![1586404760423](/img/1586404760423.png)

#### API & Libraries 层

这一层主要提供了编程 API 和 顶层类库：

- 编程 API : 用于进行流处理的 DataStream API 和用于进行批处理的 DataSet API；
- 顶层类库：包括用于复杂事件处理的 CEP 库；用于结构化数据查询的 SQL & Table 库，以及基于批处理的机器学习库 FlinkML 和 图形处理库 Gelly。

#### Runtime 核心层

这一层是 Flink 分布式计算框架的核心实现层，包括作业转换，任务调度，资源分配，任务执行等功能，基于这一层的实现，可以在流式引擎下同时运行流处理程序和批处理程序。比如：支持分布式Stream处理、JobGraph到ExecutionGraph的映射、调度等等，为上层API层提供基础服务

#### 物理部署层

Flink 的物理部署层，用于支持在不同平台上部署运行 Flink 应用。

主要涉及了Flink的部署模式，Flink支持多种部署模式：本地、集群（Standalone/YARN）、云（GCE/EC2）

![1586489584052](/img/1586489584052.png)

### Flink 分层 API

在上面介绍的 API & Libraries 这一层，Flink 又进行了更为具体的划分。具体如下：

![1586404927178](/img/1586404927178.png)

按照如上的层次结构，API 的一致性由下至上依次递增，接口的表现能力由下至上依次递减，各层的核心功能如下：

#### SQL & Table API

SQL & Table API 同时适用于批处理和流处理，这意味着你可以对有界数据流和无界数据流以相同的语义进行查询，并产生相同的结果。除了基本查询外， 它还支持自定义的标量函数，聚合函数以及表值函数，可以满足多样化的查询需求。 

#### DataStream & DataSet API

DataStream &  DataSet API 是 Flink 数据处理的核心 API，支持使用 Java 语言或 Scala 语言进行调用，提供了数据读取，数据转换和数据输出等一系列常用操作的封装。

#### Stateful Stream Processing

Stateful Stream Processing 是最低级别的抽象，它通过 Process Function 函数内嵌到 DataStream API 中。 Process Function 是 Flink 提供的最底层 API，具有最大的灵活性，允许开发者对于时间和状态进行细粒度的控制。

### Flink 集群架构

#### 核心组件

按照上面的介绍，Flink 核心架构的第二层是 Runtime 层， 该层采用标准的 Master - Slave 结构， 其中，Master 部分又包含了三个核心组件：Dispatcher、ResourceManager 和 JobManager，而 Slave 则主要是 TaskManager 进程。它们的功能分别如下：

- **Client**： 用户提交一个Flink程序时，会首先创建一个Client，该Client首先会对用户提交的Flink程序进行预处理，并提交到Flink集群， Client会将用户提交的Flink程序组装一个JobGraph， 并且是以JobGraph的形式提交的

- **Dispatcher**：负责接收客户端提交的执行程序，并传递给 JobManager 。除此之外，它还提供了一个 WEB UI 界面，用于监控作业的执行情况。

- **JobManagers** (也称为 *masters*) ：JobManagers 接收由 Dispatcher 传递过来的执行程序，该执行程序包含了作业图 (JobGraph)，逻辑数据流图 (logical dataflow graph) 及其所有的 classes 文件以及第三方类库 (libraries) 等等 。紧接着 JobManagers 会将 JobGraph 转换为执行图 (ExecutionGraph)，然后向 ResourceManager 申请资源来执行该任务，一旦申请到资源，就将执行图分发给对应的 TaskManagers 。因此每个作业 (Job) 至少有一个 JobManager；高可用部署下可以有多个 JobManagers，其中一个作为 *leader*，其余的则处于 *standby* 状态。
- **ResourceManager** ：负责管理 slots 并协调集群资源。ResourceManager 接收来自 JobManager 的资源请求，并将存在空闲 slots 的 TaskManagers 分配给 JobManager 执行任务。Flink 基于不同的部署平台，如 YARN , Mesos，K8s 等提供了不同的资源管理器，当 TaskManagers 没有足够的 slots 来执行任务时，它会向第三方平台发起会话来请求额外的资源。
- **TaskManagers** (也称为 *workers*) : TaskManagers 负责实际的子任务 (subtasks) 的执行，每个 TaskManagers 都拥有一定数量的 slots。Slot 是一组固定大小的资源的合集 (如计算能力，存储空间)。TaskManagers 启动后，会将其所拥有的 slots 注册到 ResourceManager 上，由 ResourceManager 进行统一管理。

![1586410450743](/img/1586410450743.png)

#### Task & SubTask

上面我们提到：TaskManagers 实际执行的是 SubTask，而不是 Task，这里解释一下两者的区别：

在执行分布式计算时，Flink 将可以链接的操作 (operators) 链接到一起，这就是 Task。之所以这样做， 是为了减少线程间切换和缓冲而导致的开销，在降低延迟的同时可以提高整体的吞吐量。 但不是所有的 operator 都可以被链接，如下 keyBy 等操作会导致网络 shuffle 和重分区，因此其就不能被链接，只能被单独作为一个 Task。  简单来说，一个 Task 就是一个可以链接的最小的操作链 (Operator Chains) 。如下图，source 和 map 算子被链接到一块，因此整个作业就只有三个 Task：

![1586410843615](/img/1586410843615.png)

解释完 Task ，我们在解释一下什么是 SubTask，其准确的翻译是： *A subtask is one parallel slice of a task*，即一个 Task 可以按照其并行度拆分为多个 SubTask。如上图，source & map 具有两个并行度，KeyBy 具有两个并行度，Sink 具有一个并行度，因此整个虽然只有 3 个 Task，但是却有 5 个 SubTask。Jobmanager 负责定义和拆分这些 SubTask，并将其交给 Taskmanagers 来执行，每个 SubTask 都是一个单独的线程。

#### 资源管理

理解了 SubTasks ，我们再来看看其与 Slots 的对应情况。一种可能的分配情况如下：

![1586410901685](/img/1586410901685.png)

这时每个 SubTask 线程运行在一个独立的 TaskSlot， 它们共享所属的 TaskManager 进程的TCP 连接（通过多路复用技术）和心跳信息 (heartbeat messages)，从而可以降低整体的性能开销。此时看似是最好的情况，但是每个操作需要的资源都是不尽相同的，这里假设该作业 keyBy 操作所需资源的数量比 Sink 多很多 ，那么此时 Sink 所在 Slot 的资源就没有得到有效的利用。

基于这个原因，Flink 允许多个 subtasks 共享 slots，即使它们是不同 tasks 的 subtasks，但只要它们来自同一个 Job 就可以。假设上面 souce & map 和 keyBy 的并行度调整为 6，而 Slot 的数量不变，此时情况如下：

![1586411714405](/img/1586411714405.png)

可以看到一个 Task Slot 中运行了多个 SubTask 子任务，此时每个子任务仍然在一个独立的线程中执行，只不过共享一组 Sot 资源而已。那么 Flink 到底如何确定一个 Job 至少需要多少个 Slot 呢？Flink 对于这个问题的处理很简单，默认情况一个 Job 所需要的 Slot 的数量就等于其 Operation 操作的最高并行度。如下， A，B，D 操作的并行度为 4，而 C，E 操作的并行度为 2，那么此时整个 Job 就需要至少四个 Slots 来完成。通过这个机制，Flink 就可以不必去关心一个 Job 到底会被拆分为多少个 Tasks 和 SubTasks。

![1586411745166](/img/1586411745166.png)

### Flink 优势

- 支持高吞吐、低延迟、高性能的流处理
- 支持高度灵活的窗口（Window）操作
- 支持有状态计算的Exactly-once语义
- 提供DataStream API和DataSet API

![1586489825589](/img/1586489825589.png)

### Source 和 sink

#### 自定义 Data Source

**SourceFunction**

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

#### ParallelSourceFunction 和 RichParallelSourceFunction

上面通过 SourceFunction 实现的数据源是不具有并行度的，即不支持在得到的 DataStream 上调用 `setParallelism(n)` 方法，此时会抛出如下的异常：

```shell
Exception in thread "main" java.lang.IllegalArgumentException: Source: 1 is not a parallel source
```

如果你想要实现具有并行度的输入流，则需要实现 ParallelSourceFunction 或 RichParallelSourceFunction 接口，其与 SourceFunction 的关系如下图： 

![1586415778743](/img/1586415778743.png)

ParallelSourceFunction 直接继承自 ParallelSourceFunction，具有并行度的功能。RichParallelSourceFunction 则继承自 AbstractRichFunction，同时实现了 ParallelSourceFunction 接口，所以其除了具有并行度的功能外，还提供了额外的与生命周期相关的方法，如 open() ，closen() 。

#### 自定义 Sink

除了使用内置的第三方连接器外，Flink 还支持使用自定义的 Sink 来满足多样化的输出需求。想要实现自定义的 Sink ，需要直接或者间接实现 SinkFunction 接口。通常情况下，我们都是实现其抽象类 RichSinkFunction，相比于 SinkFunction ，其提供了更多的与生命周期相关的方法。两者间的关系如下：

 ![1586421691428](/img/1586421691428.png)

这里我们以自定义一个 FlinkToMySQLSink 为例，将计算结果写出到 MySQL 数据库中，具体步骤如下：

##### 导入依赖

首先需要导入 MySQL 相关的依赖：

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.16</version>
</dependency>
```

##### 自定义 Sink

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

##### 使用自定义 Sink

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

#### Streaming Connectors

##### 内置连接器

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

###  Flink 状态

相对于其他流计算框架，Flink 一个比较重要的特性就是其支持有状态计算。即你可以将中间的计算结果进行保存，并提供给后续的计算使用：

![1586424744865](/img/1586424744865.png)

具体而言，Flink 又将状态 (State) 分为 Keyed State 与 Operator State：

#### 算子状态

算子状态 (Operator State)：顾名思义，状态是和算子进行绑定的，一个算子的状态不能被其他算子所访问到。官方文档上对 Operator State 的解释是：*each operator state is bound to one parallel operator instance*，所以更为确切的说一个算子状态是与一个并发的算子实例所绑定的，即假设算子的并行度是 2，那么其应有两个对应的算子状态：

![1586425914911](/img/1586425914911.png)

#### 键控状态

键控状态 (Keyed State) ：是一种特殊的算子状态，即状态是根据 key 值进行区分的，Flink 会为每类键值维护一个状态实例。如下图所示，每个颜色代表不同 key 值，对应四个不同的状态实例。需要注意的是键控状态只能在 `KeyedStream` 上进行使用，我们可以通过 `stream.keyBy(...)` 来得到 `KeyedStream` 。

![1586425957804](/img/1586425957804.png)

### 检查点机制

#### checkpoint

为了使 Flink 的状态具有良好的容错性，Flink 提供了检查点机制 (CheckPoints)  。通过检查点机制，Flink 定期在数据流上生成 checkpoint barrier ，当某个算子收到 barrier 时，即会基于当前状态生成一份快照，然后再将该 barrier 传递到下游算子，下游算子接收到该 barrier 后，也基于当前状态生成一份快照，依次传递直至到最后的 Sink 算子上。当出现异常后，Flink 就可以根据最近的一次的快照数据将所有算子恢复到先前的状态。

#### 开启检查点

默认情况下，检查点机制是关闭的，需要在程序中进行开启：

```java
// 开启检查点机制，并指定状态检查点之间的时间间隔
env.enableCheckpointing(1000); 

// 其他可选配置如下：
// 设置语义
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
// 设置两个检查点之间的最小时间间隔
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);
// 设置执行Checkpoint操作时的超时时间
env.getCheckpointConfig().setCheckpointTimeout(60000);
// 设置最大并发执行的检查点的数量
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
// 将检查点持久化到外部存储
env.getCheckpointConfig().enableExternalizedCheckpoints(
    ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
// 如果有更近的保存点时，是否将作业回退到该检查点
env.getCheckpointConfig().setPreferCheckpointForRecovery(true);
```

### 窗口函数

#### Tumbling Windows

滚动窗口 (Tumbling Windows) 是指彼此之间没有重叠的窗口。例如：每隔1小时统计过去1小时内的商品点击量，那么 1 天就只能分为 24 个窗口，每个窗口彼此之间是不存在重叠的，具体如下：

![1586422308463](/img/1586422308463.png)

1. 这里我们以词频统计为例，给出一个具体的用例，代码如下：

   ```java
   final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   // 接收socket上的数据输入
   DataStreamSource<String> streamSource = env.socketTextStream("hadoop001", 9999, "\n", 3);
   streamSource.flatMap(new FlatMapFunction<String, Tuple2<String, Long>>() {
       @Override
       public void flatMap(String value, Collector<Tuple2<String, Long>> out) throws Exception {
           String[] words = value.split("\t");
           for (String word : words) {
               out.collect(new Tuple2<>(word, 1L));
           }
       }
   }).keyBy(0).timeWindow(Time.seconds(3)).sum(1).print(); //每隔3秒统计一次每个单词出现的数量
   env.execute("Flink Streaming");
   ```

2. 假如我们需要统计每一分钟中用户购买的商品的总数，需要将用户的行为事件按每一分钟进行切分，这种切分被成为翻滚时间窗口（Tumbling Time Window）。翻滚窗口能将数据流切分成不重叠的窗口，每一个事件只能属于一个窗口。

   ```java
   // 用户id和购买数量 stream
   val counts: DataStream[(Int, Int)] = ...
   val tumblingCnts: DataStream[(Int, Int)] = counts
     // 用userId分组
     .keyBy(0) 
     // 1分钟的翻滚窗口宽度
     .timeWindow(Time.minutes(1))
     // 计算购买数量
     .sum(1) 
   ```

#### Sliding Windows

滑动窗口用于滚动进行聚合分析，例如：每隔 6 分钟统计一次过去一小时内所有商品的点击量，那么统计窗口彼此之间就是存在重叠的，即 1天可以分为 240 个窗口。图示如下：

![1586422565315](/img/1586422565315.png)

可以看到 window 1 - 4 这四个窗口彼此之间都存在着时间相等的重叠部分。想要实现滑动窗口，只需要在使用 timeWindow 方法时额外传递第二个参数作为滚动时间即可，具体如下：

```java
// 每隔3秒统计一次过去1分钟内的数据
keyBy(0).timeWindow(Time.minutes(1),Time.seconds(3)).sum(1)
```

#### Session Windows

当用户在进行持续浏览时，可能每时每刻都会有点击数据，例如在活动区间内，用户可能频繁的将某类商品加入和移除购物车，而你只想知道用户本次浏览最终的购物车情况，此时就可以在用户持有的会话结束后再进行统计。想要实现这类统计，可以通过 Session Windows 来进行实现。

![1586423597586](/img/1586423597586.png)

具体的实现代码如下：

```java
// 以处理时间为衡量标准，如果10秒内没有任何数据输入，就认为会话已经关闭，此时触发统计
window(ProcessingTimeSessionWindows.withGap(Time.seconds(10)))
// 以事件时间为衡量标准    
window(EventTimeSessionWindows.withGap(Time.seconds(10)))
```

#### Global Windows

最后一个窗口是全局窗口， 全局窗口会将所有 key 相同的元素分配到同一个窗口中，其通常配合触发器 (trigger) 进行使用。如果没有相应触发器，则计算将不会被执行。

![1586424315574](/img/1586424315574.png)

这里继续以上面词频统计的案例为例，示例代码如下：

```java
// 当单词累计出现的次数每达到10次时，则触发计算，计算整个窗口内该单词出现的总数
window(GlobalWindows.create()).trigger(CountTrigger.of(10)).sum(1).print();
```

#### Count Windows

Count Windows 用于以数量为维度来进行数据聚合，同样也分为滚动窗口和滑动窗口，实现方式也和时间窗口完全一致，只是调用的 API 不同，具体如下：

```java
// 滚动计数窗口，每1000次点击则计算一次
countWindow(1000)
// 滑动计数窗口，每10次点击发生后，则计算过去1000次点击的情况
countWindow(1000,10)
```

实际上计数窗口内部就是调用的我们上一部分介绍的全局窗口来实现的，其源码如下：

```java
public WindowedStream<T, KEY, GlobalWindow> countWindow(long size) {
    return window(GlobalWindows.create()).trigger(PurgingTrigger.of(CountTrigger.of(size)));
}

public WindowedStream<T, KEY, GlobalWindow> countWindow(long size, long slide) {
    return window(GlobalWindows.create())
        .evictor(CountEvictor.of(size))
        .trigger(CountTrigger.of(slide));
}
```

### 时间（Time）

#### 时间类型

- Flink中的时间与现实世界中的时间是不一致的，在flink中被划分为**事件时间，摄入时间，处理时间**三种。
- 如果以EventTime为基准来定义时间窗口将形成EventTimeWindow,要求消息本身就应该携带EventTime
- 如果以IngesingtTime为基准来定义时间窗口将形成IngestingTimeWindow,以source的systemTime为准。
- 如果以ProcessingTime基准来定义时间窗口将形成ProcessingTimeWindow，以operator的systemTime为准。

![1586577084774](/img/1586577084774.png)

#### 时间详解

**Processing Time**

Processing Time 是指事件被处理时机器的系统时间。

**Event Time**

Event Time 是事件发生的时间，一般就是数据本身携带的时间。这个时间通常是在事件到达 Flink 之前就确定的，并且可以从每个事件中获取到事件时间戳。在 Event Time 中，时间取决于数据，而跟其他没什么关系。Event Time 程序必须指定如何生成 Event Time 水印，这是表示 Event Time 进度的机制。

**Ingestion Time**

Ingestion Time 是事件进入 Flink 的时间。 在源操作处，每个事件将源的当前时间作为时间戳，并且基于时间的操作（如时间窗口）会利用这个时间戳。

#### 时间乱序存在的问题

在介绍Watermark相关内容之前我们先抛出一个具体的问题，在实际的流式计算中数据到来的顺序对计算结果的正确性有至关重要的影响，比如：某数据源中的某些数据由于某种原因(如：网络原因，外部存储自身原因)会有5秒的延时，也就是在实际时间的第1秒产生的数据有可能在第5秒中产生的数据之后到来(比如到Window处理节点).选具体某个delay的元素来说，假设在一个5秒的Tumble窗口(详见Window介绍章节)，有一个EventTime是 11秒的数据，在第16秒时候到来了。图示第11秒的数据，在16秒到来了，如下图：

![1586589440002](/img/1586589440002.png)

那么对于一个Count聚合的Tumble(5s)的window，上面的情况如何处理才能window2=4，window3=2 呢？Apache Flink的时间类型
开篇我们描述的问题是一个很常见的TimeWindow中数据乱序的问题，乱序是相对于事件产生时间和到达Apache Flink 实际处理算子的顺序而言的，关于时间在Apache Flink中有如下三种时间类型，如下图：

![1586589447529](/img/1586589447529.png)

那么对于一个Count聚合的Tumble(5s)的window，上面的情况如何处理才能window2=4，window3=2 呢？

#### 什么是Watermark

Watermark是Apache Flink为了处理EventTime 窗口计算提出的一种机制,本质上也是一种时间戳，由Apache Flink Source或者自定义的Watermark生成器按照需求Punctuated或者Periodic两种方式生成的一种系统Event，与普通数据流Event一样流转到对应的下游算子，接收到Watermark Event的算子以此不断调整自己管理的EventTime clock。 Apache Flink 框架保证Watermark单调递增，算子接收到一个Watermark时候，框架知道不会再有任何小于该Watermark的时间戳的数据元素到来了，所以Watermark可以看做是告诉Apache Flink框架数据流已经处理到什么位置(时间维度)的方式。 Watermark的产生和Apache Flink内部处理逻辑如下图所示: 

![1586589616212](/img/1586589616212.png)

#### Watermark的产生方式

目前Apache Flink 有两种生产Watermark的方式，如下：

- Punctuated - 数据流中每一个递增的EventTime都会产生一个Watermark。 

在实际的生产中Punctuated方式在TPS很高的场景下会产生大量的Watermark在一定程度上对下游算子造成压力，所以只有在实时性要求非常高的场景才会选择Punctuated的方式进行Watermark的生成。

- Periodic - 周期性的（一定时间间隔或者达到一定的记录条数）产生一个Watermark。在实际的生产中Periodic的方式必须结合时间和积累条数两个维度继续周期性产生Watermark，否则在极端情况下会有很大的延时。

所以Watermark的生成方式需要根据业务场景的不同进行不同的选择。

#### Watermark的接口定义

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

#### Watermark解决EventTime问题

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

#### 多流的Watermark处理

在实际的流计算中往往一个job中会处理多个Source的数据，对Source的数据进行GroupBy分组，那么来自不同Source的相同key值会shuffle到同一个处理节点，并携带各自的Watermark，Apache Flink内部要保证Watermark要保持单调递增，多个Source的Watermark汇聚到一起时候可能不是单调自增的，这样的情况Apache Flink内部是如何处理的呢？如下图所示：

![1586591065285](/img/1586591065285.png)

Apache Flink内部实现每一个边上只能有一个递增的Watermark， 当出现多流携带Eventtime汇聚到一起(GroupBy or Union)时候，Apache Flink会选择所有流入的Eventtime中最小的一个向下游流出。从而保证watermark的单调递增和保证数据的完整性.如下图: 

![1586591258097](/img/1586591258097.png)

本节以一个流计算常见的乱序问题介绍了Apache Flink如何利用Watermark机制来处理乱序问题. 本篇内容在一定程度上也体现了EventTime Window中的Trigger机制依赖了Watermark(后续Window篇章会介绍)。Watermark机制是流计算中处理乱序，正确处理Late Event的核心手段。

#### 使用Event Time 处理实时数据

> 如下是一段 log 日志，我们根据该日志格式，来分析客户的下单量情况。
>
> 日志格式：
> 1581490623000,James,5
> 1581490624150,John,2
> …
> 接下来，我们从并行Source 和 非并行Source 两个方向，来使用 EventTime 处理实时数据。(接下来示例，设置延迟为0s，即不延迟)

#### 非并行Source

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

#### 并行Source

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

#### 结果分析

​	在并行Source一例中，当我们输入1581490624999,Mary,3时，我们看到控制台会直接帮我们输出计算结果。

​	但是，在使用 KafkaSource 时，我们连续输入了 3次1581490624999,Mary,3，我们才看到控制台帮我们输出计算了结果。

​	那这是为什么呢？这是 并行Source 和 非并行Source 的原因导致的（这里涉及到 KafkaSource 创建的 topic，有 3 个分区的原因，如下图所示）

![1586584999171](/img/1586584999171.png)