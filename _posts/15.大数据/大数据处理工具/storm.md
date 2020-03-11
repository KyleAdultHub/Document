---
title: storm
date: "2020-03-10 16:00:00"
categories:
- 大数据
- 大数据处理
tags:
- 大数据
toc: true
typora-root-url: ..\..\..
---

## 流式计算和storm介绍

### 什么是离线计算

离线计算：批量获取数据、批量传输数据、**周期性**批量计算数据、数据展示

代表技术：Sqoop批量导入数据、HDFS批量存储数据、MapReduce批量计算数据、Hive批量计算数据、任务调度

### 什么是流式计算

流式计算：数据实时产生、数据实时传输、数据实时计算、实时展示

代表技术：Flume实时获取数据、Kafka/metaq实时数据存储、Storm/JStorm实时数据计算、Redis实时结果缓存、持久化存储(mysql)。

一句话总结：将源源不断产生的数据实时收集并实时计算，尽可能快的得到计算结果

### 离线计算和实时计算的区别

最大的区别：实时收集、实时计算、实时展示

### 什么是storm

Storm用来实时处理数据，特点：低延迟、高可用、分布式、可扩展、数据不丢失。提供简单容易理解的接口，便于开发。

### Storm 与 Hadoop的区别

- Storm用于实时计算，Hadoop用于离线计算。

- Storm处理的数据保存在内存中，源源不断；Hadoop处理的数据保存在文件系统中，一批一批。

- Storm的数据通过网络传输进来；Hadoop的数据保存在磁盘中。

- Storm与Hadoop的编程模型相似(对比图如下)

![1583828661423](/img/1583828661423.png)

### 应用场景

- **日志分析**
  - 从**海量**日志中分析出特定的数据，并将分析的结果存入外部存储器用来辅佐决策。

- **管道系统**
  - 将一个数据从一个系统传输到另外一个系统，比如将数据库同步到Hadoop

- **消息转化器**
  - 将接受到的消息按照某种格式进行转化，存储到另外一个系统如消息中间件

## Storm 架构

### Storm核心组件

![1583829287411](/img/1583829287411.png)

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

### 流式计算的一般架构

![1583829359441](/img/1583829359441.png)

- 其中flume用来获取数据。

- Kafka用来临时保存数据。

- Strom用来计算数据。

- Redis是个内存数据库，用来保存数据。

## Storm 集群部署与操作

### 集群部署流程

#### 基础环境准备

1. 配置所有机器hosts

   vi /etc/hosts

   ```
   xx.xx.xx.xx storm01 zk01 hadoop01
   xx.xx.xx.xx9 storm02 zk02 hadoop02
   xx.xx.xx.xx storm03 zk03 hadoop03
   ```

2. 关闭防火墙

   ```shell
   chkconfig iptables off  && setenforce 0
   ```

3. 创建storm用户

   ```shell
   groupadd stormuser &&　useradd stormuser && usermod -a -G stormuser stormuser 
   ```

4. 创建工作目录

   ```shell
   mkdir /export
   mkdir /export/servers
   chmod 755 -R /export
   # 切换到realtime用户下
   su stormuser
   ```

#### 安装包下载

1. 下载安装包

   ```shell
   wget http://124.202.164.6/files/1139000006794ECA/apache.fayea.com/storm/apache-storm-0.9.5/apache-storm-0.9.5.tar.gz
   ```

2. 解压安装包

   ```shell
   tar -zxvf apache-storm-0.9.5.tar.gz -C /export/servers/
   cd /export/servers/
   ln -s apache-storm-0.9.5 storm
   ```

#### 配置文件修改与安装包同步

1. 修改配置文件

   ```shell
   mv /export/servers/storm/conf/storm.yaml /export/servers/storm/conf/storm.yaml.bak
   vi /export/servers/storm/conf/storm.yaml
   ```

   输入以下内容：

   ​	指定集群中的机器，nimbus host 机器， JVM的参数配置， 以及supervisor启动worker对应的端口号

   ![1583829969384](/img/1583829969384.png)

2. 分发安装包

   ```shell
   scp -r /export/servers/apache-storm-0.9.5 storm02:/export/servers
   # 然后分别在各机器上创建软连接
   cd /export/servers/
   ln -s apache-storm-0.9.5 storm
   ```

#### 启动集群

1. 在nimbus.host所属的机器上启动 nimbus服务

   ```shell
   cd /export/servers/storm/bin/
   nohup ./storm nimbus &
   ```

2. 在nimbus.host所属的机器上启动ui服务

   ```shell
   cd /export/servers/storm/bin/
   nohup ./storm ui &
   ```

3. 在其它个点击上启动supervisor服务(拓展节点的时候，重复分发安装包，并执行该命令)

   ```shell
   cd /export/servers/storm/bin/
   nohup ./storm supervisor &
   ```

#### 查看集群

​	访问nimbus.host:/8080，即可看到storm的ui界面。

​	![1583829986939](/img/1583829986939.png)

### Storm 的集群命令

有许多简单且有用的命令可以用来管理拓扑，它们可以提交、杀死、禁用、再平衡拓扑。

#### 任务提交命令

命令格式

```shell
storm jar 【jar路径】 【拓扑包名.拓扑类名】 【拓扑名称】
```

命令示例

```shell
bin/storm jar examples/storm-starter/storm-starter-topologies-0.9.6.jar storm.starter.WordCountTopology wordcount
```

#### 杀死任务命令

命令格式

```shell
storm kill 【拓扑名称】 -w 10（执行kill命令时可以通过-w [等待秒数]指定拓扑停用以后的等待时间）
```

命令示例

```shell
storm kill topology-name -w 10
```

#### 停用任务命令

命令格式

```shell
storm deactivte  【拓扑名称】
```

命令示例

```shell
storm deactivte topology-name
```

我们能够挂起或停用运行中的拓扑。当停用拓扑时，所有已分发的元组都会得到处理，但是spouts的nextTuple方法不会被调用。销毁一个拓扑，可以使用kill命令。它会以一种安全的方式销毁一个拓扑，首先停用拓扑，在等待拓扑消息的时间段内允许拓扑完成当前的数据流。

#### 启用任务命令

命令格式

```shell
storm activate【拓扑名称】
```

命令示例

```shell
storm activate topology-name
```

#### 重新部署任务命令

命令格式

```shell
storm rebalance  【拓扑名称】
```

命令示例

```shell
storm rebalance topology-name
```

​    再平衡使你重分配集群任务。这是个很强大的命令。比如，你向一个运行中的集群增加了节点。再平衡命令将会停用拓扑，然后在相应超时时间之后重分配工人，并重启拓扑。

### Storm日志

#### nimbus的日志信息

在nimbus的服务器上

```shell
cd /export/servers/storm/logs
tail -100f /export/servers/storm/logs/nimbus.log
```

#### 查看ui运行日志信息

在ui的服务器上，一般和nimbus一个服务器

```shell
cd /export/servers/storm/logs
tail -100f /export/servers/storm/logs/ui.log
```

#### 查看supervisor运行日志信息

在supervisor服务上

```shell
cd /export/servers/storm/logs
tail -100f /export/servers/storm/logs/supervisor.log
```

#### 查看supervisor上worker运行日志信息

在supervisor服务上

```shell
cd /export/servers/storm/logs
tail -100f /export/servers/storm/logs/worker-6702.log
```

![1583832014008](/img/1583832014008.png)

## Storm流式计算生命周期

### 介绍

storm 执行流式数据处理函数的生命周期主要是通过Spout和Bolt 这两个组件的生命周期

**Spout组件涉及到的方法有：**

- declareOutputFields()   (调用一次)

- open()     (调用一次)

- activate()      (调用一次)

- nextTuple()     （循环调用 ）

- disactive()        (手动调用)

**Bolt组件涉及到的方法有**

- declareOutputFileds()      (调用一次)

- prepare()     (调用一次)

- execute()    （循环执行）

**执行顺序？**

1. 在客户端将jar包提交到集群上的时候，执行 spout 、bolt 的构造方法以及declareOutputFields（）

2. spout执行open方法一次，得到conf一些配置信息

3. activate让处理的信息发送到bolt中
4. Bolt 中 prepare 方法只执行一次

5. Spout 中 nextTuple不断的循环执行

6. Blot 中 execute  不断的循环执行

### Stream Grouping

#### Stream Grouping 的作用

Storm 的Tuple  从 Spout 中 分发到 Bolt， 以及从Bolt 分发的Bolt 的过程中，Storm定义了一些内置的分发方法和规则，

这些规则通过一些条件或者随机将具有相同特征的tuple分发到同一Bolt中进行处理，这样有利于对不同特征的数据进行集中统计

下面介绍一些常用的Grouping 种类

### Stream Grouping 种类

- Shuffle Grouping: 随机分组， 随机派发stream里面的tuple，保证每个bolt接收到的tuple数目大致相同。

- Fields Grouping：按字段分组，比如按userid来分组，具有同样userid的tuple会被分到相同的Bolts里的一个task，而不同的userid则会被分配到不同的bolts里的task。

- All Grouping：广播发送，对于每一个tuple，所有的bolts都会收到。

- Global Grouping：全局分组， 这个tuple被分配到storm中的一个bolt的其中一个task。再具体一点就是分配给id值最低的那个task。
- Non Grouping：不分组，这stream grouping个分组的意思是说stream不关心到底谁会收到它的tuple。目前这种分组和Shuffle grouping是一样的效果，有一点不同的是storm会把这个bolt放到这个bolt的订阅者同一个线程里面去执行， 即减少跨主机socket通讯。

- Direct Grouping： 直接分组， 这是一种比较特别的分组方法，用这种分组意味着消息的发送者指定由消息接收者的哪个task处理这个消息。只有被声明为Direct Stream的消息流可以声明这种分组方法。而且这种消息tuple必须使用emitDirect方法来发射。消息处理者可以通过TopologyContext来获取处理它的消息的task的id （OutputCollector.emit方法也会返回task的id）。

> Local or shuffle grouping：如果目标bolt有一个或者多个task在同一个工作进程中，tuple将会被随机发生给这些tasks。否则，和普通的Shuffle Grouping行为一致。

### 单体统计案例

#### 功能说明

设计一个topology，来实现对文档里面的单词出现的频率进行统计。

整个topology分为三个部分：

- RandomSentenceSpout：数据源，在已知的英文句子中，随机发送一条句子出去。

- SplitSentenceBolt：负责将单行文本记录（句子）切分成单词

- WordCountBolt：负责对单词的频率进行累加

#### 项目结构设计

![1583833270500](/img/1583833270500.png)

#### RandomSentenceSpout的实现

![1583833293835](/img/1583833293835.png)

#### SplitSentenceBolt的实现

![1583833626450](/img/1583833626450.png)

#### WordCountBolt的实现

![1583833649405](/img/1583833649405.png)

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

-  另一个worker执行以上步骤1的操作。

#### Disruptor  队列

**disruptor是什么**

1、 简单理解：Disruptor是一个Queue。Disruptor是实现了“队列”的功能，而且是一个有界队列。而队列的应用场景自然就是“生产者-消费者”模型。

2、 在JDK中Queue有很多实现类，包括不限于ArrayBlockingQueue、LinkBlockingQueue，这两个底层的数据结构分别是数组和链表。数组查询快，链表增删快，能够适应大多数应用场景。

3、 但是ArrayBlockingQueue、LinkBlockingQueue都是线程安全的。涉及到线程安全，就会有synchronized、lock等关键字，这就意味着CPU会打架。 

4、 Disruptor一种线程之间信息无锁的交换方式（使用CAS（Compare And Swap/Set）操作）。

**disruptor主要特点**

Disruptor可以看成一个事件监听或消息机制，在队列中一边生产者放入消息，另外一边消费者并行取出处理.

底层是单个数据结构：一个ring buffer。

每个生产者和消费者都有一个次序计算器，以显示当前缓冲工作方式。

每个生产者消费者能够操作自己的次序计数器的能够读取对方的计数器，生产者能够读取消费者的计算器确保其在没有锁的情况下是可写的。

**核心组件**

- Ring Buffer 环形的缓冲区，负责对通过 Disruptor 进行交换的数据（事件）进行存储和更新。

- Sequence 通过顺序递增的序号来编号管理通过其进行交换的数据（事件），对数据(事件)的处理过程总是沿着序号逐个递增处理。

- RingBuffer底层是个数组，次序计算器是一个64bit long 整数型，平滑增长。

![1583836641903](/img/1583836641903.png)

​	1. 接受数据并写入到脚标31的位置，之后会沿着序号一直写入，但是不会绕过消费者所在的脚标。

​	2. Joumaler和replicator同时读到24的位置，他们可以批量读取数据到30

​	3. 消费逻辑线程读到了14的位置，但是没法继续读下去，因为他的sequence暂停在15的位置上，需要等到他的sequence给他序号。如果sequence能正常工作，就能读取到30的数据。

#### Worker间的网络通讯

**Netty**

Netty是一个NIO client-server(客户端服务器)框架，使用Netty可以快速开发网络应用，例如服务器和客户端协议。Netty提供了一种新的方式来使开发网络应用程序，这种新的方式使得它很容易使用和有很强的扩展性。Netty的内部实现时很复杂的，但是Netty提供了简单易用的api从网络处理代码中解耦业务逻辑。Netty是完全基于NIO实现的，所以整个Netty都是异步的。

**ZeroMQ**

ZeroMQ是一种基于消息队列的多线程网络库，其对套接字类型、连接处理、帧、甚至路由的底层细节进行抽象，提供跨越多种传输协议的套接字。ZeroMQ是网络通信中新的一层，介于应用层和传输层之间（按照TCP/IP划分），其是一个可伸缩层，可并行运行，分散在分布式系统间。

ZeroMQ定位为：一个简单好用的传输层，像框架一样的一个socket library，他使得Socket编程更加简单、简洁和性能更高。是一个消息处理队列库，可在多个线程、内核和主机盒之间弹性伸缩。ZMQ的明确目标是“成为标准网络协议栈的一部分，之后进入Linux内核”。

### Storm 组件本地目录结构

![1583836893868](/img/1583836893868.png)

### Storm zookeeper 目录树

![1583837267075](/img/1583837267075.png)

### Storm 任务提交过程

**过程示意图**

![1583837643662](/img/1583837643662.png)

**nimbus 创建的Topo对象示意图**

![1583837698684](/img/1583837698684.png)

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