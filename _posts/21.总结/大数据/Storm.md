---
title: Storm 总结
date: "2020-04-28 13:43:11"
categories:
- 总结
tags:
- 总结
toc: true
typora-root-url: ..\..\..
---

### Storm架构

![1588128504221](/img/1588128504221.png)

- Nimbus：负责资源分配和任务调度。
- Supervisor：负责接受nimbus分配的任务，启动和停止属于自己管理的worker进程。---通过配置文件设置当前supervisor上启动多少个worker。
- Worker：运行具体处理组件逻辑的进程。Worker运行的任务类型只有两种，一种是Spout任务，一种是Bolt任务。
- Task：worker中每一个spout/bolt的线程称为一个task. 在storm0.8之后，task不再与物理线程对应，不同spout/bolt的task可能会共享一个物理线程，该线程称为executor。

### Storm 编程模型

 ![1583829319405](/img/1583829319405.png)

- Topology：Storm中运行的一个实时应用程序的名称。（拓扑）

- Spout：在一个topology中获取**源数据流**的组件。

  通常情况下spout会从外部数据源中读取数据，然后转换为topology内部的源数据。

- Bolt：接受数据然后执行处理的组件,用户可以在其中执行自己想要的操作。

- Tuple：一次消息传递的基本单元，理解为一组消息就是一个Tuple。

- StreamGrouping:数据分组策略， 一共有7种

### Storm流式计算生命周期

storm 执行流式数据处理函数的生命周期主要是通过Spout和Bolt 这两个组件的生命周期

**Spout组件涉及到的方法有：**

- declareOutputFields()   (调用一次)
- open()     (调用一次)
- activate()      (调用一次)
- nextTuple()     （循环调用 ）
- disactive()        (调用一次)

**Bolt组件涉及到的方法有**

- declareOutputFileds()      (调用一次)
- prepare()     (调用一次)
- execute()    （循环执行）

**执行顺序？**

1. 在客户端将jar包提交到集群上的时候，执行 spout 、bolt 的构造方法以及declareOutputFields（）
2. spout执行open方法一次，得到conf一些配置信息
3. Bolt 中 prepare 方法只执行一次
4. Spout 中 nextTuple不断的循环执行
5. Blot 中 execute  不断的循环执行

### Stream Grouping

#### Stream Grouping 的作用

Storm 的Tuple  从 Spout 中 分发到 Bolt， 以及从Bolt 分发的Bolt 的过程中，Storm定义了一些内置的分发方法和规则，

这些规则通过一些条件或者随机将具有相同特征的tuple分发到同一Bolt中进行处理，这样有利于对不同特征的数据进行集中统计

下面介绍一些常用的Grouping 种类

#### Stream Grouping 种类

- Shuffle Grouping: 随机分组， 随机派发stream里面的tuple，保证每个bolt接收到的tuple数目大致相同。
- Fields Grouping：按字段分组，比如按userid来分组，具有同样userid的tuple会被分到相同的Bolts里的一个task，而不同的userid则会被分配到不同的bolts里的task。
- All Grouping：广播发送，对于每一个tuple，所有的bolts都会收到。
- Global Grouping：全局分组， 这个tuple被分配到storm中的一个bolt的其中一个task。再具体一点就是分配给id值最低的那个task。
- Non Grouping：不分组，这stream grouping个分组的意思是说stream不关心到底谁会收到它的tuple。目前这种分组和Shuffle grouping是一样的效果，有一点不同的是storm会把这个bolt放到这个bolt的订阅者同一个线程里面去执行， 即减少跨主机socket通讯。
- Direct Grouping： 直接分组， 这是一种比较特别的分组方法，用这种分组意味着消息的发送者指定由消息接收者的哪个task处理这个消息。只有被声明为Direct Stream的消息流可以声明这种分组方法。而且这种消息tuple必须使用emitDirect方法来发射。消息处理者可以通过TopologyContext来获取处理它的消息的task的id （OutputCollector.emit方法也会返回task的id）。

- Local or shuffle grouping：如果目标bolt有一个或者多个task在同一个工作进程中，tuple将会被随机发生给这些tasks。否则，和普通的Shuffle Grouping行为一致。

## Storm 程序的并发机制

### 概念

- Workers (JVMs): 在一个物理节点上可以运行一个或多个独立的JVM 进程。一个Topology可以包含一个或多个worker(并行的跑在不同的物理机上), 所以worker process就是执行一个topology的子集, **并且worker只能对应于一个topology** 
- Executors (threads): 在一个worker JVM进程中运行着多个Java线程。一个executor线程可以执行一个或多个tasks。但一般默认每个executor只执行一个task。一个worker可以包含一个或多个executor, 每个component (spout或bolt)至少对应于一个executor, 所以可以说executor执行一个compenent的子集, 同时一个executor只能对应于一个component。 
- Tasks(bolt/spout instances)：Task就是具体的处理逻辑对象，每一个Spout和Bolt会被当作很多task在整个集群里面执行。每一个task对应到一个线程，而stream grouping则是定义怎么从一堆task发射tuple到另外一堆task。你可以调用TopologyBuilder.setSpout和TopBuilder.setBolt来设置并行度 — 也就是有多少个task。 

### 配置并行度

#### 通过修改配置文件和代码配置冰心额度

- 对于并发度的配置, 在storm里面可以在多个地方进行配置, 优先级为：

  defaults.yaml < storm.yaml < topology-specific configuration 

  < internal component-specific configuration < external component-specific configuration 

- worker processes的数目, 可以通过配置文件和代码中配置, worker就是执行进程, 所以考虑并发的效果, 数目至少应该大于machines的数目 

- executor的数目, component的并发线程数，只能在代码中配置(通过setBolt和setSpout的参数), 例如, setBolt("green-bolt", new GreenBolt(), 2) 

- tasks的数目, 可以不配置, 默认和executor1:1, 也可以通过setNumTasks()配置 

- Topology的worker数通过config设置，即执行该topology的worker（java）进程数。它可以通过 storm rebalance 命令任意调整。 

  ```java
  Config conf = newConfig();
  conf.setNumWorkers(2); //用2个worker
  topologyBuilder.setSpout("blue-spout", newBlueSpout(), 2); //设置2个并发度
  topologyBuilder.setBolt("green-bolt", newGreenBolt(), 2).setNumTasks(4).shuffleGrouping("blue-spout"); //设置2个并发度，4个任务
  topologyBuilder.setBolt("yellow-bolt", newYellowBolt(), 6).shuffleGrouping("green-bolt"); //设置6个并发度
  StormSubmitter.submitTopology("mytopology", conf, topologyBuilder.createTopology());
  ```

#### 动态改变任务的并行度

Storm支持在不 restart topology 的情况下, 动态的改变(增减) worker processes 的数目和 executors 的数目, 称为rebalancing. 通过Storm web UI，或者通过storm rebalance命令实现： 

```shell
storm rebalance mytopology -n 5 -e a-spout=3 -e b-bolt=10
```

## Storm 的技术分析

### Storm的通讯技术

#### 通讯机制

Worker间的通信经常需要通过网络跨节点进行，Storm使用ZeroMQ或Netty(0.9以后默认使用)作为进程间通信的消息框架。

Worker进程内部通信：**不同worker的thread通信使用LMAX Disruptor来完成。**

不同topologey之间的通信，Storm不负责，需要自己想办法实现，例如使用kafka等；

#### Worker 进程间通讯机制

worker进程间消息传递机制，消息的接收和处理的大概流程见下图

![1583835238933](/img/1583835238933.png)



- 对于worker进程来说，为了管理流入和传出的消息，每个worker进程有一个独立的接收线程(对配置的TCP端口supervisor.slots.ports进行监听);
- 对应Worker接收线程，每个worker存在一个独立的发送线程，它负责从worker的transfer-queue中读取消息，并通过网络发送给其他worker
- 每个executor有自己的incoming-queue和outgoing-queue。
- Worker接收线程将收到的消息通过task编号传递给对应的executor(一个或多个)的incoming-queues;
- 每个executor有单独的线程分别来处理spout/bolt的业务逻辑，业务逻辑输出的中间数据会存放在outgoing-queue中，当executor的outgoing-queue中的tuple达到一定的阀值，executor的发送线程将批量获取outgoing-queue中的tuple,并发送到transfer-queue中。
- 每个worker进程控制一个或多个executor线程，用户可在代码中进行配置。其实就是我们在代码中设置的并发度个数。

#### Worker 进程间通讯机制分析

![1583835379230](/img/1583835379230.png)

- Worker接受线程通过网络接受数据，并根据Tuple中包含的taskId，匹配到对应的executor；然后根据executor找到对应的incoming-queue，将数据存发送到incoming-queue队列中。
- 业务逻辑执行现成消费incoming-queue的数据，通过调用Bolt的execute(xxxx)方法，将Tuple作为参数传输给用户自定义的方法
- 业务逻辑执行完毕之后，将计算的中间数据发送给outgoing-queue队列，当outgoing-queue中的tuple达到一定的阀值，executor的发送线程将批量获取outgoing-queue中的tuple,并发送到Worker的transfer-queue中
- Worker发送线程消费transfer-queue中数据，计算Tuple的目的地，连接不同的node+port将数据通过网络传输的方式传送给另一个的Worker。
- 另一个worker执行以上步骤1的操作。

### Storm 任务提交过程

**过程示意图**

![1583837643662](/img/1583837643662.png)

**详细任务提交过程**

![1583837747089](/img/1583837747089.png)

### Storm 的容错机制

#### 总体介绍

- 在storm中，可靠的信息处理机制是从spout开始的。
- 一个提供了可靠的处理机制的spout需要记录他发射出去的tuple，当下游bolt处理tuple或者子tuple失败时spout能够重新发射。 
- Storm通过调用Spout的nextTuple()发送一个tuple。为实现可靠的消息处理，首先要给每个发出的tuple带上唯一的ID，并且将ID作为参数传递给SoputOutputCollector的emit()方法：collector.emit(new Values("value1","value2"), msgId); messageid就是用来标示唯一的tuple的，而rootid是随机生成的
- 给每个tuple指定ID告诉Storm系统，无论处理成功还是失败，spout都要接收tuple树上所有节点返回的通知。如果处理成功，spout的ack()方法将会对编号是msgId的消息应答确认；如果处理失败或者超时，会调用fail()方法。 

#### 基本实现

Storm 系统中有一组叫做"acker"的特殊的任务，它们负责跟踪DAG（有向无环图）中的每个消息。

acker任务保存了spout id到一对值的映射。第一个值就是spout的任务id，通过这个id，acker就知道消息处理完成时该通知哪个spout任务。第二个值是一个64bit的数字，我们称之为"ack val"， 它是树中所有消息的随机id的异或计算结果。

ack val表示了整棵树的的状态，无论这棵树多大，只需要这个固定大小的数字就可以跟踪整棵树。当消息被创建和被应答的时候都会有相同的消息id发送过来做异或。 每当acker发现一棵树的ack val值为0的时候，它就知道这棵树已经被完全处理了

![1583838793812](/img/1583838793812.png)

 ![1583838800997](/img/1583838800997.png)

#### 可靠性配置

有三种方法可以去掉消息的可靠性： 

1. 将参数Config.TOPOLOGY_ACKERS设置为0，通过此方法，当Spout发送一个消息的时候，它的ack方法将不会被调用;
2. Spout发送一个消息时，不指定此消息的messageID。当需要关闭特定消息可靠性的时候，可以使用此方法； 
3. 最后，如果你不在意某个消息派生出来的子孙消息的可靠性，则此消息派生出来的子消息在发送时不要做锚定，即在emit方法中不指定输入消息。因为这些子孙消息没有被锚定在任何tuple tree中，因此他们的失败不会引起任何spout重新发送消息。