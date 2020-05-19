---
title: Spark 总结
date: "2020-04-28 13:43:11"
categories:
- 总结
tags:
- 总结
toc: true
typora-root-url: ..\..\..
---

### Spark 提交任务

**命令格式**

```shell
spark-submit \
--class $class \
--master spark://$spark_master_node \
--executor-memory $memory_use \
--total-executor-cores $core_use \
$jar [args]
```

**命令示例**

```shell
/usr/local/spark-1.5.2-bin-hadoop2.6/bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master spark://node1.itcast.cn:7077 \
--executor-memory 1G \
--total-executor-cores 2 \
/usr/local/spark-1.5.2-bin-hadoop2.6/lib/spark-examples-1.5.2-hadoop2.6.0.jar \
100
```

### Spark RDD 的依赖关系

RDD和它依赖的父RDD（s）的关系有两种不同的类型，即窄依赖（narrow dependency）和宽依赖（wide dependency）。

![1588062543122](/img/1588062543122.png)

####  窄依赖

窄依赖是指一个父RDD的Partition最多被子RDD的一个Partition使用

#### 宽依赖

宽依赖是指多个RDD的partition会依赖同一个父RDD的Partition

### RDD的血统

RDD只支持粗粒度转换，即在大量记录上执行的单个操作。将创建RDD的一系列Lineage（即血统）记录下来，以便恢复丢失的分区。RDD的Lineage会记录RDD的元数据信息和转换行为，当该RDD的部分分区数据丢失时，它可以根据这些信息来重新运算和恢复丢失的数据分区

### RDD的缓存

#### 缓存的作用

Spark速度非常快的原因之一，就是在不同操作中可以在内存中持久化或缓存个数据集。当持久化某个RDD后，每一个节点都将把计算的分片结果保存在内存中，并在对此RDD或衍生出的RDD进行的其他动作中重用。这使得后续的动作变得更加迅速。RDD相关的持久化和缓存，是Spark最重要的特征之一。可以说，缓存是Spark构建迭代式算法和快速交互式查询的关键。

#### 缓存的方式

RDD通过persist方法或cache方法可以将前面的计算结果缓存，但是并不是这两个方法被调用时立即缓存，而是触发后面的action时，该RDD将会被缓存在计算节点的内存中，并供后面重用。

![1588063966651](/img/1588063966651.png)

缓存有可能丢失，或者存储存储于内存的数据由于内存不足而被删除，RDD的缓存容错机制保证了即使缓存丢失也能保证计算的正确执行。通过基于RDD的一系列转换，丢失的数据会被重算，由于RDD的各个Partition是相对独立的，因此只需要计算丢失的部分即可，并不需要重算全部Partition。

### DAG的行程

DAG(Directed Acyclic Graph)叫做有向无环图，原始的RDD通过一系列的转换就就形成了DAG，根据RDD之间的依赖关系的不同将DAG划分成不同的Stage，对于窄依赖，partition的转换处理在Stage中完成计算。对于宽依赖，由于有Shuffle的存在，只能在parent RDD处理完成后，才能开始接下来的计算，因此宽依赖是划分Stage的依据。

![1588064038664](/img/1588064038664.png)

### Spark 中的checkpoint

checkpoint在spark中主要有两块应用：一块是在spark core中对RDD做checkpoint，可以将checkpoint RDD的依赖关系，RDD数据保存到可靠存储（如HDFS）以便数据恢复；另外一块是应用在spark streaming中，使用checkpoint用来保存DStreamGraph以及相关配置信息，以便在Driver崩溃重启的时候能够接着之前进度继续进行处理（如之前waiting batch的job会在重启后继续处理）。

本文主要将详细分析checkpoint在以上两种场景的读写过程。

#### checkpoint的使用方法

使用checkpoint对RDD做快照大体如下：

> ```scala
> sc.setCheckpointDir(checkpointDir.toString)
> val rdd = sc.makeRDD(1 to 20, numSlices = 1)
> rdd.checkpoint()
> ```

首先，设置checkpoint的目录（一般是hdfs目录），这个目录用来将RDD相关的数据（包括每个partition实际数据，以及partitioner（如果有的话））。然后在RDD上调用checkpoint的方法即可。

#### checkpoint写流程

可以看到checkpoint使用非常简单，设置checkpoint目录，然后调用RDD的checkpoint方法。针对checkpoint的写入流程，主要有以下四个问题：

Q1：RDD中的数据是什么时候写入的？是在rdd调用checkpoint方法时候吗？

Q2：在做checkpoint的时候，具体写入了哪些数据到HDFS了？

Q3：在对RDD做完checkpoint以后，对做RDD的本省又做了哪些收尾工作？

Q4：实际过程中，使用RDD做checkpoint的时候需要注意什么问题？

弄清楚了以上四个问题，我想对checkpoint的写过程也就基本清楚了。接下来将一一回答上面提出的问题。

A1：首先看一下RDD中checkpoint方法，可以看到在该方法中是只是新建了一个ReliableRDDCheckpintData的对象，并没有做实际的写入工作。实际触发写入的时机是在runJob生成该RDD后，调用RDD的doCheckpoint方法来做的。

A2：在经历调用RDD.doCheckpoint → RDDCheckpintData.checkpoint → ReliableRDDCheckpintData.doCheckpoint → ReliableRDDCheckpintData.writeRDDToCheckpointDirectory后，在writeRDDToCheckpointDirectory方法中可以看到：将作为一个单独的任务（RunJob）将RDD中每个parition的数据依次写入到checkpoint目录（writePartitionToCheckpointFile），此外如果该RDD中的partitioner如果不为空，则也会将该对象序列化后存储到checkpoint目录。所以，在做checkpoint的时候，写入的hdfs中的数据主要包括：RDD中每个parition的实际数据，以及可能的partitioner对象（writePartitionerToCheckpointDir）。

A3：在写完checkpoint数据到hdfs以后，将会调用rdd的markCheckpoined方法，主要斩断该rdd的对上游的依赖，以及将paritions置空等操作。

A4：通过A1，A2可以知道，在RDD计算完毕后，会再次通过RunJob将每个partition数据保存到HDFS。这样RDD将会计算两次，所以为了避免此类情况，最好将RDD进行cache。即1.1中rdd的推荐使用方法如下：

> ```
> sc.setCheckpointDir(checkpointDir.toString)
> val rdd = sc.makeRDD(1 to 20, numSlices = 1)
> rdd.cache()
> rdd.checkpoint()
> ```

#### checkpoint 读流程

在做完checkpoint后，获取原来RDD的依赖以及partitions数据都将从CheckpointRDD中获取。也就是说获取原来rdd中每个partition数据以及partitioner等对象，都将转移到CheckPointRDD中。

在CheckPointRDD的一个具体实现ReliableRDDCheckpintRDD中的compute方法中可以看到，将会从hdfs的checkpoint目录中恢复之前写入的partition数据。而partitioner对象（如果有）也会从之前写入hdfs的paritioner对象恢复。

总的来说，checkpoint读取过程是比较简单的。

### RDD分区规则

1. **通过集合方式指定**

   通过scala 集合方式parallelize生成rdd，

   如， val rdd = sc.parallelize(1 to 10)

   这种方式下，如果在parallelize操作时没有指定分区数，则

   **rdd的分区数 = sc.defaultParallelism**

1. **textFile 分区规则**

   1.如果textFile指定分区数量为0或者1的话，defaultMinPartitions值为1，则有多少个文件，就会有多少个分区。

   2.如果不指定默认分区数量，则默认分区数量为2，则会根据所有文件字节大小totalSize除以分区数量partitons的值goalSize，然后比较goalSize和hdfs指定分块大小（这里是32M）作比较，以较小的最为goalSize作为切分大小，对每个文件进行切分，若文件大于大于goalSize，则会生成该文件大小/goalSize + 1个分区。

   3.如果指定分区数量大于等于2，则默认分区数量为指定值，生成分区数量规则同2中的规则。

## Spark 任务提交

### 任务提交的主要四个阶段

DAG的生成 => stage切分 => task的生成 => 任务提交

![1585970735319](/img/1585970735319.png)

1. 构建DAG
   用户提交的job将首先被转换成一系列RDD并通过RDD之间的依赖关系构建DAG,然后将DAG提交到调度系统；
2. DAGScheduler将DAG切分stage（切分依据是shuffle）,将stage中生成的task以taskset的形式发送给TaskScheduler
3. Scheduler 调度task（根据资源情况将task调度到Executors）
4. Executors接收task，然后将task交给线程池执行。

### 任务提交详细步骤

1. spark集群启动后，Worker向Master注册信息

![img](https://www.linuxidc.com/upload/2018_02/180211173461711.png)

1. spark-submit命令提交程序后，driver和application也会向Master注册信息

![img](https://www.linuxidc.com/upload/2018_02/180211173461712.png)

![img](https://www.linuxidc.com/upload/2018_02/180211173461713.png)

3. 创建SparkContext对象：主要的对象包含DAGScheduler和TaskScheduler

4. Driver把Application信息注册给Master后，Master会根据App信息去Worker节点启动Executor

5. Executor内部会创建运行task的线程池，然后把启动的Executor反向注册给Dirver

6. DAGScheduler：负责把Spark作业转换成Stage的DAG（Directed Acyclic Graph有向无环图），根据宽窄依赖切分Stage，然后把Stage封装成TaskSet的形式发送个TaskScheduler；同时DAGScheduler还会处理由于Shuffle数据丢失导致的失败；

7. TaskScheduler：维护所有TaskSet，分发Task给各个节点的Executor（根据数据本地化策略分发Task），监控task的运行状态，负责重试失败的task；

8. 所有task运行完成后，SparkContext向Master注销，释放资源；

### Spark stage 切分流程

![1585970495777](/img/1585970495777.png)

#### 划分stage 的思路

park划分stage的整体思路是：从后往前推，遇到宽依赖就断开，划分为一个stage；遇到窄依赖就将这个RDD加入该stage中。

#### stage 作用

shuffle个复杂是业务逻辑（将多台机器上具有相同属性的数据聚合到一台机器上），如果有shuffle，那么就意味着前面阶段产生结果后，才能执行下一个阶段(下一个阶段的计算依赖上一个阶段的数据在同一个stage中);  在每一次shuffle之前会有多个算子，可以合并到一起执行，我们可以称这样的一个执行流程为pipeline（流水线，严格按照流程、顺序执行），整个阶段就是stage；

### Spark Driver  给Executor 提交task 时序图

![1585970924876](/img/1585970924876.png)

### Spark Executor 启动和任务接受和执行时序图

![1585970969792](/img/1585970969792.png)

## Spark Shuffle

### HashShuffle机制

#### HashShuffle概述

在spark-1.6版本之前，一直使用HashShuffle，在spark-1.6版本之后使用Sort-Base Shuffle，因为HashShuffle存在的不足所以就替换了HashShuffle.

我们知道，Spark的运行主要分为2部分：一部分是驱动程序，其核心是SparkContext；另一部分是Worker节点上Task,它是运行实际任务的。程序运行的时候，Driver和Executor进程相互交互：运行什么任务，即Driver会分配Task到Executor，Driver 跟 Executor 进行网络传输; 任务数据从哪儿获取，即Task要从 Driver 抓取其他上游的 Task 的数据结果，所以有这个过程中就不断的产生网络结果。其中，下一个 Stage 向上一个 Stage 要数据这个过程，我们就称之为 Shuffle。

#### 没有优化之前的HashShuffle机制

![1586768469257](/img/1586768469257.png)

1. 在HashShuffle没有优化之前，每一个ShufflleMapTask会为每一个ReduceTask创建一个bucket缓存，并且会为每一个bucket创建一个文件。这个bucket存放的数据就是经过Partitioner操作(默认是HashPartitioner)之后找到对应的bucket然后放进去，最后将数据刷新bucket缓存的数据到磁盘上，即对应的block file.
2. 然后ShuffleMapTask将输出作为MapStatus发送到DAGScheduler的MapOutputTrackerMaster，每一个MapStatus包含了每一个ResultTask要拉取的数据的位置和大小
3. ResultTask然后去利用BlockStoreShuffleFetcher向MapOutputTrackerMaster获取MapStatus，看哪一份数据是属于自己的，然后底层通过BlockManager将数据拉取过来
4. 拉取过来的数据会组成一个内部的ShuffleRDD，优先放入内存，内存不够用则放入磁盘，然后ResulTask开始进行聚合，最后生成我们希望获取的那个MapPartitionRDD

**这种方式的缺点**

如上图所示：在这里有1个worker，2个executor，每一个executor运行2个ShuffleMapTask，有三个ReduceTask，所以总共就有4 * 3=12个bucket和12个block file。

如果数据量较大，将会生成M*R个小文件，比如ShuffleMapTask有100个，ResultTask有100个，这就会产生100\*100=10000个小文件

bucket缓存很重要，需要将ShuffleMapTask所有数据都写入bucket，才会刷到磁盘，那么如果Map端数据过多，这就很容易造成内存溢出，尽管后面有优化，bucket写入的数据达到刷新到磁盘的阀值之后，就会将数据一点一点的刷新到磁盘，但是这样磁盘I/O就多了

#### 优化后的HashShuffle

![1586768527736](/img/1586768527736.png)

1. 每一个Executor进程根据核数，决定Task的并发数量，比如executor核数是2，就是可以并发运行两个task，如果是一个则只能运行一个task

2. 假设executor核数是1，ShuffleMapTask数量是M,那么它依然会根据ResultTask的数量R，创建R个bucket缓存，然后对key进行hash，数据进入不同的bucket中，每一个bucket对应着一个block file,用于刷新bucket缓存里的数据

3. 然后下一个task运行的时候，那么不会再创建新的bucket和block file，而是复用之前的task已经创建好的bucket和block file。即所谓同一个Executor进程里所有Task都会把相同的key放入相同的bucket缓冲区中

4. 这样的话，生成文件的数量就是(本地worker的executor数量\*executor的cores\*ResultTask数量)如上图所示，即2 \* 1\* 3 = 6个文件，每一个Executor的shuffleMapTask数量100,ReduceTask数量为100，那么

   > 未优化的HashShuffle的文件数是2 \*1\* 100\*100 =20000，优化之后的数量是2\*1\*100 = 200文件，相当于少了100倍

**这种方式的缺点**：

如果 Reducer 端的并行任务或者是数据分片过多的话则 Core * Reducer Task 依旧过大，也会产生很多小文件。

### Sort-Based Shuffle

#### Sort-Based Shuffle概述

**HashShuffle回顾**

1. io\GC\内存占用大:  HashShuffle写数据的时候，内存有一个bucket缓冲区，同时在本地磁盘有对应的本地文件，如果本地有文件，那么在内存应该也有文件句柄也是需要耗费内存的。也就是说，从内存的角度考虑，即有一部分存储数据，一部分管理文件句柄。如果Mapper分片数量为1000,Reduce分片数量为1000,那么总共就需要1000000个小文件。所以就会有很多内存消耗，频繁IO以及GC频繁或者出现内存溢出。
2. 容易造成网络异常:   而且Reducer端读取Map端数据时，Mapper有这么多小文件，就需要打开很多网络通道读取，很容易造成Reducer（下一个stage）通过driver去拉取上一个stage数据的时候，说文件找不到，其实不是文件找不到而是程序不响应，因为正在GC.

#### Sorted-Based Shuffle介绍

为了缓解Shuffle过程产生文件数过多和Writer缓存开销过大的问题，spark引入了类似于hadoop Map-Reduce的shuffle机制。该机制每一个ShuffleMapTask不会为后续的任务创建单独的文件，而是会将所有的Task结果写入同一个文件，并且对应生成一个索引文件。以前的数据是放在内存缓存中，等到数据完了再刷到磁盘，现在为了减少内存的使用，在内存不够用的时候，可以将输出溢写到磁盘，结束的时候，再将这些不同的文件联合内存的数据一起进行归并，从而减少内存的使用量。一方面文件数量显著减少，另一方面减少Writer缓存所占用的内存大小，而且同时避免GC的风险和频率。

![1586768569212](/img/1586768569212.png)

Sort-Based Shuffle有几种不同的策略：**BypassMergeSortShuffleWriter、SortShuffleWriter和UnasfeSortShuffleWriter。**

**对于BypassMergeSortShuffleWriter**

使用这个模式特点：

1. 主要用于处理不需要排序和聚合的Shuffle操作，所以数据是直接写入文件，数据量较大的时候，网络I/O和内存负担较重
2. 主要适合处理Reducer任务数量比较少的情况下
3. 将每一个分区写入一个单独的文件，最后将这些文件合并,减少文件数量；但是这种方式需要并发打开多个文件，对内存消耗比较大
4. 因为BypassMergeSortShuffleWriter这种方式比SortShuffleWriter更快，所以如果在Reducer数量不大，又不需要在map端聚合和排序，而且最终产生的文件数量少，尽量使用这种方式进行shuffle。 
5. Reducer的数目 <  spark.shuffle.sort.bypassMergeThrshold指定的阀值，就是用的是这种方式。

**对于SortShuffleWriter**

使用这个模式特点：

1. 比较适合数据量很大的场景或者集群规模很大
2. 引入了外部外部排序器，可以支持在Map端进行本地聚合或者不聚合
3. 如果外部排序器enable了spill功能，如果内存不够，可以先将输出溢写到本地磁盘，最后将内存结果和本地磁盘的溢写文件进行合并

> 另外这个Sort-Based Shuffle跟Executor核数没有关系，即跟并发度没有关系，它是每一个ShuffleMapTask都会产生一个data文件和index文件，所谓合并也只是将该ShuffleMapTask的各个partition对应的分区文件合并到data文件而已。所以这个就需要个Hash-BasedShuffle的consolidation机制区别开来。

## Spark 谓词下推

### 1.SparkSql

SparkSql 是架构在 Spark 计算框架之上的分布式 Sql 引擎，使用 DataFrame 和 DataSet 承载结构化和半结构化数据来实现数据复杂查询处理，提供的 DSL可以直接使用 scala 语言完成 Sql 查询，同时也使用  thriftserver 提供服务化的 Sql 查询功能。

SparkSql 提供了 DataSource API ，用户通过这套 API 可以自己开发一套 Connector，直接查询各类数据源，数据源包括 NoSql、RDBMS、搜索引擎以及 HDFS 等分布式文件系统上的文件等。和 SparkSql 类似的系统有 Hive、PrestoDB 以及 Impala，这类系统都属于所谓的" Sql on Hadoop "系统,每个都相当火爆，毕竟在这个不搞 SQL 就是耍流氓的年代，没 SQL 确实很难找到用户使用。

### 2.谓词下推解释

所谓谓词(predicate)，就是返回值是true或者false的函数，使用过scala或者spark的同学都知道有个filter方法，这个高阶函数传入的参数就是一个返回true或者false的函数。

但是如果是在sql语言中，没有方法，只有表达式。where后边的表达式起的作用正是过滤的作用，而这部分语句被sql层解析处理后，在数据库内部正是以谓词的形式呈现的。

**那么问题来了，谓词为什么要下推呢?** 

SparkSql中的谓词下推有两层含义，第一层含义是指由谁来完成数据过滤，第二层含义是指何时完成数据过滤。要解答这两个问题我们需要了解SparkSql的Sql语句处理逻辑，大致可以把SparkSql中的查询处理流程做如下的划分：

![img](https://oscimg.oschina.net/oscnet/5b3a43b8e7526224664836ce7300f555410.jpg)

SparkSql首先会对输入的Sql语句进行一系列的分析(Analyse)，包括词法解析(可以理解为搜索引擎中的分词这个过程)、语法分析以及语义分析(例如判断database或者table是否存在、group by必须和聚合函数结合等规则)；之后是执行计划的生成，包括逻辑计划和物理计划。其中在逻辑计划阶段会有很多的优化，对谓词的处理就在这个阶段完成；而物理计划则是RDD的DAG图的生成过程；这两步完成之后则是具体的执行了(也就是各种重量级的计算逻辑，例如join、groupby、filter以及distinct等)，这就会有各种物理操作符(RDD的Transformation)的乱入。

能够完成数据过滤的主体有两个，第一是分布式Sql层(在execute阶段)，第二个是数据源**。那么谓词下推的第一层含义就是指由Sql层的Filter操作符来完成过滤，还是由Scan操作符在扫描阶段完成过滤。**

上边提到，我们可以通过封装SparkSql的Data Source API完成各类数据源的查询，那么如果底层数据源无法高效完成数据的过滤，就会执行全局扫描，把每条相关的数据都交给SparkSql的Filter操作符完成过滤，虽然SparkSql使用的Code Generation技术极大的提高了数据过滤的效率，但是这个过程无法避免大量数据的磁盘读取，甚至在某些情况下会涉及网络IO(例如数据非本地化存储时)；如果底层数据源在进行扫描时能非常快速的完成数据的过滤，那么就会把过滤交给底层数据源来完成（至于哪些数据源能高效完成数据的过滤以及SparkSql又是如何完成高效数据过滤的则不是本文讨论的重点，会在其他系列的文章中介绍）。

那么**谓词下推第二层含义，即何时完成数据过滤则一般是在指连接查询中，是先对单表数据进行过滤再和其他表连接还是在先把多表进行连接再对连接后的临时表进行过滤，则是本系列文章要分析和讨论的重点。**

### 3. 谓词下推具体含义

基本概念：谓词下推（predicate pushdown）属于逻辑优化。优化器可以将谓词过滤下推到数据源，从而使物理执行跳过无关数据。在使用Parquet或者orcfile的情况下，更可能存在文件被整块跳过的情况，同时系统还通过字典编码把字符串对比转换为开销更小的整数对比。

说白了，就是把查询相关的条件下推到数据源进行提前的过滤操作，之所以这里说是查询相关的条件，而不直接说是where 后的条件，是因为sql语句中除了where后的有条件外，join时也有条件。本文讨论的主要就是join时的条件的处理。

![1588907608226](/img/1588907608226.png)

## Spark常见问题

### Spark 数据块有多少种不同的存储方式

1. RDD数据块：用来存储所缓存的RDD数据。
2. Shuffle数据块：用来存储持久化的Shuffle数据。
3. 广播变量数据块：用来存储所存储的广播变量数据。
4. 任务返回结果数据块：用来存储在存储管理模块内部的任务返回结果。通常情况下任务返回结果随任务一起通过Akka返回到Driver端。但是当任务返回结果很大时，会引起Akka帧溢出，这时的另一种方案是将返回结果以块的形式放入存储管理模块，然后在Driver端获取该数据块即可，因为存储管理模块内部数据块的传输是通过Socket连接的，因此就不会出现Akka帧溢出了。
   流式数据块：只用在Spark Streaming中，用来存储所接收到的流式数据块
5. 流式数据块：只用在Spark Streaming中，用来存储所接收到的流式数据块

### 哪些spark算子会有shuffle

1. 去重，distinct
2. 排序，groupByKey，reduceByKey等
3. 重分区，repartition，coalesce
4. 集合或者表操作，interection，join

### **Cache和persist有什么区别和联系？**
cache调用了persist方法，cache只有一个默认的缓存级别MEMORY_ONLY ，而persist可以根据情况设置其它的缓存级别。

### RDD是弹性数据集，“弹性”体现在哪里呢？你觉得RDD有哪些缺陷

自动进行内存和磁盘切换
基于lineage的高效容错
task如果失败会特定次数的重试
stage如果失败会自动进行特定次数的重试，而且只会只计算失败的分片
checkpoint【每次对RDD操作都会产生新的RDD，如果链条比较长，计算比较笨重，就把数据放在硬盘中】和persist 【内存或磁盘中对数据进行复用】(检查点、持久化)
数据调度弹性：DAG TASK 和资源管理无关
数据分片的高度弹性repartion
**缺陷**：
惰性计算的缺陷也是明显的：中间数据默认不会保存，每次动作操作都会对数据重复计算，某些计算量比较大的操作可能会影响到系统的运算效率

### RDD有多少种持久化方式？memory_only如果内存存储不了，会怎么操作？
cache和persist
memory_and_disk，放一部分到磁盘
MEMORY_ONLY_SER:同MEMORY_ONLY，但是会使用Java序列化方式，将Java对象序列化后进行持久化。可以减少内存开销，但是需要进行反序列化，因此会加大CPU开销。
MEMORY_AND_DSK_SER:同MEMORY_AND_DSK。但是使用序列化方式持久化Java对象。
DISK_ONLY:使用非序列化Java对象的方式持久化，完全存储到磁盘上。
MEMORY_ONLY_2或者MEMORY_AND_DISK_2等：如果是尾部加了2的持久化级别，表示会将持久化数据复用一份，保存到其他节点，从而在数据丢失时，不需要再次计算，只需要使用备份数据即可。

### 当GC时间占比很大可能的原因有哪些？对应的优化方法是？
垃圾回收的开销和对象合数成正比，所以减少对象的个数，就能大大减少垃圾回收的开销。序列化存储数据，每个RDD就是一个对象。缓存RDD占用的内存可能跟工作所需的内存打架，需要控制好

### Spark中repartition和coalesce异同？coalesce什么时候效果更高，为什么

repartition(numPartitions:Int):RDD[T]
coalesce(numPartitions:Int, shuffle:Boolean=false):RDD[T]

以上为他们的定义，区别就是repartition一定会触发shuffle，而coalesce默认是不触发shuffle的。

他们两个都是RDD的分区进行重新划分，repartition只是coalesce接口中shuffle为true的简易实现，（假设RDD有N个分区，需要重新划分成M个分区）

减少分区提高效率
### Groupbykey和reducebykey哪个性能更高，为什么？
reduceByKey性能高，更适合大数据集

### Scala里trait有什么功能，与class有何异同？什么时候用trait什么时候该用class
它可以被继承，而且支持多重继承，其实它更像我们熟悉的接口（interface），但它与接口又有不同之处是：
trait中可以写方法的实现，interface不可以（java8开始支持接口中允许写方法实现代码了），这样看起来trait又很像抽象类

