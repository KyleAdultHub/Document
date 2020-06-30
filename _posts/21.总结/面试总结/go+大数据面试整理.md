---
title: go + 大数据面试整理
date: "2020-06-13 11:00:00"
categories:
- 总结
- 面试总结
tags:
- 面试
toc: true
typora-root-url: ..\..\..
---

## Golang

### 1. golang中常量是怎么实现的

从汇编中看是对字符串常量加了一个标号，同时设置为SRODATA，也就是只读，对数字常量直接在代码中作为立即数使用了

### 2. golang的make和new的区别是什么

 new有点像c++里面的new,用来初始化各种type，然后返回其指针。 只不过由于没有构造函数的存在，所以全部用零值来填充，比较特殊的是slice,map,channel， 它们的零值都是nil。另外由于golang直接可以用&struct{} 形式来初始化，所以平时用到new的机会也比较少。
make是用来初始化map,slice,以及channel的，并且可以指定初始长度， 它返回的不是指针，而是对象本身。另外，make出来的map,slice,channel都是可以直接使用的。

### 3. golang 的channel是怎么实现的

 golang的channel是个结构体，里面大概包含了三大部分：
 a. 指向内容的环形缓存区，及其相关游标
 b. 读取和写入的排队goroutine链表
 c. 锁
 任何操作前都需要获得锁， 当写满或者读空的时候，就将当前goroutine加入到recvq或者sendq中， 并出让cpu(gopark)。

### 4. golang 的GC算法

golang  采用的是三色标记法

**标记-清除算法：**

对象只有黑白两色

1. stop the world,即停止所有goroutine
2. 从根对象（全局指针和栈上的对象）出发，把所有能直接或间接访问到的对象标记为黑色，其它所有对象标志为白色
3. 清除所有白色对象
4. start the world

**三色标记法：**

对象有黑白灰三色

1. stop the world
2. 将根对象全部标记为灰色
3. start the world
4. 在goroutine中进行对灰色对象进行遍历， 将灰色对象引用的每个对象标记为灰色，然后将该灰色对象标记为黑色。
5. 重复执行4， 直接将所有灰色对象都变成黑色对象。
6. stop the world，清除所有白色对象

这里4，5是与用户程序是并发执行的，所以stw的时间被大大缩短了。 不过这样做可能会导致新创建的对象被误清除，因此使用了写屏障技术来解决该问题，大体逻辑是当创建新对象时将新对象置为灰色。

### 5.Golang中除了加Mutex锁以外还有哪些方式安全读写共享变量？

Golang中Goroutine 可以通过 Channel 进行安全读写共享变量。

### 6.go语言的并发机制以及它所使用的CSP并发模型．

Golang的CSP并发模型，是通过Goroutine和Channel来实现的。

Goroutine 是Go语言中并发的执行单位。有点抽象，其实就是和传统概念上的”线程“类似，可以理解为”线程“。 Channel是Go语言中各个并发结构体(Goroutine)之前的通信机制。通常Channel，是各个Goroutine之间通信的”管道“，有点类似于Linux中的管道。

通信机制channel也很方便，传数据用channel <- data，取数据用<-channel。

在通信过程中，传数据channel <- data和取数据<-channel必然会成对出现，因为这边传，那边取，两个goroutine之间才会实现通信。

而且不管传还是取，必阻塞，直到另外的goroutine传或者取为止。

### 7. Golang 中常用的并发模型？

- 通过channel通知实现并发控制
- 通过sync包中的WaitGroup实现并发控制
- 在Go 1.7 以后引进的强大的Context上下文，实现并发控制

### 8.协程，线程，进程的区别。

- 进程

进程是具有一定独立功能的程序关于某个数据集合上的一次运行活动,进程是系统进行资源分配和调度的一个独立单位。每个进程都有自己的独立内存空间，不同进程通过进程间通信来通信。由于进程比较重量，占据独立的内存，所以上下文进程间的切换开销（栈、寄存器、虚拟内存、文件句柄等）比较大，但相对比较稳定安全。

- 线程

线程是进程的一个实体,是CPU调度和分派的基本单位,它是比进程更小的能独立运行的基本单位.线程自己基本上不拥有系统资源,只拥有一点在运行中必不可少的资源(如程序计数器,一组寄存器和栈),但是它可与同属一个进程的其他的线程共享进程所拥有的全部资源。线程间通信主要通过共享内存，上下文切换很快，资源开销较少，但相比进程不够稳定容易丢失数据。

- 协程

协程是一种用户态的轻量级线程，协程的调度完全由用户控制。协程拥有自己的寄存器上下文和栈。协程调度切换时，将寄存器上下文和栈保存到其他地方，在切回来的时候，恢复先前保存的寄存器上下文和栈，直接操作栈则基本没有内核切换的开销，可以不加锁的访问全局变量，所以上下文的切换非常快。

### 10. 什么是channel，为什么它可以做到线程安全？

Channel是Go中的一个核心类型，可以把它看成一个管道，通过它并发核心单元就可以发送或者接收数据进行通讯(communication),Channel也可以理解是一个先进先出的队列，通过管道进行通信。

Golang的Channel,发送一个数据到Channel 和 从Channel接收一个数据 都是 原子性的。而且Go的设计思想就是:不要通过共享内存来通信，而是通过通信来共享内存，前者就是传统的加锁，后者就是Channel。也就是说，设计Channel的主要目的就是在多任务间传递数据的，这当然是安全的。

### 11.Goroutine

#### goroutine作用

goroutine的本质是协程，是实现并行计算的核心，和python的async编程类似，但是又不需要我们手动的去创建携程并通过实践循环来启动。goroutine异步执行函数，只需使用go关键字+函数名即可启动一个协程，函数则直接处于异步执行的状态。

```go
go func()//通过go关键字启动一个协程来运行函数
```

#### goroutine的原理

**携程概念介绍**

协程拥有自己的寄存器上下文和栈。协程调度切换时，将寄存器上下文和栈保存到其他地方，在切回来的时候，恢复先前保存的寄存器上下文和栈。因此，协程能保留上一次调用时的状态（即所有局部状态的一个特定组合），每次过程重入时，就相当于进入上一次调用的状态，换种说法：进入上一次离开时所处逻辑流的位置。线程和进程的操作是由程序触发系统接口，最后的执行者是系统；协程的操作执行者则是用户自身程序，goroutine也是协程。

**调度模型介**

groutine是通过GPM调度模型实现，下面就来解释下goroutine的调度模型。

Go的调度器内部有四个重要的结构：M，P，S，Sched，如上图所示（Sched未给出）
M:M代表内核级线程，一个M就是一个线程，goroutine就是跑在M之上的；M是一个很大的结构，里面维护小对象内存cache（mcache）、当前执行的goroutine、随机数发生器等等非常多的信息
G:代表一个goroutine，它有自己的栈，instruction pointer和其他信息（正在等待的channel等等），用于调度。
P:P全称是Processor，处理器，它的主要用途就是用来执行goroutine的，所以它也维护了一个goroutine队列，里面存储了所有需要它来执行的goroutine
Sched：代表调度器，它维护有存储M和G的队列以及调度器的一些状态信息等。

### 12. Golang syncmap实现原理

- 通过 read 和 dirty 两个字段将读写分离，读的数据存在只读字段 read 上，将最新写入的数据则存在 dirty 字段上
- 读取时会先查询 read，不存在再查询 dirty，写入时则只写入 dirty
- 读取 read 并不需要加锁，而读或写 dirty 都需要加锁
- 另外有 misses 字段来统计 read 被穿透的次数（被穿透指需要读 dirty 的情况），超过一定次数则将 dirty 数据同步到 read 上
- 对于删除数据则直接通过标记来延迟删除

> 一些观点，当有大量并发读写发生的时候，会有很多的miss导致不断的dirty升级。可能会影响效率

### 13.Golang Runtime

#### runtime 包的方法

- **Gosched**：让当前线程让出 `cpu` 以让其它线程运行,它不会挂起当前线程，因此当前线程未来会继续执行
- **NumCPU**：返回当前系统的 `CPU` 核数量
- **GOMAXPROCS**：设置最大的可同时使用的 `CPU` 核数
- **Goexit**：退出当前 `goroutine`(但是`defer`语句会照常执行)
- **NumGoroutine**：返回正在执行和排队的任务总数
- **GOOS**：目标操作系统

### 14.Golang  引用类型

- golang 有三种引用类型  slice、map、channel
- Go 中函数传参仅有值传递一种方式；
- slice能够通过函数传参后，修改对应的数组值，是因为 slice 内部保存了引用数组的指针，并不是因为引用传递。
- 当对slice 进行 append 操作，当slice 发生扩容的时候，将不会对被引用的变量造成影响

### 15.Golang slice 扩容规则

- 如果切片的容量小于1024个元素，那么扩容的时候slice的cap就翻番，乘以2；一旦元素个数超过1024个元素，增长因子就变成1.25，即每次增加原来容量的四分之一。
- 如果扩容之后，还没有触及原数组的容量，那么，切片中的指针指向的位置，就还是原数组，如果扩容之后，超过了原数组的容量，那么，Go就会开辟一块新的内存，把原来的值拷贝过来，这种情况丝毫不会影响到原数组。

### 16.golang defer 修改返回值

defer  发生在return执行之后，内容返回之前。

考虑defer对返回值修改的影响，需要考虑两种情况:

1. 当生命了返回函数的名称的函数，返回值变量会再函数开始的时候自动生命，defer可以在函数真正返回之前，对返回值进行修改；
2. 匿名返回值则相当于，对return的值自动生成了一个变量来存储，defer对返回值的修改不会真正的改变返回值。

### 17.uniptr 和 unsafe.pointer 区别

- unsafe.Pointer只是单纯的通用指针类型，用于转换不同类型指针，它不可以参与指针运算；
- 而uintptr是用于指针运算的，GC 不把 uintptr 当指针，也就是说 uintptr 无法持有对象， uintptr 类型的目标会被回收；
- unsafe.Pointer 可以和 普通指针 进行相互转换；
- unsafe.Pointer 可以和 uintptr 进行相互转换。

### 18.Go中Map的实现原理

#### 什么是Map

##### key，value存储

最通俗的话说Map是一种通过key来获取value的一个数据结构，其底层存储方式为数组，在存储时key不能重复，当key重复时，value进行覆盖，我们通过key进行hash运算（可以简单理解为把key转化为一个整形数字）然后对数组的长度取余，得到key存储在数组的哪个下标位置，最后将key和value组装为一个结构体，放入数组下标处，看下图：

```
length = len(array) = 4
hashkey1 = hash(xiaoming) = 4
index1  = hashkey1% length= 0
hashkey2 = hash(xiaoli) = 6
index2  = hashkey2% length= 2
```

![1592037746323](/img/1592037746323.png)

##### hash冲突

如上图所示，数组一个下标处只能存储一个元素，也就是说一个数组下标只能存储一对key，value, hashkey(xiaoming)=4占用了下标0的位置，假设我们遇到另一个key，hashkey(xiaowang)也是4，这就是hash冲突（不同的key经过hash之后得到的值一样），那么key=xiaowang的怎么存储？

**hash冲突的常见解决方法**

开放定址法

也就是说当我们存储一个key，value时，发现hashkey(key)的下标已经被别key占用，那我们在这个数组中空间中重新找一个没被占用的存储这个冲突的key，那么没被占用的有很多，找哪个好呢？常见的有线性探测法，线性补偿探测法，随机探测法，这里我们主要说一下线性探测法

- 线性探测

  字面意思就是按照顺序来，从冲突的下标处开始往后探测，到达数组末尾时，从数组开始处探测，直到找到一个空位置存储这个key，当数组都找不到的情况下回扩容（事实上当数组容量快满的时候就会扩容了）；查找某一个key的时候，找到key对应的下标，比较key是否相等，如果相等直接取出来，否则按照顺寻探测直到碰到一个空位置，说明key不存在。如下图：

![1592037761105](/img/1592037761105.png)

​	首先存储key=xiaoming在下标0处，当存储key=xiaowang时，hash冲突了，按照线性探测，存储在下标1处，（红色的线是冲突或者下标已经被占用		了） 再者key=xiaozhao存储在下标4处，当存储key=xiaoliu是，hash冲突了，按照线性探测，从头开始，存储在下标2处 （黄色的是冲突或者下标已经被占用了）

- 拉链法

  何为拉链，简单理解为链表，当key的hash冲突时，我们在冲突位置的元素上形成一个链表，通过指针互连接，当查找时，发现key冲突，顺着链表一直往下找，直到链表的尾节点，找不到则返回空，如下图：

  ![1592037774396](/img/1592037774396.png)

**开放定址（线性探测）和拉链的优缺点**

1. 由上面可以看出拉链法比线性探测处理简单
2. 线性探测查找是会被拉链法会更消耗时间
3. 线性探测会更加容易导致扩容，而拉链不会
4. 拉链存储了指针，所以空间上会比线性探测占用多一点
5. 拉链是动态申请存储空间的，所以更适合链长不确定的

#### Go中的map实现原理

> **map的源码位于 src/runtime/map.go中 笔者go的版本是1.12**

在go中，map同样也是数组存储的，每个数组下标处存储的是一个bucket,这个bucket的类型见下面代码，每个bucket中可以存储8个kv键值对.。

当每个bucket存储的kv对到达8个之后，会通过overflow指针指向一个新的bucket，从而形成一个链表,看bmap的结构。

我想大家应该很纳闷，没看见kv的结构和overflow指针啊，事实上，这两个结构体并没有显示定义，是通过指针运算进行访问的

```go
//bucket结构体定义 b就是bucket
type bmap{
    //bucketCnt 的初始值是8
    tophash [bucketCnt]uint8
}
```

看上面代码以及注释，我们能得到bucket中存储的kv是这样的，tophash用来快速查找key值是否在该bucket中，而不同每次都通过真值进行比较；

还有kv的存放，为什么不是k1v1，k2v2..... 而是k1k2...v1v2...，我们看上面的注释说的 map[int64]int8,key是int64（8个字节），value是int8（一个字节），kv的长度不同，如果按照kv格式存放，则考虑内存对齐v也会占用int64，而按照后者存储时，8个v刚好占用一个int64,从这个就可以看出go的map设计之巧妙。

![1592037797275](/img/1592037797275.png)

最后我们分析一下go的整体内存结构，阅读一下map存储的源码，如下图所示，当往map中存储一个kv对时，通过k获取hash值，hash值的低八位和bucket数组长度取余，定位到在数组中的那个下标，hash值的高八位存储在bucket中的tophash中，用来快速判断key是否存在，key和value的具体值则通过指针运算存储，当一个bucket满时，通过overfolw指针链接到下一个bucket。

![1592037810278](/img/1592037810278.png)

## Kafka

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

### Kafka生产者分区机制

**轮询策略**

也称 Round-robin 策略，即顺序分配。比如一个主题下有 3 个分区，那么第一条消息被发送到分区 0，第二条被发送到分区 1，第三条被发送到分区 2，以此类推。当生产第 4 条消息时又会重新开始，即将其分配到分区 0。

**随机策略**

也称 Randomness 策略。所谓随机就是我们随意地将消息放置到任意一个分区上，如下面这张图所示。

**按消息key保存策略**

也称 Key-ordering 策略。这个可以理解为是自定义的策略之一。

Kafka 允许为每条消息定义消息键，简称为 Key。这个 Key 的作用非常大，它可以是一个有着明确业务含义的字符串，比如客户代码、部门编号或是业务 ID 等；也可以用来表征消息元数据。特别是在 Kafka 不支持时间戳的年代，在一些场景中，工程师们都是直接将消息创建时间封装进 Key 里面的。一旦消息被定义了 Key，那么你就可以保证同一个 Key 的所有消息都进入到相同的分区里面，由于每个分区下的消息处理都是有顺序的，故这个策略被称为按消息键保序策略

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

### producer 是否直接将数据发送到 broker 的 leader(主节点)？

producer 直接将数据发送到 broker 的 leader(主节点)，不需要在多个节点进行分发。

为了帮助 producer 做到这点，所有的 Kafka 节点都可以及时的告知:哪些节点是活动的，目标topic 目标分区的 leader 在哪。这样 producer 就可以直接将消息发送到目的地了

### Kafa consumer 是否可以消费指定分区消息？

Kafa consumer 消费消息时，向 broker 发出"fetch"请求去消费特定分区的消息，consumer指定消息在日志中的偏移量（offset），就可以消费从这个位置开始的消息，customer 拥有了 offset 的控制权，可以向后回滚去重新消费之前的消息，这是很有意义的

### Kafka 消息是采用 Pull 模式，还是 Push 模式？

Kafka 最初考虑的问题是，customer 应该从 brokes 拉取消息还是 brokers 将消息推送到consumer，也就是 pull 还 push。在这方面，Kafka 遵循了一种大部分消息系统共同的传统的设计：producer 将消息推送到 broker，consumer 从 broker 拉取消息

### Kafka 与传统消息系统之间有三个关键区别

(1).Kafka 持久化日志，这些日志可以被重复读取和无限期保留

(2).Kafka 是一个分布式系统：它以集群的方式运行，可以灵活伸缩，在内部通过复制数据

提升容错能力和高可用性

(3).Kafka 支持实时的流式处理

### kafka 的 ack 机制

request.required.acks 有三个值 0 1 -1

0:生产者不会等待 broker 的 ack，这个延迟最低但是存储的保证最弱当 server 挂掉的时候

就会丢数据

1：服务端会等待 ack 值 leader 副本确认接收到消息后发送 ack 但是如果 leader 挂掉后他

不确保是否复制完成新 leader 也会导致数据丢失

-1：同样在 1 的基础上 服务端会等所有的 follower 的副本受到数据后才会受到 leader 发出

的 ack，这样数据不会丢失

### 消息重复消费和消息丢包的解决办法

保证不丢失消息：生产者（ack=all 代表至少成功发送一次)     重试机制

保证不重复消费:  1. offset手动提交，业务逻辑成功处理后，提交offset。 2. 落表（主键或者唯一索引的方式，避免重复数据） 

## Flink

### Flink 分层 

**SQL & Table API**

SQL & Table API 同时适用于批处理和流处理，这意味着你可以对有界数据流和无界数据流以相同的语义进行查询，并产生相同的结果。除了基本查询外， 它还支持自定义的标量函数，聚合函数以及表值函数，可以满足多样化的查询需求。 

**DataStream & DataSet API**

DataStream &  DataSet API 是 Flink 数据处理的核心 API，支持使用 Java 语言或 Scala 语言进行调用，提供了数据读取，数据转换和数据输出等一系列常用操作的封装。

**Stateful Stream Processing**

Stateful Stream Processing 是最低级别的抽象，它通过 Process Function 函数内嵌到 DataStream API 中。 Process Function 是 Flink 提供的最底层 API，具有最大的灵活性，允许开发者对于时间和状态进行细粒度的控制。

### Flink 集群架构

#### 核心组件

按照上面的介绍，Flink 核心架构的第二层是 Runtime 层， 该层采用标准的 Master - Slave 结构， 其中，Master 部分又包含了三个核心组件：Dispatcher、ResourceManager 和 JobManager，而 Slave 则主要是 TaskManager 进程。它们的功能分别如下：

- **Client**： 用户提交一个Flink程序时，会首先创建一个Client，该Client首先会对用户提交的Flink程序进行预处理，并提交到Flink集群， Client会将用户提交的Flink程序组装一个JobGraph， 并且是以JobGraph的形式提交的
- **Dispatcher**：负责接收客户端提交的执行程序，并传递给 JobManager 。除此之外，它还提供了一个 WEB UI 界面，用于监控作业的执行情况。
- **JobManagers** (也称为 *masters*) ：JobManagers 接收由 Dispatcher 传递过来的执行程序，该执行程序包含了作业图 (JobGraph)，逻辑数据流图 (logical dataflow graph) 及其所有的 classes 文件以及第三方类库 (libraries) 等等 。紧接着 JobManagers 会将 JobGraph 转换为执行图 (ExecutionGraph)，然后向 ResourceManager 申请资源来执行该任务，一旦申请到资源，就将执行图分发给对应的 TaskManagers 。因此每个作业 (Job) 至少有一个 JobManager；高可用部署下可以有多个 JobManagers，其中一个作为 *leader*，其余的则处于 *standby* 状态。
- **ResourceManager** ：负责管理 slots 并协调集群资源。ResourceManager 接收来自 JobManager 的资源请求，并将存在空闲 slots 的 TaskManagers 分配给 JobManager 执行任务。Flink 基于不同的部署平台，如 YARN , Mesos，K8s 等提供了不同的资源管理器，当 TaskManagers 没有足够的 slots 来执行任务时，它会向第三方平台发起会话来请求额外的资源。
- **TaskManagers** (也称为 *workers*) : TaskManagers 负责实际的子任务 (subtasks) 的执行，每个 TaskManagers 都拥有一定数量的 slots。Slot 是一组固定大小的资源的合集 (如计算能力，存储空间)。TaskManagers 启动后，会将其所拥有的 slots 注册到 ResourceManager 上，由 ResourceManager 进行统一管理。

![1592037943603](/img/1592037943603.png)

#### Task & SubTask

上面我们提到：TaskManagers 实际执行的是 SubTask，而不是 Task，这里解释一下两者的区别：

在执行分布式计算时，Flink 将可以链接的操作 (operators) 链接到一起，这就是 Task。之所以这样做， 是为了减少线程间切换和缓冲而导致的开销，在降低延迟的同时可以提高整体的吞吐量。 但不是所有的 operator 都可以被链接，如下 keyBy 等操作会导致网络 shuffle 和重分区，因此其就不能被链接，只能被单独作为一个 Task。  简单来说，一个 Task 就是一个可以链接的最小的操作链 (Operator Chains) 。如下图，source 和 map 算子被链接到一块，因此整个作业就只有三个 Task：

![1592037955656](/img/1592037955656.png)

解释完 Task ，我们在解释一下什么是 SubTask，其准确的翻译是： *A subtask is one parallel slice of a task*，即一个 Task 可以按照其并行度拆分为多个 SubTask。如上图，source & map 具有两个并行度，KeyBy 具有两个并行度，Sink 具有一个并行度，因此整个虽然只有 3 个 Task，但是却有 5 个 SubTask。Jobmanager 负责定义和拆分这些 SubTask，并将其交给 Taskmanagers 来执行，每个 SubTask 都是一个单独的线程。

#### 资源管理

理解了 SubTasks ，我们再来看看其与 Slots 的对应情况。一种可能的分配情况如下：

![1592037964625](/img/1592037964625.png)

这时每个 SubTask 线程运行在一个独立的 TaskSlot， 它们共享所属的 TaskManager 进程的TCP 连接（通过多路复用技术）和心跳信息 (heartbeat messages)，从而可以降低整体的性能开销。此时看似是最好的情况，但是每个操作需要的资源都是不尽相同的，这里假设该作业 keyBy 操作所需资源的数量比 Sink 多很多 ，那么此时 Sink 所在 Slot 的资源就没有得到有效的利用。

基于这个原因，Flink 允许多个 subtasks 共享 slots，即使它们是不同 tasks 的 subtasks，但只要它们来自同一个 Job 就可以。假设上面 souce & map 和 keyBy 的并行度调整为 6，而 Slot 的数量不变，此时情况如下：

![1592037974706](/img/1592037974706.png)

可以看到一个 Task Slot 中运行了多个 SubTask 子任务，此时每个子任务仍然在一个独立的线程中执行，只不过共享一组 Sot 资源而已。那么 Flink 到底如何确定一个 Job 至少需要多少个 Slot 呢？Flink 对于这个问题的处理很简单，默认情况一个 Job 所需要的 Slot 的数量就等于其 Operation 操作的最高并行度。如下， A，B，D 操作的并行度为 4，而 C，E 操作的并行度为 2，那么此时整个 Job 就需要至少四个 Slots 来完成。通过这个机制，Flink 就可以不必去关心一个 Job 到底会被拆分为多少个 Tasks 和 SubTasks。

![1592037985130](/img/1592037985130.png)

### Flink 优势

- 支持高吞吐、低延迟、高性能的流处理
- 支持高度灵活的窗口（Window）操作
- 支持有状态计算的Exactly-once语义
- 提供DataStream API和DataSet API

### Streaming Connectors

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

### Flink 状态

相对于其他流计算框架，Flink 一个比较重要的特性就是其支持有状态计算。即你可以将中间的计算结果进行保存，并提供给后续的计算使用：

具体而言，Flink 又将状态 (State) 分为 Keyed State 与 Operator State：

**算子状态**

算子状态 (Operator State)：顾名思义，状态是和算子进行绑定的，一个算子的状态不能被其他算子所访问到。官方文档上对 Operator State 的解释是：*each operator state is bound to one parallel operator instance*，所以更为确切的说一个算子状态是与一个并发的算子实例所绑定的，即假设算子的并行度是 2，那么其应有两个对应的算子状态：

![1586425914911](/img/1586425914911.png)

**键控状态**

键控状态 (Keyed State) ：是一种特殊的算子状态，即状态是根据 key 值进行区分的，Flink 会为每类键值维护一个状态实例。如下图所示，每个颜色代表不同 key 值，对应四个不同的状态实例。需要注意的是键控状态只能在 `KeyedStream` 上进行使用，我们可以通过 `stream.keyBy(...)` 来得到 `KeyedStream` 。

![1586425957804](/img/1586425957804.png)

### 检查点机制

**checkpoint**

为了使 Flink 的状态具有良好的容错性，Flink 提供了检查点机制 (CheckPoints)  。通过检查点机制，Flink 定期在数据流上生成 checkpoint barrier ，当某个算子收到 barrier 时，即会基于当前状态生成一份快照，然后再将该 barrier 传递到下游算子，下游算子接收到该 barrier 后，也基于当前状态生成一份快照，依次传递直至到最后的 Sink 算子上。当出现异常后，Flink 就可以根据最近的一次的快照数据将所有算子恢复到先前的状态。

**开启检查点**

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

### Event time 乱序问题

#### 使用wartermark + 时间延时

Watermark是Apache Flink为了处理EventTime 窗口计算提出的一种机制,本质上也是一种时间戳，由Watermark生成器按照需求Punctuated或者Periodic两种方式生成的一种系统Event，与普通数据流Event一样流转到对应的下游算子，接收到Watermark Event的算子以此不断调整自己管理的EventTime clock。

Apache Flink 框架保证Watermark单调递增，算子接收到一个Watermark时候，框架知道不会再有任何小于该Watermark的时间戳的数据元素到来了，所以Watermark可以看做是告诉Apache Flink框架数据流已经处理到什么位置(时间维度)的方式。 Watermark的产生和Apache Flink内部处理逻辑如下图所示: 

**如果想正确处理迟来的数据可以定义Watermark生成策略为 Watermark = EventTime -5s**

**BoundedOutOfOrdernessTimestampExtractor**

#### Watermark的产生方式

目前Apache Flink 有两种生产Watermark的方式，如下：

- Punctuated - 数据流中每一个递增的EventTime都会产生一个Watermark。 

  在实际的生产中Punctuated方式在TPS很高的场景下会产生大量的Watermark在一定程度上对下游算子造成压力，所以只有在实时性要求非常高的场景才会选择Punctuated的方式进行Watermark的生成。

- Periodic - 周期性的（一定时间间隔或者达到一定的记录条数）产生一个Watermark。在实际的生产中Periodic的方式必须结合时间和积累条数两个维度继续周期性产生Watermark，否则在极端情况下会有很大的延时。

  所以Watermark的生成方式需要根据业务场景的不同进行不同的选择。

## Hive

### Hive 的数据模型

- db：在hdfs中表现为${hive.metastore.warehouse.dir}目录下一个文件夹

- table：在hdfs中表现所属db目录下一个文件夹

- external table：外部表, 与table类似，不过其数据存放位置可以在任意指定路径

  > 普通表: 删除表后, hdfs上的文件都删了
  >
  > External外部表: 删除后, hdfs上的文件没有删除, 只是把文件删除了

- partition：在hdfs中表现为table目录下的子目录

- bucket：桶, 在hdfs中表现为同一个表目录下根据hash散列之后的多个文件, 会根据不同的文件把数据放到不同的文件中 

### Hive 架构图

![1588041374336](/img/1588041374336.png)

- 用户接口: CLI、JDBC/ODBC、WebGUI

  CLI为shell命令行；JDBC/ODBC是Hive的JAVA实现，与传统数据库JDBC类似；WebGUI是通过浏览器访问Hive。

- 元数据存储：一般存储在关系型数据库中如 mysql；

  Hive 将元数据存储在数据库中。Hive 中的元数据包括表的名字，表的列和分区及其属性，表的属性（是否为外部表等），表的数据所在目录等。

- 解释器、编译器、优化器、执行器

  解释器、编译器、优化器完成 HQL 查询语句从词法分析、语法分析、编译、优化以及查询计划的生成。生成的查询计划存储在 HDFS 中，并在随后有 MapReduce 调用执行。

### Hive基本操作

#### 创建表

**建表语句**

```sql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name 
   [(col_name data_type [COMMENT col_comment], ...)] 
   [COMMENT table_comment] 
   [PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)] 
   [CLUSTERED BY (col_name, col_name, ...) 
   [SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS] 
   [ROW FORMAT row_format] 
   [STORED AS file_format] 
   [LOCATION hdfs_path]
```

- CREATE TABLE 创建一个指定名字的表。如果相同名字的表已经存在，则抛出异常；用户可以用 IF NOT EXISTS 选项来忽略这个异常。

- EXTERNAL关键字可以让用户创建一个外部表，在建表的同时指定一个指向实际数据的路径（LOCATION）

- LIKE 允许用户复制现有的表结构，但是不复制数据。

- ROW FORMAT DELIMITED \[FIELDS TERMINATED BY char\]\[COLLECTION ITEMS TERMINATED BY char\] \[MAP KEYS TERMINATED BY char\]\[LINES TERMINATED BY char\] | SERDE serde_name [WITH SERDEPROPERTIES (property_name=property_value, property_name=property_value, ...)]

- STORED AS    SEQUENCEFILE|TEXTFILE|RCFILE     如果文件数据是纯文本，可以使用 STORED AS TEXTFILE(默认)。如果数据需要压缩，使用 STORED AS SEQUENCEFILE。

- CLUSTERED BY      对于每一个表（table）或者分区， Hive可以进一步组织成桶，也就是说桶是更为细粒度的数据范围划分。Hive也是 针对某一列进行桶的组织。Hive采用对列值哈希，然后除以桶的个数求余的方式决定该条记录存放在哪个桶当中。 

  把表（或者分区）组织成桶（Bucket）有两个理由：

  （1）获得更高的查询处理效率。桶为表加上了额外的结构，Hive 在处理有些查询时能利用这个结构。具体而言，连接两个在（包含连接列的）相同列上划分了桶的表，可以使用 Map 端连接 （Map-side join）高效的实现。比如JOIN操作。对于JOIN操作两个表有一个相同的列，如果对这两个表都进行了桶操作。那么将保存相同列值的桶进行JOIN操作就可以，可以大大较少JOIN的数据量。

  （2）使取样（sampling）更高效。在处理大规模数据集时，在开发和修改查询的阶段，如果能在数据集的一小部分数据上试运行查询，会带来很多方便。

> 备注: hive 的join操作只支持等值连接

**建表示例**

```sql
create table student(id int, age int, name string) partitioned by(stat_date string)  clustered by(id) sorted by(age) into 2 buckets row format delimited fields terminated by',';
```

**增加分区示例**

```sql
alter table student_p add partition(part='a') partition(part='b');
```

#### 数据查询

**查询语句**

```sql
SELECT [ALL | DISTINCT] select_expr, select_expr, ... 
FROM table_reference
[WHERE where_condition] 
[GROUP BY col_list [HAVING condition]] 
[ [CLUSTER BY col_list] | [DISTRIBUTE BY col_list] [SORT BY| ORDER BY col_list] ] 
[LIMIT number]
```

1. order by 会对输入做全局排序，因此只有一个reducer，会导致当输入规模较大时，需要较长的计算时间。
2. sort by不是全局排序，其在数据进入reducer前完成排序。因此，如果用sort by进行排序，并且设置mapred.reduce.tasks>1，则sort by只保证每个reducer的输出有序，不保证全局有序。
3. 根据distribute by指定的字段内容将数据分到同一个reducer。
4. Cluster by 除了具有Distribute by的功能外，还会对该字段进行排序。因此，常常认为cluster by = distribute by + sort by

**查询示例**

```sql
set mapred.reduce.tasks=4;
select id, age, name from stuent distribute by age sort by age desc;
```

#### Hive Join操作

**1. 语法结构**

```shell
table1 JOIN table2 [join_condition]
  | table1 {LEFT|RIGHT|FULL} [OUTER] JOIN table2 join_condition
  | table1 LEFT SEMI JOIN table2 join_condition
```

> 备注: 
>
>  Hive 支持等值连接（equality joins）、外连接（outer joins）和（left/right joins）。Hive **不支持非等值的连接**，因为非等值连接非常难转化到 map/reduce 任务。另外，Hive 支持多于 2 个表的连接。
>
> left semi join 是 IN/EXISTS 的高效实现

**等值join**

1. **等值join**

   ```sql
   SELECT a.* FROM a JOIN b ON (a.id = b.id)；
   SELECT a.* FROM a JOIN b ON (a.id = b.id AND a.department = b.department)；
   # 是正确的，然而:
   SELECT a.* FROM a JOIN b ON (a.id>b.id)
   # 是错误的。
   ```

2. **可以多于两张表**

   ```sql
   SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key2);
   # 如果join中多个表的 join key 是同一个，则 join 会被转化为单个 map/reduce 任务，例如：
   SELECT a.val, b.val, c.val FROM a JOIN b
   ON (a.key = b.key1) JOIN c
   ON (c.key = b.key1)
   # 被转化为单个 map/reduce 任务，因为 join 中只使用了 b.key1 作为 join key。
   SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1)
   JOIN c ON (c.key = b.key2)
   # 而这一 join 被转化为 2 个 map/reduce 任务。因为 b.key1 用于第一次 join 条件，而 b.key2 用于第二次 join。
   ```

3. **join时，每次map/reduce任务的逻辑**

   reducer 会缓存 join 序列中除了最后一个表的所有表的记录，再通过最后一个表将结果序列化到文件系统。这一实现有助于在 reduce 端减少内存的使用量。实践中，应该把最大的那个表写在最后（否则会因为缓存浪费大量内存）。例如：

   ```sql
      SELECT a.val, b.val, c.val FROM a
      JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key1)
   ```

   所有表都使用同一个 join key（使用 1 次 map/reduce 任务计算）。Reduce 端会缓存 a 表和 b 表的记	录，然后每次取得一个 c 表的记录就计算一次 join 结果，类似的还有：

   ```sql
   SELECT a.val, b.val, c.val FROM a
   JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key2)
   ```

   这里用了 2 次 map/reduce 任务。第一次缓存 a 表，用 b 表序列化；第二次缓存第一次 map/reduce 任务的结果，然后用 c 表序列化。

4. **LEFT，RIGHT 和 FULL OUTER 关键字用于处理 join 中空记录的情况**

   ```sql
   SELECT a.val, b.val FROM 
   a LEFT OUTER  JOIN b ON (a.key=b.key)
   ```

   对应所有 a 表中的记录都有一条记录输出。输出的结果应该是 a.val, b.val，当 a.key=b.key 时，而当 b.key 中找不到等值的 a.key 记录时也会输出:a.val, NULL

   所以 a 表中的所有记录都被保留了，a RIGHT OUTER JOIN b”会保留所有 b 表的记录。

   Join 发生在 WHERE 子句之前。如果你想限制 join 的输出，应该在 WHERE 子句中写过滤条件——或是在 join 子句中写。这里面一个容易混淆的问题是表分区的情况：

   ```sql
   SELECT a.val, b.val FROM a
   LEFT OUTER JOIN b ON (a.key=b.key)
   WHERE a.ds='2009-07-07' AND b.ds='2009-07-07'
   ```

   会 join a 表到 b 表（OUTER JOIN），列出 a.val 和 b.val 的记录。WHERE 从句中可以使用其他列作为过滤条件。但是，如前所述，如果 b 表中找不到对应 a 表的记录，b 表的所有列都会列出 NULL，**包括 ds 列**。也就是说，join 会过滤 b 表中不能找到匹配 a 表 join key 的所有记录。这样的话，LEFT OUTER 就使得查询结果与 WHERE 子句无关了。解决的办法是在 OUTER JOIN 时使用以下语法：

   ```sql
     SELECT a.val, b.val FROM a LEFT OUTER JOIN b
     ON (a.key=b.key AND
         b.ds='2009-07-07' AND
         a.ds='2009-07-07')
   ```

   这一查询的结果是预先在 join 阶段过滤过的，所以不会存在上述问题。这一逻辑也可以应用于 RIGHT 和 FULL 类型的 join 中。

5. **Join 是不能交换位置的。**

   无论是 LEFT 还是 RIGHT join，都是左连接的。

   ```sql
   SELECT a.val1, a.val2, b.val, c.valFROM a
   JOIN b ON (a.key = b.key)
   LEFT OUTER JOIN c ON (a.key = c.key)
   ```

   先 join a 表到 b 表，丢弃掉所有 join key 中不匹配的记录，然后用这一中间结果和 c 表做 join。这一表述有一个不太明显的问题，就是当一个 key 在 a 表和 c 表都存在，但是 b 表中不存在的时候：整个记录在第一次 join，即 a JOIN b 的时候都被丢掉了（包括a.val1，a.val2和a.key），然后我们再和 c 表 join 的时候，如果 c.key 与 a.key 或 b.key 相等，就会得到这样的结果：NULL, NULL, NULL, c.val

### hive 内部表和外部表区别

- Hive 创建内部表时，会将数据移动到数据仓库指向的路径，hive 管理数据的声明周期，若创建外部表，仅记录数据所在的路径，不会对数据的位置做任何改变。
- 在删除表的时候，内部表的元数据和数据会被一起删除， 外部表只会删除外部表的元数据， 不删除数据，这样外部表相对来说更安全一些，数据组织也更加灵活，方便共享源数据。；
- 对于一些 hive 操作不适应于外部表，比如单个查询语句创建表并向表中插入数据。
- 创建外部表时甚至不需要知道外部数据是否存在，可以把创建数据推迟到创建表之后才进行。

### hive和传统数据库区别

1. 需要注意的是传统数据库对表数据检验是 schema on write（写时模式）而 Hive 在 load时不检查数据是否符合 schema 的，hive 遵循的是 schema on read（读时模式），只有在读的时候 hive 才检查、解析具体的数据字段、schema。读时模式的优势是 load data 非常迅速，因为它不需要读取数据进行解析，仅仅进行文件的复制或者移动。写时模式的优势是提升了查询性能，因为预先解析之后可以对列建立索引并压缩，但这样会花费较多的加载时间。即使为内部表在数据加载时也不解析数据格式，如果数据和模式不匹配，只能在查询时出现 null 才知道不匹配的行。
2. hive 具有复杂的数据结构（数组，映射，结构体）
3. hive 不支持实时数据处理，对索引的支持较弱。
4. hive 不支持行级的插入。
5. 延迟高，数据量大，多存储在 hdfs 上。
6. 执行为 mapreducer。
7. hive 不支持行级操作也不支持事务。

### hive 分区的了解

是对 hive 表的一种组织形式，可以加快查询，是一种对表进行粗略划分的机制。使用分区时，在表目录下会有相应的子目录，当查询时若添加了分区谓词，该查询会定位到相应的子目录中进行查询，避免全表扫描，比如日志文件分析，将日志按天存储。分区并不会影响大范围的查询。

外部表也可以分区，具有很好的灵活性，例如
这种灵活性有一个有趣的优点是我们可以使用像 Amazon S3 这样的廉价的存储设备存储旧的数据，同时保持较新的数据到 HDFS 中。例如，每天我们可以使用如下的处理过程将一个月前的旧数据转移到 S3 中。

### 使用hive分区的注意事项

1. 当分区过多且数据很大时，可以使用严格模式，避免触发一个大的 mapreducer 任务。当分区数量过多且数据量较大时，执行宽范围的数据扫描会触发一个很大的 mapreducer 任务，在严格模式下，当 where 中没有分区过滤条件时会禁止执行。
2. hive 如果有过多的分区，由于底层是存储在 HDFS 上，HDFS适用于存储大文件而非小文件，因此过多的分区会增加 namenode 负担。
3. hive 会转化成 mapreducer，mapreducer 会转化为多个 task，过多的小文件的话，每个文件一个task，每个task 一个 jvm实例，jvm 开启与销毁会降低系统效率。

### Hive 分桶的理由

1. 取样更高效。具体划分桶是将值进行hash，然后除以桶的个数取余，任何一个桶内部是一个随机划分的用户集合。在处理大规模数据集时，在开发和修改查询的阶段，如果能在数据集的一小部分上试运行查询，会方便很多。
2. 获得更高的查询处理效率。桶为表加上了额外的结构，Hive 在处理有些查询时能利用这个结构。具体而言，连接两个在（包含连接列的）相同列上划分了桶的表，可以使用 Map 端连接（Map-side join）高效的实现。处理左边表的某个桶的 mapper 就知道右边表内相匹配的行在对应的桶，这样 mapper 直接就可以在对应的右边表的桶获取数据进行 join。并不一定要求两个表必须有相同的桶的个数，倍数也行。

## Mysql

### B 树

**B树特征**

1. 根结点至少有两个子女。
2. 每个中间节点都包含k-1个元素和k个孩子，其中 m/2 <= k <= m
3. 每一个叶子节点都包含k-1个元素，其中 m/2 <= k <= m
4. 所有的叶子结点都位于同一层。
5. 每个节点中的元素从小到大排列，节点当中k-1个元素正好是k个孩子包含的元素的值域分划。

**B+ 树特征**

1. 有k个子树的中间节点包含有k个元素（B树中是k-1个元素），每个元素不保存数据，只用来索引，所有数据都保存在叶子节点。
2. 所有的叶子结点中包含了全部元素的信息，及指向含这些元素记录的指针，且叶子结点本身依关键字的大小自小而大顺序链接。
3. 所有的中间节点元素都同时存在于子节点，在子节点元素中是最大（或最小）元素。

![1588818370345](/img/1588818370345.png)

**B+树的优势：**

1. 单一节点存储更多的元素，使得查询的IO次数更少。
2. 所有查询都要查找到叶子节点，查询性能稳定。
3. 所有叶子节点形成有序链表，便于范围查询。

> 备注: m为阶数

### Mysql存储引擎

#### 存储引擎的区别

> 联机事务处理OLTP（on-line transaction processing）:传统的关系型数据库的主要应用，主要是基本的、日常的事务处理，例如银行交易。
>
> 联机分析处理OLAP（On-Line Analytical Processing）:是数据仓库系统的主要应用，支持复杂的分析操作，侧重决策支持，并且提供直观易懂的查询结果

#### InnoDB:

- InnoDB 存储引擎支持事务、支持外键、支持非锁定读、行锁设计其设计主要面向OLTP 应用。
- InnoDB 存储引擎表采用聚集的方式存储，因此每张表的存储顺序都按主键的顺序存放，如果没有指定主键，InnoDB 存储引擎会为每一行生成一个6字节的ROWID并以此作为主键。
- InnoDB 存储引擎通过MVCC 获的高并发性，并提供了插入缓冲、二次写、自适应哈希索引和预读等高性能高可用功能
- InnoDB 存储引擎默认隔离级别为REPEATABLE_READ（重复读）并采用next-key locking(间隙锁)和MVCC来避免幻读

#### MySIAM:

- MYISAM 存储引擎不支持事务、表锁设计、支持全文索引其设计主要面向OLAP 应用
- MYISAM 存储引擎表由frm、MYD 和MYI 组成，frm 文件存放表格定义，MYD 用来存放数据文件，MYI 存放索引文件。MYISAM 存储引擎与众不同的地方在于它的缓冲池只缓存索引文件而不缓存数据文件，数据文件的缓存依赖于操作系统。
  操作区别：

> MYISAM 保存表的具体行数，不带where 是可直接返回。InnoDB 要扫描全表。
> DELETE 表时，InnoDB 是一行一行的删除，MYISAM 是先drop表，然后重建表
> InnoDB 跨平台可直接拷贝使用，MYISAM 不行
> InnoDB 表格很难被压缩，MYISAM 可以

#### 引擎选择：

MyISAM相对简单所以在效率上要优于InnoDB。

如果系统读多，写少。对原子性要求低,那么MyISAM最好的选择。且MyISAM恢复速度快。可直接用备份覆盖恢复。

InnoDB 更适合系统读少，写多的时候，尤其是高并发场景。

### Mysql索引

#### mysql 索引介绍

Mysql 中常用的索引有B+ 树索引（包括普通索引、唯一索引、主键索引），哈希索引，全文索引，R-TREE 索引（空间索引，主要用于地理空间数据类型，很少使用）。

Mysql 传统意义上的索引为B+ 树索引，B+ 树索引的本质就是B+ 树在数据库中的实现，由于B+ 树的高扇出性，数据库中的B+ 树的高一般为2-4层，因此查找某一键值的行记录只需2-4次IO，大概0.02~0.04秒。

> （扇出性：是指该模块直接调用的下级模块的个数。扇出大表示模块的复杂度高，需要控制和协调过多的下级模块）

#### B+ 树索引分类

**聚集索引和辅助索引**

- 聚集索引是根据每张表的主键建造的一棵B+ 树，叶子节点中存放的是整张表的行记录。一张表只能有一个聚集索引。因为聚集索引在逻辑上是连续的，所以它对于主键的排序查找和范围查找速度非常快。
- 辅助索引与聚集索引不同的地方在于，辅助索引不是唯一的，它的叶子节点只包含行记录的部分数据以及对应聚集索引的节点位置。通过辅助索引来查找数据时，先遍历辅助索引找到对应主键索引，再通过主键索引查找对应记录。

> 在MYISAM 中主键索引和辅助索引都相当上述辅助索引，索引页中存放的是主键和指向数据页的偏移量，数据页中存放的是主键和该主键所属行记录的地址空间。唯一的区别是MYISAM 中主键索引不能重复，辅助索引可以。

**联合索引和覆盖索引**

- 联合索引是指对表上的多个列进行索引。它对对应多个列的指定获取比较快。另外一个好处是联合索引对第二个键已经排好序了，所以对两个列的排序获取可以避免多做一次排序操作。
- 覆盖索引其实更算一种思想，能够从辅助索引中获取信息，就不需要查询聚集索引中的数据。使用辅助索引的好处在于辅助索引包含的信息少，所以大小远小于聚集索引，因此可以大大减少IO 操作。

#### hash索引

哈希索引是一种自适应的索引，数据库会根据表的使用情况自动生成哈希索引，我们人为是没办法干预的。

InnoDB 储存引擎采用的哈希函数为除法散列方式，采用的冲突处理方法为链地址法。它指定查询的速度很快，但是范围查询就无能为力了。

#### 全文索引

全文索引用于实现关键词搜索。但它只能根据空格分词，因此不支持中文。

#### 索引的优缺点

**索引的优点：**

- 通过创建唯一性索引，可以保证数据库表中每一行数据的唯一性。
- 可以大大加快 数据的检索速度，这也是创建索引的最主要的原因。
- 可以加速表和表之间的连接，特别是在实现数据的参考完整性方面特别有意义。
- 在使用分组和排序 子句进行数据检索时，同样可以显著减少查询中分组和排序的时间。
- 通过使用索引，可以在查询的过程中，使用优化隐藏器，提高系统的性能。

**索引的缺点**

- 创建索引和维护索引要耗费时间，这种时间随着数据量的增加而增加。
- 索引需要占物理空间，除了数据表占数据空间之外，每一个索引还要占一定的物理空间，如果要建立聚簇索引，那么需要的空间就会更大。
- 当对表中的数据进行增加、删除和修改的时候，索引也要动态的维护，这样就降低了数据的维护速度。
- 哪些情况需要加索引？

### Mysql 锁

数据库锁设计的初衷是处理并发问题。作为多用户共享的资源，当出现并发访问的时候，数据库需要合理地控制资源的访问规则。而锁就是用来实现这些访问规则的重要数据结构。

根据加锁的范围，MySQL 里面的锁大致可以分成全局锁、表级锁和行锁三类。

#### mysql锁的分类

**全局锁**

顾名思义，全局锁就是对整个数据库实例加锁。MySQL 提供了一个加全局读锁的方法，命令是 Flush tables with read lock (FTWRL)。

当你需要让整个库处于只读状态的时候，可以使用这个命令，之后其他线程的以下语句会被阻塞：数据更新语句（数据的增删改）、数据定义语句（包括建表、修改表结构等）和更新类事务的提交语句。

**全局锁的典型使用场景是，做全库逻辑备份。**也就是把整库每个表都 select 出来存成文本。

官方自带的逻辑备份工具是 mysqldump。当 mysqldump 使用参数–single-transaction 的时候，导数据之前就会启动一个事务，来确保拿到一致性视图。而由于 MVCC 的支持，这个过程中数据是可以正常更新的。

你一定在疑惑，有了这个功能，为什么还需要 FTWRL 呢？**一致性读是好，但前提是引擎要支持这个隔离级别。**比如，对于 MyISAM 这种不支持事务的引擎，如果备份过程中有更新，总是只能取到最新的数据，那么就破坏了备份的一致性。这时，我们就需要使用 FTWRL 命令了。

所以，**single-transaction 方法只适用于所有的表使用事务引擎的库。**如果有的表使用了不支持事务的引擎，那么备份就只能通过 FTWRL 方法。这往往是 DBA 要求业务开发人员使用 InnoDB 替代 MyISAM 的原因之一。

**表级锁**

MySQL 里面表级别的锁有两种：一种是表锁，一种是元数据锁（meta data lock，MDL)。

表锁的语法是 lock tables … read/write。与 FTWRL 类似，可以用 unlock tables 主动释放锁，也可以在客户端断开的时候自动释放。

需要注意，lock tables 语法除了会限制别的线程的读写外，也限定了本线程接下来的操作对象。

在还没有出现更细粒度的锁的时候，表锁是最常用的处理并发的方式。而对于 InnoDB 这种支持行锁的引擎，一般不使用 lock tables 命令来控制并发，毕竟锁住整个表的影响面还是太大。

**元数据锁**

MDL 不需要显式使用，在访问一个表的时候会被自动加上。MDL 的作用是，保证读写的正确性。你可以想象一下，如果一个查询正在遍历一个表中的数据，而执行期间另一个线程对这个表结构做变更，删了一列，那么查询线程拿到的结果跟表结构对不上，肯定是不行的。

因此，在 MySQL 5.5 版本中引入了 MDL，当对一个表做增删改查操作的时候，加 MDL 读锁；当要对表做结构变更操作的时候，加 MDL 写锁。

- 读锁之间不互斥，因此你可以有多个线程同时对一张表增删改查。
- 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行。

需要注意的是：MDL 会直到事务提交才释放，在做表结构变更的时候，你一定要小心不要导致锁住线上查询和更新。

**行锁**

MySQL 的行锁是在引擎层由各个引擎自己实现的。但并不是所有的引擎都支持行锁，比如 MyISAM 引擎就不支持行锁。不支持行锁意味着并发控制只能使用表锁，对于这种引擎的表，同一张表上任何时刻只能有一个更新在执行，这就会影响到业务并发度。InnoDB 是支持行锁的，这也是 MyISAM 被 InnoDB 替代的重要原因之一。

**在 InnoDB 事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是两阶段锁协议。**

知道了这个设定，对我们使用事务有什么帮助呢？那就是，如果你的事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放。

举个例子。

假设你负责实现一个电影票在线交易业务，顾客 A 要在影院 B 购买电影票。我们简化一点，这个业务需要涉及到以下操作：

1. 从顾客 A 账户余额中扣除电影票价；
2. 给影院 B 的账户余额增加这张电影票价；
3. 记录一条交易日志。

也就是说，要完成这个交易，我们需要 update 两条记录，并 insert 一条记录。当然，为了保证交易的原子性，我们要把这三个操作放在一个事务中。那么，你会怎样安排这三个语句在事务中的顺序呢？

试想如果同时有另外一个顾客 C 要在影院 B 买票，那么这两个事务冲突的部分就是语句 2 了。因为它们要更新同一个影院账户的余额，需要修改同一行数据。

根据两阶段锁协议，不论你怎样安排语句顺序，所有的操作需要的行锁都是在事务提交的时候才释放的。所以，如果你把语句 2 安排在最后，比如按照 3、1、2 这样的顺序，那么影院账户余额这一行的锁时间就最少。这就最大程度地减少了事务之间的锁等待，提升了并发度。

#### 死锁和死锁检测

当并发系统中不同线程出现循环资源依赖，涉及的线程都在等待别的线程释放资源时，就会导致这几个线程都进入无限等待的状态，称为死锁。

![1588819238826](/img/1588819238826.png)

这时候，事务 A 在等待事务 B 释放 id=2 的行锁，而事务 B 在等待事务 A 释放 id=1 的行锁。 事务 A 和事务 B 在互相等待对方的资源释放，就是进入了死锁状态。当出现死锁以后，有两种策略：

- 一种策略是，直接进入等待，直到超时。这个超时时间可以通过参数 innodb_lock_wait_timeout 来设置。
- 另一种策略是，发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。将参数 innodb_deadlock_detect 设置为 on，表示开启这个逻辑。

减少死锁的主要方向，就是控制访问相同资源的并发事务量。

### Mysql 乐观锁&悲观锁

乐观并发控制和悲观并发控制是并发控制采用的主要方法。

乐观锁和悲观锁不仅在关系数据库里应用，在Hibernate、Memcache等等也有相关概念。

#### 1. 悲观锁

现在互联网高并发的架构中，受到fail-fast思路的影响，悲观锁已经非常少见了。

悲观锁(Pessimistic Locking)，悲观锁是指在数据处理过程，使数据处于锁定状态，一般使用数据库的锁机制实现。

**1.1 数据表中的实现**

在MySQL中使用悲观锁，必须关闭MySQL的自动提交，set autocommit=0，MySQL默认使用自动提交autocommit模式，也即你执行一个更新操作，MySQL会自动将结果提交。

```bash
set autocommit=0
```

举个🌰栗子：

假设商品表中有一个字段quantity表示当前该商品的库存量。假设有一件Dulex套套，其id为100，quantity=8个；如果不使用锁，那么操作方法

如下：

```csharp
//step1: 查出商品剩余量
 select quantity from items where id=100;
//step2: 如果剩余量大于0，则根据商品信息生成订单
 insert into orders(id,item_id) values(null,100);
 //step3: 修改商品的库存
 update Items set quantity=quantity-1 where id=100;
```

这样子的写法，在小作坊真的很正常，No Problems，但是在高并发环境下可能出现问题。

如下:

![1592038079311](/img/1592038079311.png)

其实在①或者②环节，已经有人下单并且减完库存了，这个时候仍然去执行step3，就造成了**超卖**。

但是使用悲观锁，就可以解决这个问题，在上面的场景中，商品信息从查询出来到修改，中间有一个生成订单的过程，使用悲观锁的原理就是，当我们在查询出items信息后就把当前的数据锁定，直到我们修改完毕后再解锁。那么在这个过程中，因为数据被锁定了，就不会出现有第三者来对其进行修改了。而这样做的前提是需要将要执行的SQL语句放在同一个事物中，否则达不到锁定数据行的目的。

如下：

```csharp
//step1: 查出商品状态
select quantity from items where id=100 for update;
//step2: 根据商品信息生成订单
insert into orders(id,item_id) values(null,100);
//step3: 修改商品的库存
update Items set quantity=quantity-2 where id=100;
```

**select...for update**是MySQL提供的实现悲观锁的方式。此时在items表中，id为100的那条数据就被我们锁定了，其它的要执行select quantity from items where id=100 for update的事务必须等本次事务提交之后才能执行。这样我们可以保证当前的数据不会被其它事务修改。

**1.2 扩展思考**

需要注意的是，当我执行select quantity from items where id=100 for update后。

如果我是在第二个事务中执行select quantity from items where id=100（不带for update）仍能正常查询出数据，不会受第一个事务的影响。

另外，MySQL还有个问题是select...for update语句执行中所有扫描过的行都会被锁上，因此**在MySQL中用悲观锁务必须确定走了索引，而不是全表扫描，否则将会将整个数据表锁住**。

悲观锁并不是适用于任何场景，它也存在一些不足，因为悲观锁大多数情况下依靠数据库的锁机制实现，以保证操作最大程度的独占性。如果加锁的时间过长，其他用户长时间无法访问，影响了程序的并发访问性，同时这样对数据库性能开销影响也很大，特别是对长事务而言，这样的开销往往无法承受，这时就需要乐观锁。

> 在此和大家分享一下，在Oracle中，也存在select ... for update，和mysql一样，但是Oracle还存在了select ... for update nowait，即发现被锁后不等待，立刻报错。

#### 2. 乐观锁

乐观锁相对悲观锁而言，它认为数据一般情况下不会造成冲突，所以在数据进行提交更新的时候，才会正式对数据的冲突与否进行检测，如果发现冲突了，则让返回错误信息，让用户决定如何去做。接下来我们看一下乐观锁在数据表和缓存中的实现。

**2.1 数据表中的实现**

利用数据版本号（**version**）机制是乐观锁最常用的一种实现方式。一般通过为数据库表增加一个数字类型的 “version” 字段，当读取数据时，将version字段的值一同读出，数据每更新一次，对此**version值+1**。当我们提交更新的时候，判断数据库表对应记录的当前版本信息与第一次取出来的version值进行比对，如果数据库表当前版本号与第一次取出来的version值相等，则予以更新，否则认为是过期数据，返回更新失败。

> 放个被用烂了的图

![1592038092276](/img/1592038092276.png)

image.png

**举个栗子🌰：**

```csharp
//step1: 查询出商品信息
select (quantity,version) from items where id=100;
//step2: 根据商品信息生成订单
insert into orders(id,item_id) values(null,100);
//step3: 修改商品的库存
update items set quantity=quantity-1,version=version+1 where id=100 and version=#{version};
```

既然可以用**version**，那还可以使用**时间戳**字段，该方法同样是在表中增加一个时间戳字段，和上面的version类似，也是在更新提交的时候检查当前数据库中数据的时间戳和自己更新前取到的时间戳进行对比，如果一致则OK，否则就是版本冲突。

> 需要注意的是，如果你的数据表是读写分离的表，当master表中写入的数据没有及时同步到slave表中时会造成更新一直失败的问题。此时，需要强制读取master表中的数据（将select语句放在事物中）。

即：**把select语句放在事务中，查询的就是master主库了！**

### 间隙锁

幻读指的是一个事务 前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行

为了解决幻读问题，InnoDB 只好引入新的锁，也就是 间隙锁 (Gap Lock),间隙锁，锁的就是两个值之间的空隙

跟间隙锁存在冲突关系的，是“往这个间隙中插入一个记录”这个操作

间隙锁和行锁合称 next-key lock，每个 next-key lock 是前开后闭区间

> 我们把间隙锁记为开区间，把 next-key lock 记为前开后闭区间

InnoDB 给每个索引加了一个不存在的最大值 suprenum，这样才符合我们前面说的“都是前开后闭区间”

间隙锁和 next-key lock 的引入，帮我们解决了幻读的问题，但同时也带来了一些“困扰”

间隙锁的引入，可能会导致同样的语句锁住更大的范围，这其实是影响了并发度的

### Mysql事务

#### 什么是事务

事务就是一组原子性的操作，这些操作要么全部发生，要么全部不发生。事务把数据库从一种一致性状态转换成另一种一致性状态。

事务具有ACID 四种特性，即原子性(atomicity)，一致性(consistency)，隔离性(isolation)，持久性(durability)：

- 原子性，指的是事务是一个不可分割的操作，要么全都正确执行，要么全都不执行。
- 一致性，指的是事务把数据库从一种一致性状态转换成另一种一致性状态，事务开始前和事务结束后，数据库的完整性约束没有被破坏。
- 隔离性，要求每个读写事务相互之间是分开的，在事务提交前对其他事务是不可见的
- 持久性，指的是事务一旦提交，其结果就是永久性的，即使宕机也能恢复。

事务有4 个隔离级别，分别是：

- 读未提交(read uncommit)
- 读已提交(read commit)
- 可重复读(repeatable read)
- 和序列化(serializable)。

> 隔离级别依次提高，分别解决了脏读、不可重读和幻读。

> 1、脏读：事务A读取了事务B更新的数据，然后B回滚操作，那么A读取到的数据是脏数据
>
> 2、不可重复读：事务 A 多次读取同一数据，事务 B 在事务A多次读取的过程中，对数据作了更新并提交，导致事务A多次读取同一数据时，结果不一致。
>
> 3、幻读：系统管理员A将数据库中所有学生的成绩从具体分数改为ABCDE等级，但是系统管理员B就在这个时候插入了一条具体分数的记录，当系统管理员A改结束后发现还有一条记录没有改过来，就好像发生了幻觉一样，这就叫幻读。
>
> 不可重复读的和幻读很容易混淆，不可重复读侧重于修改，幻读侧重于新增或删除。解决不可重复读的问题只需锁住满足条件的行，解决幻读需要锁表
>
> InnoDB 默认隔离级别为repeatable read，但是通过next-key lock 解决了幻读，保证了ACID。

#### 事务的实现原理

事务是基于重做日志文件(redo log)和回滚日志(undo log)实现的。

每提交一个事务必须先将该事务的所有日志写入到重做日志文件进行持久化，数据库就可以通过重做日志来保证事务的原子性和持久性。

每当有修改事务时，还会产生undo log，如果需要回滚，则根据undo log 的反向语句进行逻辑操作，比如insert 一条记录就delete 一条记录。

#### 使用事务应该注意的问题

- 不要再循环中使用事务（循环提交会导致大量的redo log）
- 不要使用自动提交
- 不要使用自动回滚
- 长事务切分处理
- SQL 优化

### Mysql 的隔离级别和事务实现方式

#### mysql 的隔离级别

- 读未提交是指:  一个事务还没提交时，它做的变更就能被别的事务看到。

- 读提交是指:  一个事务提交之后，它做的变更才会被其他事务看到。

- 可重复读是指:  一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据一致的。当然在可重复读隔离级别下，未提交变更对其他事务也是不可见的。

- 串行化:  顾名思义是对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当 出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行

  在实现上，数据库里面会创建一个视图，访问的时候以视图的逻辑结果为准。 

  在“可重复读”隔离级别下，这个视图是在事务启动时创建的，整个事务存在期间都用这个视图。 

  在“读提交”隔离级别下，这个视图是在每个 SQL 语句开始执行的时候创建的。

  这里需要注意的是， “读未提交”隔离级别下直接返回记录上的最新值，没有视图概念; 而“串行化”隔离级别下直接用加锁的方式来避免并行访问

#### 事务隔离的实现

在 MySQL 中，实际上每条记录在更新的时候都会同时记录一条回滚操作。记录上的最新值，通过回滚操作，都可以得到前一个状态的值

在查询这条记录的时候，不同时刻启动的事务会有不同的 read-view

系统中可以存在多个版本，就是数据库的多版本并发控制(MVCC)

**回滚日志总不能一直保留，什么时候删除呢?**

答案是，在不需要的时候才删除。也就是说，系统会判断，当没有事务再需要用到这些回滚日志时，回滚日志会被删除

**什么时候才不需要了呢?**

就是当系统里没有比这个回滚日志更早的 read-view 的时候

长事务意味着系统里面会存在很老的事务视图。由于这些事务随时可能访问数据库里面的任何数据，所以这个事务提交之前，数据库里面它可能用到的回滚记录都必须保留，这就会导致大量占用存储空间

除了对回滚段的影响，长事务还占用锁资源，也可能拖垮整个库

#### 快照读与当前读

如果是可重复读隔离级别，事务 T 启动的时候会创建一个视图 read-view，之后事务T 执行期间，即使有其他事务修改了数据，事务 T 看到的仍然跟在启动时看到的一样。

也就是说，一个在可重复读隔离级别下执行的事务，好像与世无争，不受外界影响

行锁的时候，一个事务要更新一行，如果刚好有另外一个事务拥有这一行的行锁，它又不能这么超然了，会被锁住，进入等待状态。

问题是，既然进入了等待状态，那么等到这个事务自己获取到行锁要更新数据的时候，它读到的值又是什么呢?

**快照在 MVCC 里是怎么工作的**？

在可重复读隔离级别下，事务在启动的时候就“拍了快照”。这个快照是基于整库的

InnoDB 里面每个事务有一个唯一的事务 ID，叫作 transaction id。它是在事务开始的时候向 InnoDB 的事务系统申请的，是按申请顺序严格递增的

而每行数据也都是有多个版本的。每次事务更新数据的时候，都会生成一个新的数据版本，并且把 transaction id 赋值给这个数据版本的事务 ID，记为trx_id。

同时，旧的数据版本要保留，并且在新的数据版本中，能够有信息可以直接拿到它

也就是说，数据表中的一行记录，其实可能有多个版本 (row)，每个版本有自己的 row trx_id。 每个版本不是真实存在的而是每次需要的时候根据当前版本和 undo.log 计算出来的

因此，一个事务只需要在启动的时候声明说，“以我启动的时刻为准，如果一个数据版本是在我启动之前生成的，就认;

如果是我启动以后才生成的，我就不认，我必须要找到它的上一个版本

当然，如果“上一个版本”也不可见，那就得继续往前找。还有，如果是这个事务自己更新的数据，它自己还是要认的。

在实现上， InnoDB 为每个事务构造了一个数组，用来保存这个事务启动瞬间，当前正在“活跃”的所有事务 ID。“活跃”指的就是，启动了但还没提交。

数组里面事务 ID 的最小值记为低水位，当前系统里面已经创建过的事务 ID 的最大值加 1 记为高水位。 这个视图数组和高水位，就组成了当前事务的一致性视图

而数据版本的可见性规则，就是基于数据的 row trx_id 和这个一致性视图的对比结果得到的。

> InnoDB 利用了“所有数据都有多个版本”的这个特性，实现了“秒级创建快照”的能力

**一致性读（快照读）**

事务不论在什么时候查询，看到这行数据的结果都是一致的，所以我们称之为一致性读

一个数据版本，对于一个事务视图来说，除了自己的更新总是可见以外，有三种情况:

1. 版本未提交，不可见;
2. 版本已提交，但是是在视图创建后提交的，不可见;
3. 版本已提交，而且是在视图创建前提交的，可见。

**当前读**

当事情要去更新数据的时候，就不能再在历史版本上更新了，否则其他事物的更新就丢失了

更新数据都是先读后写的，而这个读，只能读当前的值，称为“当前读”

除了 update 语句外，select 语句如果加锁，也是当前读

**可重复读**

可重复读的核心就是一致性读(consistent read);而事务更新数据的时候，只能用当前读。

如果当前的记录的行锁被其他事务占用的话，就需要进入锁等待

**读提交**

而读提交的逻辑和可重复读的逻辑类似，它们最主要的区别是: 

在可重复读隔离级别下，只需要在事务开始的时候创建一致性视图，之后事务里的其他查询都共用这个一致性视图; 

在读提交隔离级别下，每一个语句执行前都会重新算出一个新的视图

**为什么表结构不支持“可重复读”?**

这是因为表结构没有对应的行数据，也没有 row trx_id，因此只能遵循当前读的逻辑

**总结**

mysql 的事务是依靠mvcc来实现可重复的隔离级别实现了一致性读，通过间隙锁来解决当前读会产生幻读的问题。

### Mysql 主从同步过程

#### 在master机器上的操作

当master上的数据发生改变的时候，该事件(insert、update、delete)变化会按照顺序写入到binlog中。

当slave连接到master的时候，master机器会为slave开启binlog dump线程。如果读取的进度已经跟上了master，就进入睡眠状态并等待master产生新的事件。

当master 的 binlog发生变化的时候，binlog dump线程会通知slave，并将相应的binlog内容发送给slave。

#### 在slave机器上的操作

当主从同步开启的时候，slave上会创建2个线程。

- I/O线程。该线程连接到master机器，master机器上的**binlog dump线程**会将binlog的内容发送给该**I/O线程**。该**I/O线程**接收到binlog内容后，再将内容写入到本地的relay log。
- SQL线程。该线程读取I/O线程写入的relay log。并且根据relay log的内容对slave数据库做相应的操作。

### mysql 允许null 的字段设置索引哪些需要注意

mysql 进行is null 查询会使用索引的，  进行  not  null  判断则不会使用索引。

虽然MySQL可以在含有null的列上使用索引，但不代表null和其他数据在索引中是一样的。

不建议列上允许为空。最好限制not null，并设置一个默认值，比如0和''空字符串等，如果是datetime类型，可以设置成'1970-01-01 00:00:00'这样的特殊值。

对MySQL来说，null是一个特殊的值，s。比如：不能使用=,<,>这样的运算符，对null做算术运算的结果都是null，count时不会包括null行等，null比空字符串需要更多的存储空间等。

### mysql datetime  和timestamp 的区别

其中，datetime和timestamp这两种类型都是用于表示`YYYY-MM-DD HH:MM:SS` 这种年月日时分秒格式的数据，但两者还是有些许不同之处的。

1. 存储范围不同：datetime的存储范围是 `1000-01-01 00:00:00.000000` 到 `9999-12-31 23:59:59.999999`，而timestamp的范围是 `1970-01-01 00:00:01.000000` 到 `2038-01-19 03:14:07.999999`；
2. datetime存储与时区无关，而timestamp存储的是与时区有关，这也是两者最大的不同。
3. datetime适用于记录数据的创建时间，因为这个时间是不会变的。而timestamp有自动修改更新的功能，也就是说，我们对表里的其他数据进行修改，timestamp修饰的字段会自动更新为当前的时间，这个特性称为自动初始化和自动更新(Automatic Initialization and Updating)。

### mysql  exlain 各参数解释

1、id 主要是用来标识sql的执行顺序，如果没有子查询，一般来说id只有一个，执行顺序也是从上到下

2、select_type 每个select子句的类型

　　a:  simple 查询中不包含任何子查询或者union

　　b:  primary 查询中包含了任何复杂的子部分，最外层的就会变为primary

　　c:  subquery 在selecth或者where中包含了子查询

　　d:  derived  在from中包含了子查询

　　e:  union 如果第二个select 出现在union之后，则被标记为union,如果union包含在from子句的子查询中，外层select会被标记成derived

　　f：union  result 从 union表中获取结果的select

3、type 是指mysql在表中找到所需行的方式，也就是访问行的类型。从a开始效率上升

　　a （All   全表扫描)      

​	b （index 会根据索引树进行遍历）  

​	c （range 根据索引范围扫描，返回匹配值域的行）  

​	d:（ref 非唯一性索引扫描，返回匹配某个单独值的所有行。一般是指多列的唯一索引中的某一列）       

​	e （eq_ref 唯一性索引扫描。表中只有一条与之匹配） 

​	f （const、system 主要针对查询中有常量的情况，如果结果中只有一行，会变成System）  

​	g （null即不走表 也不走索引）

4、possible_keys  预估计了mysql能为当前查询选择的索引。这个字段是完全独立于执行计划中输出的表的顺序。意味着在实际查询中，可能用不到这些索引。如果该字段为null则意味着没有可使用的索引。这个时候你可以考虑为where后面的字段加上索引

5、key 这个字段表示mysql真实使用的索引。如果mysql优化过程中没有加索引，可以强制加索引

6、key_len 顾名思义就是索引长度字段。表示mysql使用的索引的长度

7、ref 这个字段一般是指一些常量用于选择过滤

8、rows  预估结果集的条数，可能不一定完全准确

9、extra   包含额外的信息。

​	a:   using filesort:       说明mysql无法利用索引进行排序，只能利用排序算法进行排序，会消耗额外的位置

​		explain select * from emp order by sal;

​	b:   using temporary:    建立临时表来保存中间结果，查询完成之后把临时表删除

​		explain select ename,count(*) from emp where deptno = 10 group by ename;

​	c:   using index:     这个表示当前的查询时覆盖索引的，直接从索引中读取数据，而不用访问数据表。如果同时出现using where 表名索引被用来执行索引键值的查找，如果没有，表面索引被用来读取数据，而不是真的查找

​		explain select deptno,count(*) from emp group by deptno limit 10;

​	d:   using where:       使用where进行条件过滤

​		explain select * from t_user where id = 1;

​	e:   using join buffer:      使用连接缓存

​	f:    impossible where：where语句的结果总是false

​		explain select * from emp where empno = 7469;


### Mysql 联合索引

#### 联合索引的最左前缀

MySQL的user表中，对a,b,c三个字段建立联合索引，那么查询时使用其中的2个作为查询条件，是否还会走索引？

根据查询字段的位置不同来决定，如查询a,     a,b    a,b,c    a,c   都可以走索引的，其他条件的查询不能走索引。

组合索引 有“最左前缀”原则。就是只从最左面的开始组合，并不是所有只要含有这三列存在的字段的查询都会用到该组合索引。

#### 联合索引的存储方式

1. 先把各个记录和页按照从左往右第一列进行排序。

2. 在记录的前一列相同的情况下，采用下一列进行排序

3. 依此类推

   通过以上操作，保证了所有索引数据是按照索引列的值从小到大的顺序排好序的。
   当我们要查找匹配索引条件时，只需从左往右依次匹配。

#### 范围查询对联合索引的影响

在组合索引中，使用between、>、<、like等进行匹配都会导致后面的列无法继续走联合索引，因为通过以上方式匹配到的数据是不可知的，后面的列无法确定根据哪些数据继续向下进行索引。

#### 在group by  和 order  by 字段上使用联合索引

创建联合索引,    c1234(c1,c2,c3,c4)

**1、只有where的情况，遵从最左原则**

条件必须有左边的字段，才会用到索引，中间如果断开了，则都不会用到后面的索引，

例子： where c1 = '1' and c2 = '1' and c4 = '1'，这种情况因为c3没有在条件中，所以只会用到c1,c2索引。

特殊情况，使用范围条件的时候，也会使用到该处的索引，但后面的索引都不会用到

例子： where c1 = '1' and c2 > '1' and c3 = '1'，这种情况从c2处已经断开，会使用到c1,c2索引，不会再使用到后面的c3,c4索引。

**2、group by和order by 其实一样，也是遵从最左原则**

可以看做继承where的条件顺序，但需要where作为基础铺垫，即没有where语句，单纯的group by | order by 也是不会使用任何索引的，并且需要和联合索引顺序一致才会使用索引。

例子：

group by c1 | order by c1，由于没有where的铺垫，不使用任何索引

where c1 = '1' group | order by c2，使用c1,c2索引

where c1 = '1' group | order by c3，只使用c1索引

where c1 > '1' group | order by c2，前面也说了，范围搜索会断掉连接，所以也只会使用c1索引

关于顺序：

where c1 = '1' group | order by c2,c3，使用c1,c2,c3索引

where c1 = '1' group | order by c3,c2，只使用c1索引

### 主键和唯一索引区别

1.主键是一种约束，唯一索引是一种索引；

2.一张表只能有一个主键，但可以创建多个唯一索引；

3.主键创建后一定包含一个唯一索引，唯一索引并一定是主键；

4.主键不能为null，唯一索引可以为null；

5.主键可以做为外键，唯一索引不行；

6.主键产生唯一的聚集索引，唯一索引产生唯一的非聚集索引

### innodb什么时候会产生回表

当索引字段为非聚簇索引，并且查询字段不是索引字段；

## Redis

### Redis 持久化机制

Redis是一个支持持久化的内存数据库，通过持久化机制把内存中的数据同步到硬盘文件来保证数据持久化。当Redis重启后通过把硬盘文件重新加载到内存，就能达到恢复数据的目的。

#### 实现方式

单独创建fork()一个子进程，将当前父进程的数据库数据复制到子进程的内存中，然后由子进程写入到临时文件中，持久化的过程结束了，再用这个临时文件替换上次的快照文件，然后子进程退出，内存释放。

#### 持久化方式 

- RDB： 是Redis默认的持久化方式,按照一定的时间周期策略把内存的数据以快照的形式保存到硬盘的二进制文件。即Snapshot快照存储，对应产生的数据文件为dump.rdb，通过配置文件中的save参数来定义快照的周期。（ 快照可以是其所表示的数据的一个副本，也可以是数据的一个复制品。）
- AOF：Redis会将每一个收到的写命令都通过Write函数追加到文件最后，类似于MySQL的binlog。当Redis重启是会通过重新执行文件中保存的写命令来在内存中重建整个数据库的内容。
  当两种方式同时开启时，数据恢复Redis会优先选择AOF恢复。

### Redis 常见线上问题

#### 缓存雪崩

**问题解释**

缓存雪崩我们可以简单的理解为：由于原有缓存失效，新缓存未到期间

例如：我们设置缓存时采用了相同的过期时间，在同一时刻出现大面积的缓存过期，所有原本应该访问缓存的请求都去查询数据库了，而对数据库CPU和内存造成巨大压力，严重的会造成数据库宕机。从而形成一系列连锁反应，造成整个系统崩溃。

**解决办法 **

大多数系统设计者考虑用加锁（ 最多的解决方案）或者队列的方式保证来保证不会有大量的线程对数据库一次性进行读写，从而避免失效时大量的并发请求落到底层存储系统上。还有一个简单方案就时讲缓存失效时间分散开。

#### 缓存穿透

**问题解释**

缓存穿透是指用户查询数据，在数据库没有，自然在缓存中也不会有。

这样就导致用户查询的时候，在缓存中找不到，每次都要去数据库再查询一遍，然后返回空（相当于进行了两次无用的查询）。

这样请求就绕过缓存直接查数据库，这也是经常提的缓存命中率问题。

**解决办法**

最常见的则是采用布隆过滤器，将所有可能存在的数据哈希到一个足够大的bitmap中，一个一定不存在的数据会被这个bitmap拦截掉，从而避免了对底层存储系统的查询压力。
另外也有一个更为简单粗暴的方法，如果一个查询返回的数据为空（不管是数据不存在，还是系统故障），我们仍然把这个空结果进行缓存，但它的过期时间会很短，最长不超过五分钟。通过这个直接设置的默认值存放到缓存，这样第二次到缓冲中获取就有值了，而不会继续访问数据库，这种办法最简单粗暴。

#### 缓存击穿

**问题解释**

指一个key非常热点，大并发集中对这个key进行访问，当这个key在失效的瞬间，仍然持续的大并发访问就穿破缓存，转而直接请求数据库。

**解决方案**

在访问key之前，采用SETNX（set if not exists）来设置另一个短期key来锁住当前key的访问，访问结束再删除该短期key。

#### 缓存预热

**问题解释**

缓存预热这个应该是一个比较常见的概念，相信很多小伙伴都应该可以很容易的理解，缓存预热就是系统上线后，将相关的缓存数据直接加载到缓存系统。

这样就可以避免在用户请求的时候，先查询数据库，然后再将数据缓存的问题！用户直接查询事先被预热的缓存数据！

**解决思路**
1、直接写个缓存刷新页面，上线时手工操作下；
2、数据量不大，可以在项目启动的时候自动进行加载；
3、定时刷新缓存；

#### 缓存更新

除了缓存服务器自带的缓存失效策略之外（Redis默认的有6种策略可供选择），我们还可以根据具体的业务需求进行自定义的缓存淘汰，常见的策略有两种：
（1）定时去清理过期的缓存；
（2）当有用户请求过来时，再判断这个请求所用到的缓存是否过期，过期的话就去底层系统得到新数据并更新缓存。
两者各有优劣，第一种的缺点是维护大量缓存的key是比较麻烦的，第二种的缺点就是每次用户请求过来都要判断缓存失效，逻辑相对比较复杂！具体用哪种方案，大家可以根据自己的应用场景来权衡。

#### 缓存降级

当访问量剧增、服务出现问题（如响应时间慢或不响应）或非核心服务影响到核心流程的性能时，仍然需要保证服务还是可用的，即使是有损服务。

系统可以根据一些关键数据进行自动降级，也可以配置开关实现人工降级。

降级的最终目的是保证核心服务可用，即使是有损的。而且有些服务是无法降级的（如加入购物车、结算）。

以参考日志级别设置预案：
（1）一般：比如有些服务偶尔因为网络抖动或者服务正在上线而超时，可以自动降级；
（2）警告：有些服务在一段时间内成功率有波动（如在95~100%之间），可以自动降级或人工降级，并发送告警；
（3）错误：比如可用率低于90%，或者数据库连接池被打爆了，或者访问量突然猛增到系统能承受的最大阀值，此时可以根据情况自动降级或者人工降级；
（4）严重错误：比如因为特殊原因数据错误了，此时需要紧急人工降级。

服务降级的目的，是为了防止Redis服务故障，导致数据库跟着一起发生雪崩问题。因此，对于不重要的缓存数据，可以采取服务降级策略，例如一个比较常见的做法就是，Redis出现问题，不去数据库查询，而是直接返回默认值给用户。

### 单线程redis

#### 单线程的redis为什么这么快

1. 纯内存操作
2. 单线程操作，避免了频繁的上下文切换
3. 采用了非阻塞I/O多路复用机制

#### Redis 为什么是单线程的

官方FAQ表示，因为Redis是基于内存的操作，CPU不是Redis的瓶颈，Redis的瓶颈最有可能是机器内存的大小或者网络带宽。

既然单线程容易实现，而且CPU不会成为瓶颈，那就顺理成章地采用单线程的方案了（毕竟采用多线程会有很多麻烦！）Redis利用队列技术将并发访问变为串行访问
1）绝大部分请求是纯粹的内存操作（非常快速）

2）采用单线程,避免了不必要的上下文切换和竞争条件

3）非阻塞IO优点采用了非阻塞I/O多路复用机制

### redis的数据类型

1. String
   这个其实没啥好说的，最常规的set/get操作，value可以是String也可以是数字。一般做一些复杂的计数功能的缓存。

2. hash
   这里value存放的是结构化的对象，比较方便的就是操作其中的某个字段。博主在做单点登录的时候，就是用这种数据结构存储用户信息，以cookieId作为key，设置30分钟为缓存过期时间，能很好的模拟出类似session的效果。

3. list
   使用List的数据结构，可以做简单的消息队列的功能。另外还有一个就是，可以利用lrange命令，做基于redis的分页功能，性能极佳，用户体验好。本人还用一个场景，很合适—取行情信息。就也是个生产者和消费者的场景。LIST可以很好的完成排队，先进先出的原则。

4. set
   因为set堆放的是一堆不重复值的集合。所以可以做全局去重的功能。为什么不用JVM自带的Set进行去重？因为我们的系统一般都是集群部署，使用JVM自带的Set，比较麻烦，难道为了一个做一个全局去重，再起一个公共服务，太麻烦了。
   另外，就是利用交集、并集、差集等操作，可以计算共同喜好，全部的喜好，自己独有的喜好等功能。

5. sorted set
   sorted set多了一个权重参数score,集合中的元素能够按score进行排列。可以做排行榜应用，取TOP N操作。

6. bit arrays   

   简单的位映射

7. hyperloglogs

   概率数据结构

### Redis 内部结构

- dict：  本质上是为了解决算法中的查找问题（Searching）是一个用于维护key和value映射关系的数据结构，与很多语言中的Map或dictionary类似。 本质上是为了解决算法中的查找问题（Searching）
- sds：  sds就等同于char * 它可以存储任意二进制数据，不能像C语言字符串那样以字符’\0’来标识字符串的结 束，因此它必然有个长度字段。
- skiplist： （跳跃表） 跳表是一种实现起来很简单，单层多指针的链表，它查找效率很高，堪比优化过的二叉平衡树，且比平衡树的实现，
- quicklist
- ziplist:        压缩表 ziplist是一个编码后的列表，是由一系列特殊编码的连续内存块组成的顺序型数据结构，

### Redis Zset 数据结构

 zset底层的存储结构包括ziplist或skiplist，在同时满足以下两个条件的时候使用ziplist，其他时候使用skiplist，两个条件如下：

- 有序集合保存的元素数量小于128个
- 有序集合保存的所有元素的长度小于64字节

### redis 的pipeline

#### pipeline出现的背景：

redis客户端执行一条命令分4个过程：

  发送命令－〉命令排队－〉命令执行－〉返回结果

> 这个过程称为Round trip time(简称RTT, 往返时间)，mget mset有效节约了RTT，但大部分命令（如hgetall，并没有mhgetall）不支持批量操作，需要消耗N次RTT ，这个时候需要pipeline来解决这个问题

#### pepeline的性能

1、未使用pipeline执行N条命令

![1592038125957](/img/1592038125957.png)

2、使用了pipeline执行N条命令

![1592038134671](/img/1592038134671.png)

#### 原生批操作和pipeline对比

1、原生批命令是原子性，pipeline是非原子性
(原子性概念:一个事务是一个不可分割的最小工作单位,要么都成功要么都失败。原子操作是指你的一个业务逻辑必须是不可拆分的. 处理一件事情要么都成功，要么都失败，原子不可拆分)

2、原生批命令一命令多个key, 但pipeline支持多命令（存在事务），非原子性
3、原生批命令是服务端实现，而pipeline需要服务端与客户端共同完成

#### redis事务

pipeline是多条命令的组合，为了保证它的原子性，redis提供了简单的事务。

**1、redis的简单事务**

一组需要一起执行的命令放到multi和exec两个命令之间，其中multi代表事务开始，exec代表事务结束。
![1592038145769](/img/1592038145769.png)

**2、停止事务discard**

![1592038156883](/img/1592038156883.png)



### redis的过期策略、内存淘汰机制

#### redis 的过期策略

redis采用的是定期删除+惰性删除策略。

**为什么不用定时删除策略?**

定时删除,用一个定时器来负责监视key,过期则自动删除。虽然内存及时释放，但是十分消耗CPU资源。

在大并发请求下，CPU要将时间应用在处理请求，而不是删除key,因此没有采用这一策略.

**定期删除+惰性删除是如何工作的呢?**

定期删除，redis默认每个100ms检查，是否有过期的key,有过期key则删除。

需要说明的是，redis不是每个100ms将所有的key检查一次，而是随机抽取进行检查(如果每隔100ms,全部key进行检查，redis岂不是卡死)。

因此，如果只采用定期删除策略，会导致很多key到时间没有删除。

于是，惰性删除派上用场。也就是说在你获取某个key的时候，redis会检查一下，这个key如果设置了过期时间那么是否过期了？如果过期了此时就会删除。

#### redis 内存淘汰机制

**采用定期删除+惰性删除就没其他问题了么?**

不是的，如果定期删除没删除key。然后你也没即时去请求key，也就是说惰性删除也没生效。这样，redis的内存会越来越高。那么就应该采用内存淘汰机制。

在redis.conf中有一行配置

maxmemory-policy volatile-lru

该配置就是配内存淘汰策略的:

- volatile-lru：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰

- volatile-ttl：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰

- volatile-random：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰

- allkeys-lru：从数据集（server.db[i].dict）中挑选最近最少使用的数据淘汰

- allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰

- no-enviction（驱逐）：禁止驱逐数据，新写入操作会报错

  > ps：如果没有设置 expire 的key, 不满足先决条件(prerequisites); 那么 volatile-lru, volatile-random 和 volatile-ttl 策略的行为, 和 noeviction(不删除) 基本上一致。

### Redis 优点

1. 速度快，因为数据存在内存中，类似于HashMap，HashMap的优势就是查找和操作的时间复杂度都是O(1)
2. 支持丰富数据类型，支持string，list，set，sorted set，hash
3. 支持事务，操作都是原子性，所谓的原子性就是对数据的更改要么全部执行，要么全部不执行
4. 丰富的特性：可用于缓存，消息，按key设置过期时间，过期后将会自动删除

### Redis key的资源竞争问题

#### **同时有多个子系统去set一个key。这个时候要注意什么呢**？ 

不推荐使用redis的事务机制。因为我们的生产环境，基本都是redis集群环境，做了数据分片操作。

你一个事务中有涉及到多个key操作的时候，这多个key不一定都存储在同一个redis-server上。因此，redis的事务机制，十分鸡肋。

**解决办法**

(1)如果对这个key操作，不要求顺序： 准备一个分布式锁，大家去抢锁，抢到锁就做set操作即可
(2)如果对这个key操作，要求顺序： 分布式锁+时间戳。 假设这会系统B先抢到锁，将key1设置为{valueB 3:05}。接下来系统A抢到锁，发现自己的valueA的时间戳早于缓存中的时间戳，那就不做set操作了。以此类推。
(3) 利用队列，将set方法变成串行访问也可以

#### redis遇到高并发，如果保证读写key的一致性

对redis的操作都是具有原子性的,是线程安全的操作,你不用考虑并发问题,redis内部已经帮你处理好并发的问题了。

### Redis 常见性能问题和解决方案？

(1) Master 最好不要做任何持久化工作，如 RDB 内存快照和 AOF 日志文件
(2) 如果数据比较重要，某个 Slave 开启 AOF 备份数据，策略设置为每秒同步一次
(3) 为了主从复制的速度和连接的稳定性， Master 和 Slave 最好在同一个局域网内
(4) 尽量避免在压力很大的主库上增加从库
(5) 主从复制不要用图状结构，用单向链表结构更为稳定，即： Master <- Slave1 <- Slave2 <- Slave3…

### Redis 原子性

对于Redis而言，命令的原子性指的是：一个操作的不可以再分，操作要么执行，要么不执行。

Redis的操作之所以是原子性的，是因为Redis是单线程的。

Redis本身提供的所有API都是原子操作，Redis中的事务其实是要保证批量操作的原子性。

**多个命令在并发中也是原子性的吗？**

不一定， 将get和set改成单命令操作，incr 。使用Redis的事务，或者使用Redis+Lua==的方式实现.

### Redis实现分布式锁

Redis为单进程单线程模式，采用队列模式将并发访问变成串行访问，且多客户端对Redis的连接并不存在竞争关系Redis中可以使用SETNX命令实现分布式锁。
将 key 的值设为 value ，当且仅当 key 不存在。 若给定的 key 已经存在，则 SETNX 不做任何动作

解锁：使用 del key 命令就能释放锁
解决死锁：
1）通过Redis中expire()给锁设定最大持有时间，如果超过，则Redis来帮我们释放锁。
2 )  使用 setnx key “当前系统时间+锁持有的时间”和getset key “当前系统时间+锁持有的时间”组合的命令就可以实现。

### Redis主从复制

#### 全量同步

Redis全量复制一般发生在Slave初始化阶段，这时Slave需要将Master上的所有数据都复制一份。具体步骤如下： 

- 从服务器连接主服务器，发送SYNC命令； 
- 主服务器接收到SYNC命名后，开始执行BGSAVE命令生成RDB文件并使用缓冲区记录此后执行的所有写命令； 
- 主服务器BGSAVE执行完后，向所有从服务器发送快照文件，并在发送期间继续记录被执行的写命令； 
- 从服务器收到快照文件后丢弃所有旧数据，载入收到的快照； 
- 主服务器快照发送完毕后开始向从服务器发送缓冲区中的写命令； 
- 从服务器完成对快照的载入，开始接收命令请求，并执行来自主服务器缓冲区的写命令；

完成上面几个步骤后就完成了从服务器数据初始化的所有操作，从服务器此时可以接收来自用户的读请求。

#### 增量同步

Redis增量复制是指Slave初始化后开始正常工作时主服务器发生的写操作同步到从服务器的过程。 
增量复制的过程主要是主服务器每执行一个写命令就会向从服务器发送相同的写命令，从服务器接收并执行收到的写命令。

#### Redis主从同步策略

主从刚刚连接的时候，进行全量同步；全同步结束后，进行增量同步。当然，如果有需要，slave 在任何时候都可以发起全量同步。redis 策略是，无论如何，首先会尝试进行增量同步，如不成功，要求从机进行全量同步。

## HDFS

### HDFS 工作机制介绍

1. HDFS集群分为两大角色：NameNode、DataNode  (Secondary Namenode)
2. NameNode负责管理整个文件系统的元数据
3. DataNode 负责管理用户的文件数据块
4. 文件会按照固定的大小（blocksize）切成若干块后分布式存储在若干台datanode上
5. 每一个文件块可以有多个副本，并存放在不同的datanode上
6. Datanode会定期向Namenode汇报自身所保存的文件block信息，而namenode则会负责保持文件的副本数量
7. HDFS的内部工作机制对客户端保持透明，客户端请求访问HDFS都是通过向namenode申请来进行

### HDFS 写数据流程

![1587966152835](/img/1587966152835.png)

1. 客户端与namenode通信请求上传文件，namenode检查目标文件是否已存在，父目录是否存在
2. namenode返回是否可以上传
3. client请求第一个 block该传输到哪些datanode服务器上
4. namenode返回可以上传的节点, 示例3个datanode服务器ABC
5. client请求3台dn中的一台A上传数据（本质上是一个RPC调用，建立pipeline），A收到请求会继续调用B，然后B调用C，将整个pipeline建立完成，逐级返回客户端
6. client开始往A上传第一个block（先从磁盘读取数据放到一个本地内存缓存），以packet为单位(chunk为校验单位)，A收到一个packet就会传给B，B传给C；A每传一个packet会放入一个应答队列等待应答
7. 当一个block传输完成之后(只要有一个节点上传成功，就算成功)，client再次请求namenode上传第二个block的服务器。

### HDFS 读数据流程

![1587966235182](/img/1587966235182.png)

1. client跟namenode通信查询元数据，找到文件块所在的datanode服务器
2. cilent挑选一台datanode（就近原则，然后随机）服务器，请求建立socket流
3. datanode开始发送数据（从磁盘里面读取数据放入流，以packet为传输单位，chunk为校验单位）
4. 客户端以packet为单位接收，先在本地缓存，然后写入目标文件

### Namenode 工作内容

**职责**

- 负责客户端请求的响应
- 元数据的管理（查询，修改）

**元数据管理**

- 内存元数据(NameSystem)
- 磁盘元数据镜像文件(fsimage)
- 数据操作日志文件(edits文件，可以通过日志运算出元数据)

**元数据存储机制**

1. 内存中有一份完整的原数据(内存metadate)
2. 磁盘中有一个"准完整"的原数据镜像(fsimage)文件(在namenode的工作目录中)
3. 用于衔接metadata和持久化元数据镜像的fsimage之间的操作日志(edits文件), 当客户端对hdfs中的文件进行新增或者修改操作，操作记录会首先被记录到edits日志文件中，当客户端操作成功后，相应的原数据会更新到内存meta.data中， 并且每隔一定的间隔hdfs会将当前的metadata同步到fsimage镜像文件中

### 元数据checkpoint机制

由于在数据备份的时候会占用计算资源，所以为了减轻namenode的负载，通常可以将数据备份的工作交给另外一个专门用来做数据备份的namenode--> sencondary namenode

每隔一段时间，会由secondary namenode 将namenode上积累的所有edits和一个最新的fsimage下载到本地(只有第一次merge才会下载fsimage)，并加载到内存进行merge(这个过程称之为checkpoint)

![1587967356303](/img/1587967356303.png)

### namenode的一些情况

**namenode如果宕机，hdfs是否还能正常提供服务**

不能，secondarynamenode虽然有元数据信息，但是不能更新元数据， 不能充当namenode使用

**如果namenode的硬盘损坏，元数据是否能回复，能恢复多少?**

可以恢复最后一次merge之前的数据， 只需要将secondarynamenode的数据目录替换成namenode的数据目录

**配置namenode的工作目录时，有哪些可以注意的事项**

可以将namenode的元数据保存到多块物理磁盘上例如如下的namenode配置

### 元数据目录文件介绍

#### VEERSION文件

VERSION文件是Java属性文件，内容大致如下：

```shell
#Fri Nov 15 19:47:46 CST 2013
namespaceID=934548976
clusterID=CID-cdff7d73-93cd-4783-9399-0a22e6dce196
cTime=0
storageType=NAME_NODE
blockpoolID=BP-893790215-192.168.24.72-1383809616115
layoutVersion=-47
```

1. namespaceID是文件系统的唯一标识符，在文件系统首次格式化之后生成的；

2. storageType说明这个文件存储的是什么进程的数据结构信息（如果是DataNode，storageType=DATA_NODE）；

3. cTime 属性标记了 namenode 存储系统的创建时间（即FSImage对象），对于刚刚格式化的存储系统，这个属性为 0； 但是在文件系统升级之后，该值会更新到新的时间戳。

4. layoutVersion：表示HDFS永久性数据结构的版本信息， 只要数据结构变更，版本号也要递减，此时的HDFS也需要升级，否则磁盘仍旧是使用旧版本的数据结构，这会导致新版本的NameNode无法使用

5. clusterID是系统生成或手动指定的集群ID，在-clusterid选项中可以使用它；如下说明

   ```shell
   # 使用如下命令格式化一个Namenode：选择一个唯一的cluster_id，并且这个cluster_id不能与环境中其他集群有冲突。如果没有提供cluster_id，则会自动生成一个唯一的ClusterID。
   hadoop namenode -format -clusterId <cluster_id>
   
   # 升级集群至最新版本。在升级过程中需要提供一个ClusterID，如果没有提供ClusterID，则会自动生成一个ClusterID。
   hadoop start namenode --config $HADOOP_CONF_DIR  -upgrade -clusterId <cluster_ID>
   ```

6. blockpoolID是针对每一个Namespace所对应的blockpool的ID，上面的这个BP-893790215-192.168.24.72-1383809616115就是在我的ns1的namespace下的存储块池的ID，这个ID包括了其对应的NameNode节点的ip地址。

**seen_txid文件**

是存放transactionId的文件，format之后是0，它代表的是namenode里面的edits_*文件的尾数，namenode重启的时候，会按照seen_txid的数字，循序从头跑edits_0000001~到seen_txid的数字。所以当你的hdfs发生异常重启的时候，一定要比对seen_txid内的数字是不是你edits最后的尾数，不然会发生建置namenode时metaData的资料有缺少，导致误删Datanode上多余Block的资讯。

文件中记录的是edits滚动的序号，每次重启namenode时，namenode就知道要将哪些edits进行加载。

**fsimage文件和edits文件**

fsimage: 元数据的镜像文件

edits: 元数据的滚动日志文件，每次merge之后会对之前的日志文件进行清除

### MAPREDUCE 的运行实例

1. MRAppMaster: 负责整个程序的过程调度以及状态协调
2. mapTask: 负责map阶段的整个数据处理流程
3. ReduceTask: 负责reduce阶段的整个数据处理流程

### YARN的角色

1. yarn中的主管角色叫ResourceManager
2. yarn中具体提供运算资源的角色叫NodeManager

### MR运行任务的流程

![1587988401191](/img/1587988401191.png)

![1587988654365](/img/1587988654365.png)

1. 一个mr程序启动的时候，最先启动的是MRAppMaster，MRAppMaster启动后根据本次job的描述信息，计算出需要的maptask实例数量，然后向集群申请机器启动相应数量的maptask进程

2. maptask进程启动之后，根据给定的数据切片范围进行数据处理，主体流程为：

   a. 利用客户指定的inputformat来获取RecordReader读取数据，形成输入KV对

   b. 将输入KV对传递给客户定义的map()方法，做逻辑运算，并将map()方法输出的KV对收集到缓存

   c. 将缓存中的KV对按照K分区排序后不断溢写到磁盘文件

3. MRAppMaster监控到所有maptask进程任务完成之后，会根据客户指定的参数启动相应数量的reducetask进程，并告知reducetask进程要处理的数据范围（数据分区）

4. Reducetask进程启动之后，根据MRAppMaster告知的待处理数据所在位置，从若干台maptask运行所在机器上获取到若干个maptask输出结果文件，并在本地进行重新归并排序，然后按照相同key的KV为一个组，调用客户定义的reduce()方法进行逻辑运算，并收集运算输出的结果KV，然后调用客户指定的outputformat将结果数据输出到外部存储

### MapReduce运行原理图

![1587991593319](/img/1587991593319.png)

### MAPREDUCE的shuffle机制

#### shuffle机制概述

- mapreduce中，map阶段处理的数据如何传递给reduce阶段，是mapreduce框架中最关键的一个流程，这个流程就叫shuffle(如上图绿色框部分)；
- shuffle: 洗牌、发牌 ——（核心机制：数据分区，排序，缓存）；
- 具体来说：就是将maptask输出的处理结果数据，分发给reducetask，并在分发的过程中，对数据按key进行了分区、合并和排序；

#### shuffle主要流程

![1578902472291](/img/1578902472291.png)

shuffle是MR处理流程中的一个过程，它的每一个处理步骤是分散在各个mapTask和reduceTask几点上完成的，主要有3个动作:

1. 将结果分区
2. 根据key进行排序
3. 使用Combiner进行局部排序

在整个流程中有几个关键的步骤:

1. 对缓冲区输出的结果进行分区和排序， 数据的分区和排序最开始行程的位置
2. 对碎片数据的合并，包含第一次从缓冲区经过排序分区的结果，以及每个taskMap输出的结果，并且都会经过combine整合
3. 对不同mapTask的分区结果进行整合

#### shuffle的详细流程

1. maptask收集我们的map()方法输出的kv对，放到内存缓冲区中
2. 从内存缓冲区不断溢出本地磁盘文件，可能会溢出多个文件
3. 多个溢出文件会被合并成大的溢出文件
4. 在溢出过程中，及合并的过程中，都要调用partitoner进行分组和针对key进行排序
5. reducetask根据自己的分区号，去各个maptask机器上取相应的结果分区数据
6. reducetask会取到同一个分区的来自不同maptask的结果文件，reducetask会将这些文件再进行合并（归并排序）
7. 合并成大文件后，shuffle的过程也就结束了，后面进入reducetask的逻辑运算过程（从文件中取出一个一个的键值对group，调用用户自定义的reduce()方法） 

> Shuffle中的缓冲区大小会影响到mapreduce程序的执行效率，原则上说，缓冲区越大，磁盘io的次数越少，执行速度就越快 
>
> 缓冲区的大小可以通过参数调整,  参数：io.sort.mb  默认100M

### Mapreduce 一些问题

1. 自定义分区:  自定义Partitioner类(继承partitioner)
2. 对map的中间结果进行合并: 自定义Combine(继承Reducer)类
3. 对自定义Bean进行分组: 自定义GroupingComparator继承(WritableComparator), 实现Compare方法
4. 实现灵活的文件读取和文件输出，通过自定义inputformat和outputformat
5. 全局计数，通过context.getCounter()

## 排序

### 归并排序

```python
        def merge_sort(li):
                        # 当列表的长度为1时，不需要再拆分
                        if len(li) == 1:
                                return li
                        # 取一个中间位置，把数据拆分成左右两半
                        mid = len(li)//2
                        left = li[:mid]
                        right = li[mid:]

                        # 递归继续拆分
                        left_res = merge_sort(left)
                        right_res = merge_sort(right)
                        # 合并下层方法返回的结果，并且进行排序
                        res = merge(left_res,right_res)
                        return res
                def merge(left,right):
                        """把两个有序的序列组成有序序列"""
                        # 定义两个下标，分别指向两个序列的0位置
                        left_index = 0
                        right_index = 0
                        # 创建临时列表
                        res = []
                        while left_index < len(left) and right_index < len(right):
                                if left[left_index] <= right[right_index]:
                                        res.append(left[left_index])
                                        left_index += 1
                                else:
                                        res.append(right[right_index])
                                        right_index += 1
                        res = res + left[left_index:]
                        res = res + right[right_index:]
                        return res
```

### 快速排序

```python
 def quick_sort(li,start,end):
            # 当start=end时，证明只对一个数排序，不需要排序
                        # 当start>end时，证明没有需要排序的数据
            if start >= end:
                return
            # 定义两个下标，一个是头,一个是尾
            left = start
            right = end
            # 把第一个数当作中间值
            mid = li[left]
            # 不停的让右边下标往左移动，左边下标往右移动，直到左右下标相等
            while left < right:
               # 右边下标往左移动，找到一个小于mid的数据
                while left < right and li[right] >= mid:
                    right -= 1
                # 循环结束时，right指向的数据小于mid，把数据放到左边下标位置
                li[left] = li[right]
                # 左边下标往右移动，找到一个大于mid的数据
                while left < right and li[left] <= mid:
                    left += 1
                # 循环结束时，left指向的数据大于mid，把数据放到右边下标位置
                li[right] = li[left]
            # 循环结束时，left=right，把mid放到此位置
            li[left] = mid
            # 递归让左边的数据继续快排
            quick_sort(li,start,left-1)
            # 递归让右边的数据继续快排
            quick_sort(li,left+1,end)
if __name__ == '__main__':
    l = [7,6,5,4,3,2]
    quick_sort(l,0,len(l)-1)
    print(l)
```

## 其他

### docker 容器和虚拟机隔离级别的区别

- 从两者的架构图上看，虚拟机是在硬件级别进行虚拟化，模拟硬件搭建操作系统；而Docker是在操作系统的层面虚拟化，复用操作系统，运行Docker容器。
- Docker的速度很快，秒级，而虚拟机的速度通常要按分钟计算。
- Docker所用的资源更少，性能更高。同样一个物理机器，Docker运行的镜像数量远多于虚拟机的数量。
- 虚拟机实现了操作系统之间的隔离，Docker算是进程之间的隔离，虚拟机隔离级别更高、安全性方面也更强。
- 虚拟机和Docker各有优势，不存在谁替代掉谁的问题，很多企业都采用物理机上做虚拟机，虚拟机中跑Docker的方式。

### http缓存机制

#### http缓存分类

- 数据库缓存
- 服务器端缓存（代理服务器缓存、CDN 缓存）
- 浏览器缓存 （ HTTP 缓存、indexDB、cookie、localstorage 等等）

#### 缓存相关术语

- 缓存命中率：从缓存中得到数据的请求数与所有请求数的比率。理想状态是越高越好。
- 过期内容：超过设置的有效时间，被标记为“陈旧”的内容。通常过期内容不能用于回复客户端的请求，必须重新向源服务器请求新的内容或者验证缓存的内容是否仍然准备。
- 验证：验证缓存中的过期内容是否仍然有效，验证通过的话刷新过期时间。
- 失效：失效就是把内容从缓存中移除。当内容发生改变时就必须移除失效的内容。
- 浏览器缓存： 是 HTTP 协议定义的缓存机制。HTML meta 标签，例如  Pragma:  no-store； 含义是让浏览器不缓存当前页面。但是代理服务器不解析 HTML 内容，一般应用广泛的是用 HTTP 头信息控制缓存。

#### 浏览器缓存分类

浏览器缓存主要有两类：缓存协商和彻底缓存，也有称之为协商缓存和强缓存。

1.强缓存：不会向服务器发送请求，直接从缓存中读取资源，在chrome控制台的network选项中可以看到该请求返回200的状态码;

2.协商缓存：向服务器发送请求，服务器会根据这个请求的request header的一些参数来判断是否命中协商缓存，如果命中，则返回304状态码并带上新的response header通知浏览器从缓存中读取资源；

两者的共同点是，都是从客户端缓存中读取资源；区别是强缓存不会发请求，协商缓存会发请求。

#### 浏览器使用缓存加载页面流程

浏览器缓存分为强缓存和协商缓存，浏览器加载一个页面的简单流程如下：

1. 浏览器先根据这个资源的http头信息来判断是否命中强缓存。如果命中则直接加在缓存中的资源，并不会将请求发送到服务器。
2. 如果未命中强缓存，则浏览器会将资源加载请求发送到服务器。服务器来判断浏览器本地缓存是否失效。若可以使用，则服务器并不会返回资源信息，浏览器继续从缓存加载资源。
3. 如果未命中协商缓存，则服务器会将完整的资源返回给浏览器，浏览器加载新资源，并更新缓存。