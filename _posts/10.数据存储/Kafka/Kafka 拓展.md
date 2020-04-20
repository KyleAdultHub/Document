---
title: Kafka 拓展
date: "2020-03-15 14:00:00"
categories:
- 数据存储
- Kafka
tags:
- Kafka
- 数据存储
toc: true
typora-root-url: ..\..\..\
---

## Kafka 介绍

### 什么是Kafka

在流式计算中，Kafka一般用来缓存数据，Storm通过消费Kafka的数据进行计算。

KAFKA + STORM +REDIS

- Apache Kafka是一个开源消息系统，由Scala写成。是由Apache软件基金会开发的一个开源消息系统项目。

- Kafka最初是由LinkedIn开发，并于2011年初开源。2012年10月从Apache Incubator毕业。该项目的目标是为处理实时数据提供一个统一、高通量、低等待的平台。

- Kafka是一个分布式消息队列：生产者、消费者的功能。它提供了类似于JMS的特性，但是在设计实现上完全不同，此外它并不是JMS规范的实现。

- Kafka对消息保存时根据Topic进行归类，发送消息者称为Producer,消息接受者称为Consumer,此外kafka集群有多个kafka实例组成，每个实例(server)成为broker。

- 无论是kafka集群，还是producer和consumer都依赖于**zookeeper**集群保存一些meta信息，来保证系统可用性

### 什么是JMS

JMS是什么：JMS是Java提供的一套技术规范

JMS干什么用：用来异构系统 集成通信，缓解系统瓶颈，提高系统的伸缩性增强系统用户体验，使得系统模块化和组件化变得可行并更加灵活

通过什么方式：生产消费者模式（生产者、服务器、消费者）

 ![1584253000929](/img/1584253000929.png)

- 点对点模式（一对一，消费者主动拉取数据，消息收到后消息清除）

- 点对点模型通常是一个基于拉取或者轮询的消息传送模型，这种模型从队列中请求信息，而不是将消息推送到客户端。这个模型的特点是发送到队列的消息*一个且只有一个接收者接收处理，即使有多个消息监听者也是如此。

- 发布/订阅模式（一对多，数据生产后，推送给所有订阅者）

- 发布订阅模型则是一个基于推送的消息传送模型。发布订阅模型可以有多种不同的订阅者，临时订阅者只在主动监听主题时才接收消息，而持久订阅者则监听主题的所有消息，即时当前订阅者不可用，处于离线状态。

  ```
  queue.put（object）  数据生产
  queue.take(object)    数据消费
  ```


![1584253027931](/img/1584253027931.png)

### JMS 核心组件

- Destination：消息发送的目的地，也就是前面说的Queue和Topic。

- Message ：从字面上就可以看出是被发送的消息。

- Producer： 消息的生产者，要发送一个消息，必须通过这个生产者来发送。

- MessageConsumer： 与生产者相对应，这是消息的消费者或接收者，通过它来接收一个消息。

![1584253111203](/img/1584253111203.png)

### 为什么需要JMS组件（消息队列）

消息系统的核心作用就是三点：解耦，异步和并行

以用户注册的案列来说明消息系统的作用

### Kafka 核心组件

- Topic ：消息根据Topic进行归类

- Producer：发送消息者

- Consumer：消息接受者

- broker：每个kafka实例(server)

- Zookeeper：依赖集群保存meta信息。

![1584253241846](/img/1584253241846.png)

## Kafka 部署步骤

1. **下载安装包**

   <http://kafka.apache.org/downloads.html>

   在linux中使用wget命令下载安装包

    wget http://mirrors.hust.edu.cn/apache/kafka/0.8.2.2/kafka_2.11-0.8.2.2.tgz

2. **解压安装包**

   tar -zxvf /export/software/kafka_2.11-0.8.2.2.tgz -C /export/servers/

   cd /export/servers/

   ln -s kafka_2.11-0.8.2.2 kafka

3. **修改配置文件**

   cp   /export/servers/kafka/config/server.properties

   /export/servers/kafka/config/server.properties.bak

   vi  /export/servers/kafka/config/server.properties

   输入以下内容

   ![1584253538452](/img/1584253538452.png)

   4. **分发安装包**

      scp -r /export/servers/kafka_2.11-0.8.2.2 kafka02:/export/servers

      然后分别在各机器上创建软连

      cd /export/servers/

      ln -s kafka_2.11-0.8.2.2 kafka

   5. **再次修改配置文件（重要）**

      依次修改各服务器上配置文件的的broker.id，分别是0,1,2不得重复。

   6. **启动集群**

      依次在各节点上启动kafka

      bin/kafka-server-start.sh  config/server.properties

## kafka集群命令

- 查看当前服务器中的所有topic

  ```shell
  bin/kafka-topics.sh --list --zookeeper  zk01:2181
  ```

- 创建topic

  ```shell
  ./kafka-topics.sh --create --zookeeper mini1:2181 --replication-factor 1 --partitions 3 --topic first
  ```

- 删除topic

  ```shell
  sh bin/kafka-topics.sh --delete --zookeeper zk01:2181 --topic test
  ```

  需要server.properties中设置delete.topic.enable=true否则只是标记删除或者直接重启。

- 通过shell命令发送消息

  ```shell
  kafka-console-producer.sh --broker-list kafka01:9092 --topic itheima
  ```

- 通过shell消费消息

  ```shell
  sh bin/kafka-console-consumer.sh --zookeeper zk01:2181 --from-beginning --topic test1
  ```

- 查看消费位置

  ```shell
  sh kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --zookeeper zk01:2181 --group testGroup
  ```

- 查看某个Topic的详情

  ```shell
  sh kafka-topics.sh --topic test --describe --zookeeper zk01:2181
  ```

 ## Kafka java api

### 生产者相关api

![1584253713384](/img/1584253713384.png)

### 消费者相关api

![1584253728155](/img/1584253728155.png)

## Kafka 架构介绍

### Kafka 整体结构介绍

- Producer ：消息生产者，就是向kafka broker发消息的客户端。

- Consumer ：消息消费者，向kafka broker取消息的客户端

- Topic ：咋们可以理解为一个队列。

- Consumer Group （CG）：这是kafka用来实现一个topic消息的广播（发给所有的consumer）和单播（发给任意一个consumer）的手段。一个topic可以有多个CG。topic的消息会复制（不是真的复制，是概念上的）到所有的CG，但每个partion只会把消息发给该CG中的一个consumer。如果需要实现广播，只要每个consumer有一个独立的CG就可以了。要实现单播只要所有的consumer在同一个CG。用CG还可以将consumer进行自由的分组而不需要多次发送消息到不同的topic。

- Broker ：一台kafka服务器就是一个broker。一个集群由多个broker组成。一个broker可以容纳多个topic。

- Partition：为了实现扩展性，一个非常大的topic可以分布到多个broker（即服务器）上，一个topic可以分为多个partition，每个partition是一个有序的队列。partition中的每条消息都会被分配一个有序的id（offset）。kafka只保证按一个partition中的顺序将消息发给consumer，不保证一个topic的整体（多个partition间）的顺序。

- Offset：kafka的存储文件都是按照offset.kafka来命名，用offset做名字的好处是方便查找。例如你想找位于2049的位置，只要找到2048.kafka的文件即可。当然the first offset就是00000000000.kafka

### Consumer 与topic 的关系

本质上kafka只支持Topic；

- 每个group中可以有多个consumer，每个consumer属于一个consumer group；

  通常情况下，一个group中会包含多个consumer，这样不仅可以提高topic中消息的并发消费能力，而且还能提高"故障容错"性，如果group中的某个consumer失效那么其消费的partitions将会有其他consumer自动接管。

- 对于Topic中的一条特定的消息，只会被订阅此Topic的每个group中的其中一个consumer消费，此消息不会发送给一个group的多个consumer；

  那么一个group中所有的consumer将会交错的消费整个Topic，每个group中consumer消息消费互相独立，我们可以认为一个group是一个"订阅"者。

- 在kafka中,一个partition中的消息只会被group中的一个consumer消费**(同一时刻)**；

  一个Topic中的每个partions，只会被一个"订阅者"中的一个consumer消费，不过一个consumer可以同时消费多个partitions中的消息。

- kafka的设计原理决定,对于一个topic，同一个group中不能有多于partitions个数的consumer同时消费，否则将意味着某些consumer将无法得到消息。

  kafka只能保证一个partition中的消息被某个consumer消费时是顺序的；事实上，从Topic角度来说,当有多个partitions时,消息仍不是全局有序的。

### Kafka 消息分发

**Producer客户端负责消息的分发**

- kafka集群中的任何一个broker都可以向producer提供metadata信息,这些metadata中包含"集群中存活的servers列表"/"partitions leader列表"等信息；

- 当producer获取到metadata信息之后, producer将会和Topic下所有partition leader保持socket连接；

- 消息由producer直接通过socket发送到broker，中间不会经过任何"路由层"，事实上，消息被路由到哪个partition上由producer客户端决定；

  比如可以采用"random""key-hash""轮询"等,**如果一个topic中有多个partitions,那么在producer端实现"消息均衡分发"是必要的。**

- 在producer端的配置文件中,开发者可以指定partition路由的方式。

**Producer消息发送的应答机制**

设置发送数据是否需要服务端的反馈,有三个值0,1,-1

- 0: producer不会等待broker发送ack 

- 1: 当leader接收到消息之后发送ack 

- -1: 当所有的follower都同步消息成功后发送ack

```INI
request.required.acks=0
```

### Kafka 消费负载均衡

当一个group中,有consumer加入或者离开时,会触发partitions均衡.均衡的最终目的,是提升topic的并发消费能力，步骤如下：

1、 假如topic1,具有如下partitions: P0,P1,P2,P3

2、 加入group中,有如下consumer: C1,C2

3、 首先根据partition索引号对partitions排序: P0,P1,P2,P3

4、 根据consumer.id排序: C0,C1

5、 计算倍数: M = [P0,P1,P2,P3].size / [C0,C1].size,本例值M=2(向上取整)

6、 然后依次分配partitions: C0 = [P0,P1],C1=[P2,P3],即Ci = [P(i * M),P((i + 1) * M -1)]

![1584254099386](/img/1584254099386.png)

## Kafka生产者分区机制

所谓分区策略是决定生产者将消息发送到哪个分区的算法。Kafka 为我们提供了默认的分区策略，同时它也支持你自定义分区策略。

如果要自定义分区策略，你需要显式地配置生产者端的参数partitioner.class。这个参数该怎么设定呢？方法很简单，在编写生产者程序时，你可以编写一个具体的类实现org.apache.kafka.clients.producer.Partitioner接口。这个接口也很简单，只定义了两个方法：partition()和close()，通常你只需要实现最重要的 partition 方法。我们来看看这个方法的方法签名：

```java
int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster);
```

这里的topic、key、keyBytes、value和valueBytes都属于消息数据，cluster则是集群信息（比如当前 Kafka 集群共有多少主题、多少 Broker 等）。Kafka 给你这么多信息，就是希望让你能够充分地利用这些信息对消息进行分区，计算出它要被发送到哪个分区中。只要你自己的实现类定义好了 partition 方法，同时设置partitioner.class参数为你自己实现类的 Full Qualified Name，那么生产者程序就会按照你的代码逻辑对消息进行分区。虽说可以有无数种分区的可能，但比较常见的分区策略也就那么几种，下面我来详细介绍一下。

### 轮询策略
也称 Round-robin 策略，即顺序分配。比如一个主题下有 3 个分区，那么第一条消息被发送到分区 0，第二条被发送到分区 1，第三条被发送到分区 2，以此类推。当生产第 4 条消息时又会重新开始，即将其分配到分区 0，就像下面这张图展示的那样。

![1586945275048](/img/1586945275048.png)

这就是所谓的轮询策略。轮询策略是 Kafka Java 生产者 API 默认提供的分区策略。如果你未指定partitioner.class参数，那么你的生产者程序会按照轮询的方式在主题的所有分区间均匀地“码放”消息。

轮询策略有非常优秀的负载均衡表现，它总是能保证消息最大限度地被平均分配到所有分区上，故默认情况下它是最合理的分区策略，也是我们最常用的分区策略之一。

### 随机策略

也称 Randomness 策略。所谓随机就是我们随意地将消息放置到任意一个分区上，如下面这张图所示。

![1586945298056](/img/1586945298056.png)

如果要实现随机策略版的 partition 方法，很简单，只需要两行代码即可：

```java
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return ThreadLocalRandom.current().nextInt(partitions.size());
```

先计算出该主题总的分区数，然后随机地返回一个小于它的正整数。

本质上看随机策略也是力求将数据均匀地打散到各个分区，但从实际表现来看，它要逊于轮询策略，所以如果追求数据的均匀分布，还是使用轮询策略比较好。事实上，随机策略是老版本生产者使用的分区策略，在新版本中已经改为轮询了。

### 按消息key保存策略

也称 Key-ordering 策略。这个可以理解为是自定义的策略之一。

Kafka 允许为每条消息定义消息键，简称为 Key。这个 Key 的作用非常大，它可以是一个有着明确业务含义的字符串，比如客户代码、部门编号或是业务 ID 等；也可以用来表征消息元数据。特别是在 Kafka 不支持时间戳的年代，在一些场景中，工程师们都是直接将消息创建时间封装进 Key 里面的。一旦消息被定义了 Key，那么你就可以保证同一个 Key 的所有消息都进入到相同的分区里面，由于每个分区下的消息处理都是有顺序的，故这个策略被称为按消息键保序策略，如下图所示。

![1586945356746](/img/1586945356746.png)

实现这个策略的 partition 方法同样简单，只需要下面两行代码即可：

```java
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return Math.abs(key.hashCode()) % partitions.size();
```

前面提到的 Kafka 默认分区策略实际上同时实现了两种策略：如果指定了 Key，那么默认实现按消息Key保序策略；如果没有指定 Key，则使用轮询策略。

### 其他分区策略
上面这几种分区策略都是比较基础的策略，除此之外你还能想到哪些有实际用途的分区策略？其实还有一种比较常见的，即所谓的基于地理位置的分区策略。当然这种策略一般只针对那些大规模的 Kafka 集群，特别是跨城市、跨国家甚至是跨大洲的集群。

假设有个厂商所有服务都部署在北京的一个机房（这里我假设它是自建机房，不考虑公有云方案。其实即使是公有云，实现逻辑也差不多），现在该厂商考虑在南方找个城市（比如深圳）再创建一个机房；另外从两个机房中选取一部分机器共同组成一个大的 Kafka 集群。显然，这个集群中必然有一部分机器在北京，另外一部分机器在深圳。

假设该厂商计划为每个新注册用户提供一份注册礼品，比如南方的用户注册的可以免费得到一碗“甜豆腐脑”，而北方的新注册用户可以得到一碗“咸豆腐脑”。如果用 Kafka 来实现则很简单，只需要创建一个双分区的主题，然后再创建两个消费者程序分别处理南北方注册用户逻辑即可。

但问题是你需要把南北方注册用户的注册消息正确地发送到位于南北方的不同机房中，因为处理这些消息的消费者程序只可能在某一个机房中启动着。换句话说，送甜豆腐脑的消费者程序只在深圳机房启动着，而送咸豆腐脑的程序只在北京的机房中，如果你向深圳机房中的 Broker 发送北方注册用户的消息，那么这个用户将无法得到礼品！

此时我们就可以根据 Broker 所在的 IP 地址实现定制化的分区策略。比如下面这段代码：

```java
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return partitions.stream().filter(p -> isSouth(p.leader().host())).map(PartitionInfo::partition).findAny().get()
```

我们可以从所有分区中找出那些 Leader 副本在南方的所有分区，然后随机挑选一个进行消息发送。

## Kafka 文件存储机制

### Kafka 文件存储基本结构

- 在Kafka文件存储中，同一个topic下有多个不同partition，每个partition为一个目录，partiton命名规则为topic名称+有序序号，第一个partiton序号从0开始，序号最大值为partitions数量减1。

- 每个partion(目录)相当于一个巨型文件被平均分配到多个大小相等segment(段)数据文件中。但每个段segment file消息数量不一定相等，这种特性方便old segment file快速被删除。默认保留7天的数据。

![1584254146347](/img/1584254146347.png)

- 每个partiton只需要支持顺序读写就行了，segment文件生命周期由服务端配置参数决定。（什么时候创建，什么时候删除）

![1584254164761](/img/1584254164761.png)

**数据有序的讨论？**

​	一个partition的数据是否是有序的？	间隔性有序，不连续

​	针对一个topic里面的数据，只能做到partition内部有序，不能做到全局有序。

​	特别加入消费者的场景后，如何保证消费者消费的数据全局有序的？伪命题。

​	只有一种情况下才能保证全局有序？就是只有一个partition。

### Kafka Partition Segment

- Segment file组成：由2大部分组成，分别为index file和data file，此2个文件一 一对应，成对出现，后缀".index"和“.log”分别表示为segment索引文件、数据文件。

![1584254195839](/img/1584254195839.png)

- Segment文件命名规则：partion全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值。数值最大为64位long大小，19位数字字符长度，没有数字用0填充。

- 索引文件存储大量元数据，数据文件存储大量消息，索引文件中元数据指向对应数据文件中message的物理偏移地址。

![1584254238529](/img/1584254238529.png)

​																3，497：当前log文件中的第几条信息，存放在磁盘上的那个地方

上述图中索引文件存储大量元数据，数据文件存储大量消息，索引文件中元数据指向对应数据文件中message的物理偏移地址。

其中以索引文件中元数据3,497为例，依次在数据文件中表示第3个message(在全局partiton表示第368772个message)、以及该消息的物理偏移地址为497。

- segment data file由许多message组成， 物理结构如下：

![1584254294384](/img/1584254294384.png)

### Kafka 查找Message

读取offset=368776的message，需要通过下面2个步骤查找。

![1584254319949](/img/1584254319949.png)

### 查找segment file

00000000000000000000.index表示最开始的文件，起始偏移量(offset)为0

00000000000000368769.index的消息量起始偏移量为368770 = 368769 + 1

00000000000000737337.index的起始偏移量为737338=737337 + 1

其他后续文件依次类推。

以起始偏移量命名并排序这些文件，只要根据offset 二分查找文件列表，就可以快速定位到具体文件。当offset=368776时定位到00000000000000368769.index和对应log文件。

### 通过segment file 查找 message

当offset=368776时，依次定位到00000000000000368769.index的元数据物理位置和00000000000000368769.log的物理偏移地址

然后再通过00000000000000368769.log顺序查找直到offset=368776为止。