---
title: Kafka 面试题整理
date: "2020-06-13 11:00:00"
categories:
- 总结
- 面试总结
tags:
- 面试
toc: true
typora-root-url: ..\..\..
---

## Kafka架构

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

### kafka 相对传统技术的优势

- 快速:单一的Kafka代理可以处理成千上万的客户端，每秒处理数兆字节的读写操作。
- 可伸缩:在一组机器上对数据进行分区和简化，以支持更大的数据
- 持久:消息是持久性的，并在集群中进行复制，以防止数据丢失。
- 设计:它提供了容错保证和持久性

### Kafka 使用zookeeper 的作用

- Zookeeper主要用于在集群中不同节点之间进行通信
- 在Kafka中，它被用于提交偏移量，因此如果节点在任何情况下都失败了，它都可以从之前提交的偏移量中获取
- 除此之外，它还执行其他活动，如: leader检测、分布式同步、配置管理、识别新节点何时离开或连接、集群、节点实时状态等等。

### Kafka 判断一个节点是否还活着有那两个条件？

（1）节点必须可以维护和 ZooKeeper 的连接，Zookeeper 通过心跳机制检查每个节点的连接

（2）如果节点是个 follower,他必须能及时的同步 leader 的写操作，延时不能太久

## Kafka分区

### Kafka生产者分区机制

**轮询策略**

也称 Round-robin 策略，即顺序分配。比如一个主题下有 3 个分区，那么第一条消息被发送到分区 0，第二条被发送到分区 1，第三条被发送到分区 2，以此类推。当生产第 4 条消息时又会重新开始，即将其分配到分区 0。

**随机策略**

也称 Randomness 策略。所谓随机就是我们随意地将消息放置到任意一个分区上，如下面这张图所示。

**按消息key保存策略**

也称 Key-ordering 策略。这个可以理解为是自定义的策略之一。

Kafka 允许为每条消息定义消息键，简称为 Key。这个 Key 的作用非常大，它可以是一个有着明确业务含义的字符串，比如客户代码、部门编号或是业务 ID 等；也可以用来表征消息元数据。特别是在 Kafka 不支持时间戳的年代，在一些场景中，工程师们都是直接将消息创建时间封装进 Key 里面的。一旦消息被定义了 Key，那么你就可以保证同一个 Key 的所有消息都进入到相同的分区里面，由于每个分区下的消息处理都是有顺序的，故这个策略被称为按消息键保序策略

## Kafka 生产者

### producer 是否直接将数据发送到 broker 的 leader(主节点)？

producer 直接将数据发送到 broker 的 leader(主节点)，不需要在多个节点进行分发。

为了帮助 producer 做到这点，所有的 Kafka 节点都可以及时的告知:哪些节点是活动的，目标topic 目标分区的 leader 在哪。这样 producer 就可以直接将消息发送到目的地了

## Kafka 消费者

### Kafa consumer 是否可以消费指定分区消息？

Kafa consumer 消费消息时，向 broker 发出"fetch"请求去消费特定分区的消息，consumer指定消息在日志中的偏移量（offset），就可以消费从这个位置开始的消息，customer 拥有了 offset 的控制权，可以向后回滚去重新消费之前的消息，这是很有意义的

### Kafka 消息是采用 Pull 模式，还是 Push 模式？

Kafka 最初考虑的问题是，customer 应该从 brokes 拉取消息还是 brokers 将消息推送到consumer，也就是 pull 还 push。在这方面，Kafka 遵循了一种大部分消息系统共同的传统的设计：producer 将消息推送到 broker，consumer 从 broker 拉取消息

## Kafka 一致性

### kafka 怎么确保数据不丢失, ack 机制

设置发送数据是否需要服务端的反馈,有三个值0,1,-1

- 0: producer不会等待broker发送ack 
- 1: 当leader接收到消息之后发送ack 
- -1: 当所有的follower都同步消息成功后发送ack

```INI
request.required.acks=0
```

### Kafka如何保证消费数据的一致性？

LEO：指每个follower的最大的offset

HW（高水位）：指消费者能见到的最大的offset，ISR队列中最小的LEO，也就是说消费者只能看到1~6的数据，后面的数据看不到，也消费不了 

避免leader挂掉后，比如当前消费者消费8这条数据后，leader挂 了，此时比如f2成为leader，f2根本就没有9这条数据，那么消费者就会报错，所以设计了HW这个参数，只暴露最少的数据给消费者，避免上面的问题

### kafka 怎么确保producer数据不丢失, ack 机制

request.required.acks 有三个值 0 1 -1

0:生产者不会等待 broker 的 ack，这个延迟最低但是存储的保证最弱当 server 挂掉的时候

就会丢数据

1：服务端会等待 ack 值 leader 副本确认接收到消息后发送 ack 但是如果 leader 挂掉后他

不确保是否复制完成新 leader 也会导致数据丢失

-1：同样在 1 的基础上 服务端会等所有的 follower 的副本受到数据后才会受到 leader 发出

的 ack，这样数据不会丢失

### KAFKA 确保数据一致性的详细解释 ISR

这里介绍的数据一致性主要是说不论是老的 Leader 还是新选举的 Leader，Consumer 都能读到一样的数据。那么 Kafka 是如何实现的呢？

![1594010991499](/img/1594010991499.png)

假设分区的副本为3，其中副本0是 Leader，副本1和副本2是 follower，并且在 ISR 列表里面。虽然副本0已经写入了 Message4，但是 Consumer 只能读取到 Message2。因为所有的 ISR 都同步了 Message2，只有 High Water Mark 以上的消息才支持 Consumer 读取，而 High Water Mark 取决于 ISR 列表里面偏移量最小的分区，对应于上图的副本2，这个很类似于木桶原理。

这样做的原因是还没有被足够多副本复制的消息被认为是“不安全”的，如果 Leader 发生崩溃，另一个副本成为新 Leader，那么这些消息很可能丢失了。如果我们允许消费者读取这些消息，可能就会破坏一致性。试想，一个消费者从当前 Leader（副本0） 读取并处理了 Message4，这个时候 Leader 挂掉了，选举了副本1为新的 Leader，这时候另一个消费者再去从新的 Leader 读取消息，发现这个消息其实并不存在，这就导致了数据不一致性问题。

当然，引入了 High Water Mark 机制，会导致 Broker 间的消息复制因为某些原因变慢，那么消息到达消费者的时间也会随之变长（因为我们会先等待消息复制完毕）。延迟时间可以通过参数 `replica.lag.time.max.ms` 参数配置，它指定了副本在复制消息时可被允许的最大延迟时间。

## Kafka 文件系统

### Kafka 零复制技术

正常情况下，先把数据读到内核空间，在从内核空间把数据读到用户空间，然后在调操作系统的io接口写到内核空间，最终在写到硬盘中

Kafka是这样做的，直接在内核空间流转io流，所以kafka的性能非常高

kafka利用了Linux的sendFile技术（NIO），省去了进程切换和一次数据拷贝，让性能变得更好。

### Kafka 文件存储机制

**Kafka 文件存储基本结构**

- 在Kafka文件存储中，同一个topic下有多个不同partition，每个partition为一个目录，partiton命名规则为topic名称+有序序号，第一个partiton序号从0开始，序号最大值为partitions数量减1。
- 每个partion(目录)相当于一个巨型文件被平均分配到多个大小相等segment(段)数据文件中。但每个段segment file消息数量不一定相等，这种特性方便old segment file快速被删除。默认保留7天的数据。

![1592037825676](/img/1592037825676.png)

- 每个partiton只需要支持顺序读写就行了，segment文件生命周期由服务端配置参数决定。（什么时候创建，什么时候删除）

![1592037834703](/img/1592037834703.png)

**数据有序的讨论？**

​	一个partition的数据是否是有序的？	间隔性有序，不连续

​	针对一个topic里面的数据，只能做到partition内部有序，不能做到全局有序。

​	特别加入消费者的场景后，如何保证消费者消费的数据全局有序的？伪命题。

​	只有一种情况下才能保证全局有序？就是只有一个partition。

**Kafka Partition Segment**

- Segment file组成：由2大部分组成，分别为index file和data file，此2个文件一 一对应，成对出现，后缀".index"和“.log”分别表示为segment索引文件、数据文件。

![1592037845195](/img/1592037845195.png)

- Segment文件命名规则：partion全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值。数值最大为64位long大小，19位数字字符长度，没有数字用0填充。
- 索引文件存储大量元数据，数据文件存储大量消息，索引文件中元数据指向对应数据文件中message的物理偏移地址。

![1592037856146](/img/1592037856146.png)

> ​	3，497：当前log文件中的第几条信息，存放在磁盘上的那个地方
>
> ​	上述图中索引文件存储大量元数据，数据文件存储大量消息，索引文件中元数据指向对应数据文件中message的物理偏移地址。
>
> ​	其中以索引文件中元数据3,497为例，依次在数据文件中表示第3个message(在全局partiton表示第368772个message)
>
> ​	以及该消息的物理偏移地址为497。

- segment data file由许多message组成， 物理结构如下：

![1592037918624](/img/1592037918624.png)

### Kafka 查找Message

读取offset=368776的message，需要通过下面2个步骤查找。

![1592037926311](/img/1592037926311.png)

**1.查找segment file**

00000000000000000000.index表示最开始的文件，起始偏移量(offset)为0

00000000000000368769.index的消息量起始偏移量为368770 = 368769 + 1

00000000000000737337.index的起始偏移量为737338=737337 + 1

其他后续文件依次类推。

以起始偏移量命名并排序这些文件，只要根据offset 二分查找文件列表，就可以快速定位到具体文件。当offset=368776时定位到00000000000000368769.index和对应log文件。

**2.通过segment file 查找 message**

当offset=368776时，依次定位到00000000000000368769.index的元数据物理位置和00000000000000368769.log的物理偏移地址

然后再通过00000000000000368769.log顺序查找直到offset=368776为止。













