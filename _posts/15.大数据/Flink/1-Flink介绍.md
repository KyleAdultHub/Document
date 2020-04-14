---
title: Flink 介绍
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

## 一、Flink 简介

Apache Flink 诞生于柏林工业大学的一个研究性项目，原名 StratoSphere 。2014 年，由 StratoSphere 项目孵化出 Flink，并于同年捐赠 Apache，之后成为 Apache 的顶级项目。2019 年 1 年，阿里巴巴收购了 Flink 的母公司 Data Artisans，并宣布开源内部的 Blink，Blink 是阿里巴巴基于 Flink 优化后的版本，增加了大量的新功能，并在性能和稳定性上进行了各种优化，经历过阿里内部多种复杂业务的挑战和检验。同时阿里巴巴也表示会逐步将这些新功能和特性 Merge 回社区版本的 Flink 中，因此 Flink 成为目前最为火热的大数据处理框架。

简单来说，Flink 是一个分布式的流处理框架，它能够对有界和无界的数据流进行高效的处理。Flink 的核心是流处理，当然它也能支持批处理，Flink 将批处理看成是流处理的一种特殊情况，即数据流是有明确界限的。这和 Spark Streaming 的思想是完全相反的，Spark Streaming 的核心是批处理，它将流处理看成是批处理的一种特殊情况， 即把数据流进行极小粒度的拆分，拆分为多个微批处理。

Flink 有界数据流和无界数据流：

![1586404205174](/img/1586404205174.png)

Spark Streaming 数据流的拆分：

![1586404251340](/img/1586404251340.png)

## 二、Flink 核心架构

### 2.0 Flink组件栈

Flink 采用分层的架构设计，从而保证各层在功能和职责上的清晰。如下图所示，由上而下分别是 API & Libraries 层、Runtime 核心层以及物理部署层：

![1586404760423](/img/1586404760423.png)

### 2.1 API & Libraries 层

这一层主要提供了编程 API 和 顶层类库：

- 编程 API : 用于进行流处理的 DataStream API 和用于进行批处理的 DataSet API；
- 顶层类库：包括用于复杂事件处理的 CEP 库；用于结构化数据查询的 SQL & Table 库，以及基于批处理的机器学习库 FlinkML 和 图形处理库 Gelly。

### 2.2 Runtime 核心层

这一层是 Flink 分布式计算框架的核心实现层，包括作业转换，任务调度，资源分配，任务执行等功能，基于这一层的实现，可以在流式引擎下同时运行流处理程序和批处理程序。比如：支持分布式Stream处理、JobGraph到ExecutionGraph的映射、调度等等，为上层API层提供基础服务

### 2.3 物理部署层

Flink 的物理部署层，用于支持在不同平台上部署运行 Flink 应用。

主要涉及了Flink的部署模式，Flink支持多种部署模式：本地、集群（Standalone/YARN）、云（GCE/EC2）

![1586489584052](/img/1586489584052.png)

## 三、Flink 分层 API

在上面介绍的 API & Libraries 这一层，Flink 又进行了更为具体的划分。具体如下：

![1586404927178](/img/1586404927178.png)

按照如上的层次结构，API 的一致性由下至上依次递增，接口的表现能力由下至上依次递减，各层的核心功能如下：

### 3.1 SQL & Table API

SQL & Table API 同时适用于批处理和流处理，这意味着你可以对有界数据流和无界数据流以相同的语义进行查询，并产生相同的结果。除了基本查询外， 它还支持自定义的标量函数，聚合函数以及表值函数，可以满足多样化的查询需求。 

### 3.2 DataStream & DataSet API

DataStream &  DataSet API 是 Flink 数据处理的核心 API，支持使用 Java 语言或 Scala 语言进行调用，提供了数据读取，数据转换和数据输出等一系列常用操作的封装。

### 3.3 Stateful Stream Processing

Stateful Stream Processing 是最低级别的抽象，它通过 Process Function 函数内嵌到 DataStream API 中。 Process Function 是 Flink 提供的最底层 API，具有最大的灵活性，允许开发者对于时间和状态进行细粒度的控制。

## 四、Flink 集群架构

### 4.1  核心组件

按照上面的介绍，Flink 核心架构的第二层是 Runtime 层， 该层采用标准的 Master - Slave 结构， 其中，Master 部分又包含了三个核心组件：Dispatcher、ResourceManager 和 JobManager，而 Slave 则主要是 TaskManager 进程。它们的功能分别如下：

- **JobManagers** (也称为 *masters*) ：JobManagers 接收由 Dispatcher 传递过来的执行程序，该执行程序包含了作业图 (JobGraph)，逻辑数据流图 (logical dataflow graph) 及其所有的 classes 文件以及第三方类库 (libraries) 等等 。紧接着 JobManagers 会将 JobGraph 转换为执行图 (ExecutionGraph)，然后向 ResourceManager 申请资源来执行该任务，一旦申请到资源，就将执行图分发给对应的 TaskManagers 。因此每个作业 (Job) 至少有一个 JobManager；高可用部署下可以有多个 JobManagers，其中一个作为 *leader*，其余的则处于 *standby* 状态。
- **TaskManagers** (也称为 *workers*) : TaskManagers 负责实际的子任务 (subtasks) 的执行，每个 TaskManagers 都拥有一定数量的 slots。Slot 是一组固定大小的资源的合集 (如计算能力，存储空间)。TaskManagers 启动后，会将其所拥有的 slots 注册到 ResourceManager 上，由 ResourceManager 进行统一管理。
- **Dispatcher**：负责接收客户端提交的执行程序，并传递给 JobManager 。除此之外，它还提供了一个 WEB UI 界面，用于监控作业的执行情况。
- **Client**： 用户提交一个Flink程序时，会首先创建一个Client，该Client首先会对用户提交的Flink程序进行预处理，并提交到Flink集群， Client会将用户提交的Flink程序组装一个JobGraph， 并且是以JobGraph的形式提交的
- **ResourceManager** ：负责管理 slots 并协调集群资源。ResourceManager 接收来自 JobManager 的资源请求，并将存在空闲 slots 的 TaskManagers 分配给 JobManager 执行任务。Flink 基于不同的部署平台，如 YARN , Mesos，K8s 等提供了不同的资源管理器，当 TaskManagers 没有足够的 slots 来执行任务时，它会向第三方平台发起会话来请求额外的资源。

![1586410450743](/img/1586410450743.png)

### 4.2  Task & SubTask

上面我们提到：TaskManagers 实际执行的是 SubTask，而不是 Task，这里解释一下两者的区别：

在执行分布式计算时，Flink 将可以链接的操作 (operators) 链接到一起，这就是 Task。之所以这样做， 是为了减少线程间切换和缓冲而导致的开销，在降低延迟的同时可以提高整体的吞吐量。 但不是所有的 operator 都可以被链接，如下 keyBy 等操作会导致网络 shuffle 和重分区，因此其就不能被链接，只能被单独作为一个 Task。  简单来说，一个 Task 就是一个可以链接的最小的操作链 (Operator Chains) 。如下图，source 和 map 算子被链接到一块，因此整个作业就只有三个 Task：

![1586410843615](/img/1586410843615.png)

解释完 Task ，我们在解释一下什么是 SubTask，其准确的翻译是： *A subtask is one parallel slice of a task*，即一个 Task 可以按照其并行度拆分为多个 SubTask。如上图，source & map 具有两个并行度，KeyBy 具有两个并行度，Sink 具有一个并行度，因此整个虽然只有 3 个 Task，但是却有 5 个 SubTask。Jobmanager 负责定义和拆分这些 SubTask，并将其交给 Taskmanagers 来执行，每个 SubTask 都是一个单独的线程。

### 4.3  资源管理

理解了 SubTasks ，我们再来看看其与 Slots 的对应情况。一种可能的分配情况如下：

![1586410901685](/img/1586410901685.png)

这时每个 SubTask 线程运行在一个独立的 TaskSlot， 它们共享所属的 TaskManager 进程的TCP 连接（通过多路复用技术）和心跳信息 (heartbeat messages)，从而可以降低整体的性能开销。此时看似是最好的情况，但是每个操作需要的资源都是不尽相同的，这里假设该作业 keyBy 操作所需资源的数量比 Sink 多很多 ，那么此时 Sink 所在 Slot 的资源就没有得到有效的利用。

基于这个原因，Flink 允许多个 subtasks 共享 slots，即使它们是不同 tasks 的 subtasks，但只要它们来自同一个 Job 就可以。假设上面 souce & map 和 keyBy 的并行度调整为 6，而 Slot 的数量不变，此时情况如下：

![1586411714405](/img/1586411714405.png)

可以看到一个 Task Slot 中运行了多个 SubTask 子任务，此时每个子任务仍然在一个独立的线程中执行，只不过共享一组 Sot 资源而已。那么 Flink 到底如何确定一个 Job 至少需要多少个 Slot 呢？Flink 对于这个问题的处理很简单，默认情况一个 Job 所需要的 Slot 的数量就等于其 Operation 操作的最高并行度。如下， A，B，D 操作的并行度为 4，而 C，E 操作的并行度为 2，那么此时整个 Job 就需要至少四个 Slots 来完成。通过这个机制，Flink 就可以不必去关心一个 Job 到底会被拆分为多少个 Tasks 和 SubTasks。

![1586411745166](/img/1586411745166.png)

### 4.4 组件通讯

> - Flink是基于Master-Slave风格的架构
> - Flink集群启动时，会启动一个JobManager进程、至少一个TaskManager进程

Flink 的所有组件都基于 Actor System 来进行通讯。Actor system是多种角色的 actor 的容器，它提供调度，配置，日志记录等多种服务，并包含一个可以启动所有 actor 的线程池，如果 actor 是本地的，则消息通过共享内存进行共享，但如果 actor 是远程的，则通过 RPC 的调用来传递消息。

![1586411910002](/img/1586411910002.png)

## 五、Flink编程模型

> - Flink程序的基础构建模块是流(streams) 与 转换(transformations)
> - 每一个数据流起始于一个或多个 source，并终止于一个或多个 sink

每一个数据流起始于一个或多个 source，并终止于一个或多个 sink

下面是一个由Flink程序映射为Streaming Dataflow的示意图:

![1586490354970](/img/1586490354970.png)

并行数据流示意图:

![1586490366945](/img/1586490366945.png)

## 五、Flink的优势

- 支持高吞吐、低延迟、高性能的流处理
- 支持高度灵活的窗口（Window）操作
- 支持有状态计算的Exactly-once语义
- 提供DataStream API和DataSet API

![1586489825589](/img/1586489825589.png)

提供DataStream API和DataSet API

## 六、Flink 的优点

最后基于上面的介绍，来总结一下 Flink 的优点：

- Flink 是基于事件驱动 (Event-driven) 的应用，能够同时支持流处理和批处理；
- 基于内存的计算，能够保证高吞吐和低延迟，具有优越的性能表现；
- 支持精确一次 (Exactly-once) 语意，能够完美地保证一致性和正确性；
- 分层 API ，能够满足各个层次的开发需求；
- 支持高可用配置，支持保存点机制，能够提供安全性和稳定性上的保证；
- 多样化的部署方式，支持本地，远端，云端等多种部署方案；
- 具有横向扩展架构，能够按照用户的需求进行动态扩容；
- 活跃度极高的社区和完善的生态圈的支持。

## 七、 开发环境搭建

### 使用 IDEA 构建

如果你使用的是开发工具是 IDEA ，可以直接在项目创建页面选择 Maven Flink Archetype 进行项目初始化：

![1586412347986](/img/1586412347986.png)

如果你的 IDEA 没有上述 Archetype， 可以通过点击右上角的 `ADD ARCHETYPE` ，来进行添加，依次填入所需信息，这些信息都可以从上述的 `archetype:generate ` 语句中获取。点击  `OK` 保存后，该 Archetype 就会一直存在于你的 IDEA 中，之后每次创建项目时，只需要直接选择该 Archetype 即可：

![1586412365845](/img/1586412365845.png)

选中 Flink Archetype ，然后点击 `NEXT` 按钮，之后的所有步骤都和正常的 Maven 工程相同。

### 项目结构

创建完成后的自动生成的项目结构如下：

![1586412395452](/img/1586412395452.png)

其中 BatchJob 为批处理的样例代码，源码如下：

```scala
import org.apache.flink.api.scala._

object BatchJob {
  def main(args: Array[String]) {
    val env = ExecutionEnvironment.getExecutionEnvironment
      ....
    env.execute("Flink Batch Scala API Skeleton")
  }
}
```

getExecutionEnvironment 代表获取批处理的执行环境，如果是本地运行则获取到的就是本地的执行环境；如果在集群上运行，得到的就是集群的执行环境。如果想要获取流处理的执行环境，则只需要将 `ExecutionEnvironment` 替换为 `StreamExecutionEnvironment`， 对应的代码样例在 StreamingJob 中：

```scala
import org.apache.flink.streaming.api.scala._

object StreamingJob {
  def main(args: Array[String]) {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
      ...
    env.execute("Flink Streaming Scala API Skeleton")
  }
}

```

需要注意的是对于流处理项目 `env.execute()` 这句代码是必须的，否则流处理程序就不会被执行，但是对于批处理项目则是可选的。

### 主要依赖

基于 Maven 骨架创建的项目主要提供了以下核心依赖：其中 `flink-scala` 用于支持开发批处理程序 ；`flink-streaming-scala` 用于支持开发流处理程序 ；`scala-library` 用于提供 Scala 语言所需要的类库。如果在使用 Maven 骨架创建时选择的是 Java 语言，则默认提供的则是 `flink-java` 和 `flink-streaming-java` 依赖。

```xml
<!-- Apache Flink dependencies -->
<!-- These dependencies are provided, because they should not be packaged into the JAR file. -->
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-scala_${scala.binary.version}</artifactId>
    <version>${flink.version}</version>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-streaming-scala_${scala.binary.version}</artifactId>
    <version>${flink.version}</version>
    <scope>provided</scope>
</dependency>

<!-- Scala Library, provided by Flink as well. -->
<dependency>
    <groupId>org.scala-lang</groupId>
    <artifactId>scala-library</artifactId>
    <version>${scala.version}</version>
    <scope>provided</scope>
</dependency>
```

需要特别注意的以上依赖的 `scope` 标签全部被标识为 provided ，这意味着这些依赖都不会被打入最终的 JAR 包。因为 Flink 的安装包中已经提供了这些依赖，位于其 lib 目录下，名为  `flink-dist_*.jar`  ，它包含了 Flink 的所有核心类和依赖：

![1586412435189](/img/1586412435189.png)

 `scope` 标签被标识为 provided 会导致你在 IDEA 中启动项目时会抛出 ClassNotFoundException 异常。基于这个原因，在使用 IDEA 创建项目时还自动生成了以下 profile 配置：

```xml
<!-- This profile helps to make things run out of the box in IntelliJ -->
<!-- Its adds Flink's core classes to the runtime class path. -->
<!-- Otherwise they are missing in IntelliJ, because the dependency is 'provided' -->
<profiles>
    <profile>
        <id>add-dependencies-for-IDEA</id>

        <activation>
            <property>
                <name>idea.version</name>
            </property>
        </activation>

        <dependencies>
            <dependency>
                <groupId>org.apache.flink</groupId>
                <artifactId>flink-scala_${scala.binary.version}</artifactId>
                <version>${flink.version}</version>
                <scope>compile</scope>
            </dependency>
            <dependency>
                <groupId>org.apache.flink</groupId>
                <artifactId>flink-streaming-scala_${scala.binary.version}</artifactId>
                <version>${flink.version}</version>
                <scope>compile</scope>
            </dependency>
            <dependency>
                <groupId>org.scala-lang</groupId>
                <artifactId>scala-library</artifactId>
                <version>${scala.version}</version>
                <scope>compile</scope>
            </dependency>
        </dependencies>
    </profile>
</profiles>
```

在 id 为 `add-dependencies-for-IDEA` 的 profile 中，所有的核心依赖都被标识为 compile，此时你可以无需改动任何代码，只需要在 IDEA 的 Maven 面板中勾选该 profile，即可直接在 IDEA 中运行 Flink 项目：

![1586413547337](/img/1586413547337.png)

### 词频统计案例

项目创建完成后，可以先书写一个简单的词频统计的案例来尝试运行 Flink 项目，以下以 Scala 语言为例，分别介绍流处理程序和批处理程序的编程示例：

1. 批处理示例

   ```scala
   import org.apache.flink.api.scala._
   
   object WordCountBatch {
   
     def main(args: Array[String]): Unit = {
       val benv = ExecutionEnvironment.getExecutionEnvironment
       val dataSet = benv.readTextFile("D:\\wordcount.txt")
       dataSet.flatMap { _.toLowerCase.split(",")}
               .filter (_.nonEmpty)
               .map { (_, 1) }
               .groupBy(0)
               .sum(1)
               .print()
     }
   }
   ```

   其中 `wordcount.txt` 中的内容如下：

   ```shell
   a,a,a,a,a
   b,b,b
   c,c
   d,d
   ```

   本机不需要配置其他任何的 Flink 环境，直接运行 Main 方法即可

2. 流处理示例

   ```scala
   import org.apache.flink.streaming.api.scala._
   import org.apache.flink.streaming.api.windowing.time.Time
   
   object WordCountStreaming {
   
     def main(args: Array[String]): Unit = {
   
       val senv = StreamExecutionEnvironment.getExecutionEnvironment
   
       val dataStream: DataStream[String] = senv.socketTextStream("192.168.0.229", 9999, '\n')
       dataStream.flatMap { line => line.toLowerCase.split(",") }
                 .filter(_.nonEmpty)
                 .map { word => (word, 1) }
                 .keyBy(0)
                 .timeWindow(Time.seconds(3))
                 .sum(1)
                 .print()
       senv.execute("Streaming WordCount")
     }
   }
   ```

   这里以监听指定端口号上的内容为例，使用以下命令来开启端口服务：

   ```shell
   nc -lk 9999
   ```

   之后输入测试数据即可观察到流处理程序的处理情况。

## 八、使用 Scala Shell

对于日常的 Demo 项目，如果你不想频繁地启动 IDEA 来观察测试结果，可以像 Spark 一样，直接使用 Scala Shell 来运行程序，这对于日常的学习来说，效果更加直观，也更省时。Flink 安装包的下载地址如下：

```shell
https://flink.apache.org/downloads.html
```

Flink 大多数版本都提供有 Scala 2.11 和 Scala 2.12 两个版本的安装包可供下载：

![1586414175349](/img/1586414175349.png)

下载完成后进行解压即可，Scala Shell 位于安装目录的 bin 目录下，直接使用以下命令即可以本地模式启动：

```shell
./start-scala-shell.sh local
```

命令行启动完成后，其已经提供了批处理 （benv 和 btenv）和流处理（senv 和 stenv）的运行环境，可以直接运行 Scala Flink 程序，示例如下：

![1586414226321](/img/1586414226321.png)

最后解释一个常见的异常：这里我使用的 Flink 版本为 1.9.1，启动时会抛出如下异常。这里因为按照官方的说明，目前所有 Scala 2.12 版本的安装包暂时都不支持 Scala Shell，所以如果想要使用 Scala Shell，只能选择 Scala 2.11 版本的安装包。 

```shell
[root@hadoop001 bin]# ./start-scala-shell.sh local
错误: 找不到或无法加载主类 org.apache.flink.api.scala.FlinkShell
```





