---
title: Spark 面试题整理
date: "2020-06-13 11:00:00"
categories:
- 总结
- 面试总结
tags:
- 面试
toc: true
typora-root-url: ..\..\..
---

## Spark 原理

### Spark 任务提交

#### 任务提交的主要四个阶段

DAG的生成 => stage切分 => task的生成 => 任务提交

![1585970735319](/img/1585970735319.png)

1. 构建DAG
   用户提交的job将首先被转换成一系列RDD并通过RDD之间的依赖关系构建DAG,然后将DAG提交到调度系统；
2. DAGScheduler将DAG切分stage（切分依据是shuffle）,将stage中生成的task以taskset的形式发送给TaskScheduler
3. Scheduler 调度task（根据资源情况将task调度到Executors）
4. Executors接收task，然后将task交给线程池执行。

#### 任务提交详细步骤

1. spark集群启动后，Worker向Master注册信息

![img](https://www.linuxidc.com/upload/2018_02/180211173461711.png)

1. spark-submit命令提交程序后，driver和application也会向Master注册信息

![img](https://www.linuxidc.com/upload/2018_02/180211173461712.png)

![img](https://www.linuxidc.com/upload/2018_02/180211173461713.png)

1. 创建SparkContext对象：主要的对象包含DAGScheduler和TaskScheduler
2. Driver把Application信息注册给Master后，Master会根据App信息去Worker节点启动Executor
3. Executor内部会创建运行task的线程池，然后把启动的Executor反向注册给Dirver
4. DAGScheduler：负责把Spark作业转换成Stage的DAG（Directed Acyclic Graph有向无环图），根据宽窄依赖切分Stage，然后把Stage封装成TaskSet的形式发送个TaskScheduler；同时DAGScheduler还会处理由于Shuffle数据丢失导致的失败；
5. TaskScheduler：维护所有TaskSet，分发Task给各个节点的Executor（根据数据本地化策略分发Task），监控task的运行状态，负责重试失败的task；
6. 所有task运行完成后，SparkContext向Master注销，释放资源；

### Spark stage 切分

![1585970495777](/img/1585970495777.png)

#### 划分stage 的思路

park划分stage的整体思路是：从后往前推，遇到宽依赖就断开，划分为一个stage；遇到窄依赖就将这个RDD加入该stage中。

#### stage 作用

这是个复杂是业务逻辑（将多台机器上具有相同属性的数据聚合到一台机器上:shuffle）如果有shuffle，那么就意味着前面阶段产生结果后，才能执行下一个阶段，下一个阶段的计算依赖上一个阶段的数据在同一个stage中，会有多个算子，可以合并到一起，我们很难‘’称其为pipeline（流水线，严格按照流程、顺序执行）

## 缓存、checkpoint

### 缓存

#### RDD缓存及其作用

Spark速度非常快的原因之一，就是在不同操作中可以在内存中持久化或缓存个数据集。

当持久化某个RDD后，每一个节点都将把计算的分片结果保存在内存中，并在对此RDD或衍生出的RDD进行的其他动作中重用。这使得后续的动作变得更加迅速。RDD相关的持久化和缓存，是Spark最重要的特征之一。可以说，缓存是Spark构建迭代式算法和快速交互式查询的关键。

### checkpoint

#### checkpoint作用

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

A1：首先看一下RDD中checkpoint方法，可以看到在该方法中是只是新建了一个ReliableRDDCheckpintData的对象，并没有做实际的写入工作。实际触发写入的时机是在runJob生成改RDD后，调用RDD的doCheckpoint方法来做的。

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

#### checkpoint 和cache 区别

- cache 机制保证了需要访问重复数据的应用（如迭代型算法和交互式应用）可以运行的更快。
- 与 Hadoop MapReduce job 不同的是 Spark 的逻辑/物理执行图可能很庞大，task 中 computing chain 可能会很长，计算某些 RDD 也可能会很耗时。这时，如果 task 中途运行出错，那么 task 的整个 computing chain 需要重算，代价太高。因此，有必要将计算代价较大的 RDD checkpoint 一下，这样，当下游 RDD 计算出错时，可以直接从 checkpoint 过的 RDD 那里读取数据继续算。

## Spark Shuffle

### 概述

Shuffle就是对数据进行重组，由于分布式计算的特性和要求，在实现细节上更加繁琐和复杂

在MapReduce框架，Shuffle是连接Map和Reduce之间的桥梁，Map阶段通过shuffle读取数据并输出到对应的Reduce；而Reduce阶段负责从Map端拉取数据并进行计算。在整个shuffle过程中，往往伴随着大量的磁盘和网络I/O。所以shuffle性能的高低也直接决定了整个程序的性能高低。Spark也会有自己的shuffle实现过程

![1586768403401](/img/1586768403401.png)

在DAG调度的过程中，Stage阶段的划分是根据是否有shuffle过程，也就是存在ShuffleDependency宽依赖的时候，需要进行shuffle,这时候会将作业job划分成多个Stage；并且在划分Stage的时候，构建ShuffleDependency的时候进行shuffle注册，获取后续数据读取所需要的ShuffleHandle,最终每一个job提交后都会生成一个ResultStage和若干个ShuffleMapStage，其中ResultStage表示生成作业的最终结果所在的Stage. ResultStage与ShuffleMapStage中的task分别对应着ResultTask与ShuffleMapTask。一个作业，除了最终的ResultStage外，其他若干ShuffleMapStage中各个ShuffleMapTask都需要将最终的数据根据相应的Partitioner对数据进行分组，然后持久化分区的数据。

 ![1586768416749](/img/1586768416749.png)

### HashShuffle机制(1.6之前)

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

如果数据量较大，将会生成M*R个小文件，比如ShuffleMapTask有100个，ResultTask有100个，这就会产生100*100=10000个小文件

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

### Sort-Based Shuffle(1.6之后)

#### Sort-Based Shuffle概述

**HashShuffle回顾**

1. io\GC\内存占用大:  HashShuffle写数据的时候，内存有一个bucket缓冲区，同时在本地磁盘有对应的本地文件，如果本地有文件，那么在内存应该也有文件句柄也是需要耗费内存的。也就是说，从内存的角度考虑，即有一部分存储数据，一部分管理文件句柄。如果Mapper分片数量为1000,Reduce分片数量为1000,那么总共就需要1000000个小文件。所以就会有很多内存消耗，频繁IO以及GC频繁或者出现内存溢出。
2. 容易造成网络异常:   而且Reducer端读取Map端数据时，Mapper有这么多小文件，就需要打开很多网络通道读取，很容易造成Reducer（下一个stage）通过driver去拉取上一个stage数据的时候，说文件找不到，其实不是文件找不到而是程序不响应，因为正在GC.

#### Sorted-Based Shuffle介绍

为了缓解Shuffle过程产生文件数过多和Writer缓存开销过大的问题，spark引入了类似于hadoop Map-Reduce的shuffle机制。该机制每一个ShuffleMapTask不会为后续的任务创建单独的文件，而是会将所有的Task结果写入同一个文件，并且对应生成一个索引文件。以前的数据是放在内存缓存中，等到数据完了再刷到磁盘，现在为了减少内存的使用，在内存不够用的时候，可以将输出溢写到磁盘，结束的时候，再将这些不同的文件联合内存的数据一起进行归并，从而减少内存的使用量。一方面文件数量显著减少，另一方面减少Writer缓存所占用的内存大小，而且同时避免GC的风险和频率。

![1586768569212](/img/1586768569212.png)

Sort-Based Shuffle有几种不同的策略：BypassMergeSortShuffleWriter、SortShuffleWriter和UnasfeSortShuffleWriter。

**使用BypassMergeSortShuffleWriter这个模式特点：**

1. 主要用于处理不需要排序和聚合的Shuffle操作，所以数据是直接写入文件，数据量较大的时候，网络I/O和内存负担较重
2. 主要适合处理Reducer任务数量比较少的情况下
3. 将每一个分区写入一个单独的文件，最后将这些文件合并,减少文件数量；但是这种方式需要并发打开多个文件，对内存消耗比较大
4. 因为BypassMergeSortShuffleWriter这种方式比SortShuffleWriter更快，所以如果在Reducer数量不大，又不需要在map端聚合和排序，而且最终产生的文件数量少，尽量使用这种方式进行shuffle。 
5. Reducer的数目 <  spark.shuffle.sort.bypassMergeThrshold指定的阀值，就是用的是这种方式。

**对于SortShuffleWriter这个模式特点**：

1. 比较适合数据量很大的场景或者集群规模很大
2. 引入了外部外部排序器，可以支持在Map端进行本地聚合或者不聚合
3. 如果外部排序器enable了spill功能，如果内存不够，可以先将输出溢写到本地磁盘，最后将内存结果和本地磁盘的溢写文件进行合并

> 另外这个Sort-Based Shuffle跟Executor核数没有关系，即跟并发度没有关系，它是每一个ShuffleMapTask都会产生一个data文件和index文件，所谓合并也只是将该ShuffleMapTask的各个partition对应的分区文件合并到data文件而已。所以这个就需要个Hash-BasedShuffle的consolidation机制区别开来。

## Spark优化

### hadoop 和 spark 在处理数据时，处理出现内存溢出的方法有哪些？

1. **map过程产生大量对象导致内存溢出**

   这种溢出的原因是在单个map中产生了大量的对象导致的。

   例如：rdd.map(x=>for(i <- 1 to 10000) yield i.toString)，这个操作在rdd中，每个对象都产生了10000个对象，这肯定很容易产生内存溢出的问题。针对这种问题，在不增加内存的情况下，可以通过减少每个Task的大小，以便达到每个Task即使产生大量的对象Executor的内存也能够装得下。具体做法可以在会产生大量对象的map操作之前调用repartition方法，分区成更小的块传入map。例如：rdd.repartition(10000).map(x=>for(i <- 1 to 10000) yield i.toString)。

   面对这种问题注意，不能使用rdd.coalesce方法，这个方法只能减少分区，不能增加分区，不会有shuffle的过程。

2. **数据不平衡导致内存溢出**

   数据不平衡除了有可能导致内存溢出外，也有可能导致性能的问题，解决方法和上面说的类似，就是调用repartition重新分区。这里就不再累赘了。

3. **coalesce调用导致内存溢出**

   这是我最近才遇到的一个问题，因为hdfs中不适合存小文件，所以Spark计算后如果产生的文件太小，我们会调用coalesce合并文件再存入hdfs中。

   但是这会导致一个问题，例如在coalesce之前有100个文件，这也意味着能够有100个Task，现在调用coalesce(10)，最后只产生10个文件，因为coalesce并不是shuffle操作，这意味着coalesce并不是按照我原本想的那样先执行100个Task，再将Task的执行结果合并成10个，而是从头到位只有10个Task在执行，原本100个文件是分开执行的，现在每个Task同时一次读取10个文件，使用的内存是原来的10倍，这导致了OOM。解决这个问题的方法是令程序按照我们想的先执行100个Task再将结果合并成10个文件，这个问题同样可以通过repartition解决，调用repartition(10)，因为这就有一个shuffle的过程，shuffle前后是两个Stage，一个100个分区，一个是10个分区，就能按照我们的想法执行。

4. **shuffle后内存溢出**

   shuffle内存溢出的情况可以说都是shuffle后，单个文件过大导致的。

   在Spark中，join，reduceByKey这一类型的过程，都会有shuffle的过程，在shuffle的使用，需要传入一个partitioner，大部分Spark中的shuffle操作，默认的partitioner都是HashPatitioner，默认值是父RDD中最大的分区数,这个参数通过spark.default.parallelism控制(在spark-sql中用spark.sql.shuffle.partitions) ， spark.default.parallelism参数只对HashPartitioner有效，所以如果是别的Partitioner或者自己实现的Partitioner就不能使用spark.default.parallelism这个参数来控制shuffle的并发量了。

   如果是别的partitioner导致的shuffle内存溢出，就需要从partitioner的代码增加partitions的数量。

5. **standalone模式下资源分配不均匀导致内存溢出**

   在standalone的模式下如果配置了–total-executor-cores 和 –executor-memory 这两个参数，但是没有配置–executor-cores这个参数的话，就有可能导致，每个Executor的memory是一样的，但是cores的数量不同，那么在cores数量多的Executor中，由于能够同时执行多个Task，就容易导致内存溢出的情况。这种情况的解决方法就是同时配置–executor-cores或者spark.executor.cores参数，确保Executor资源分配均匀。

### hadoop 和 spark 在处理数据时，处理出现内存溢出的方法有哪些？

1. 使用foreachPartitions替代foreach。
2. 设置num-executors参数

3. 设置executor-memory参数

4. executor-cores

5. driver-memory
6. spark.default.parallelism
7. spark.storage.memoryFraction

8. spark.shuffle.memoryFraction

### reduceByKey 与 groupByKey 哪个性能更高?

1.reduceByKey将结果发送给reducer之前在本地进行merge（the merging locally on each mapper before sending results to a reducer），这样数据量会大幅度减小，从而减小传输，保证reduce端能够更快的进行结果计算。

2.groupByKey会对每一个RDD中的value值进行聚合形成一个序列(Iterator)，所有的键值对都会被移动，此操作发生在reduce端，大量的数据通过网络进行传输，效率低下。

### 在 2.5亿个整数中，找出不重复的整数，注意：内存不足以容纳 2.5亿个整数。

采用2-Bitmap（每个数分配2bit，00表示不存在，01表示出现一次，10表示多次，11无意义）进行，共需内存2*10^8 * 2.5bit=1 GB内存，还可以接受。然后扫描这2.5亿个整数，查看Bitmap中相对应位，如果是00变01，01变10，10保持不变。所描完事后，查看bitmap，把对应位是01的整数输出即可。