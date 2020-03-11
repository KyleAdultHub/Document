---
title: flume
date: "2020-02-13 23:00:00"
categories:
- 大数据
- 大数据集群工具
tags:
- 大数据
toc: true
typora-root-url: ..\..\..
---

## Flume介绍

### 概述

- Flume是一个分布式、可靠、和高可用的海量日志采集、聚合和传输的系统。
- Flume可以采集文件，socket数据包等各种形式源数据，又可以将采集到的数据输出到HDFS、hbase、hive、kafka等众多外部存储系统中
- 一般的采集需求，通过对flume的简单配置即可实现
- Flume针对特殊场景也具备良好的自定义扩展能力，因此，flume可以适用于大部分的日常数据采集场景

### 运行机制

- Flume分布式系统中最**核心的角色是agent**，flume采集系统就是由一个个agent所连接起来形成
- 每一个agent相当于一个数据传递员，内部有三个组件：
  1. Source：采集源，用于跟数据源对接，以获取数据
  2. Sink：下沉地，采集数据的传送目的，用于往下一级agent传递数据或者往最终存储系统传递数据
  3. Channel：angent内部的数据传输通道，用于从source将数据传递到sink

- **一个完整的工作流程**：source不断的接收数据，将数据封装成一个一个的event，然后将event发送给channel，chanel作为一个缓冲区会临时存放这些event数据，随后sink会将channel中的event数据发送到指定的地方—-例如HDFS等。

### Flume采集系统结构图

#### 简单的结构

单个agent采集数据

![1581651251714](/img/1581651251714.png)

#### 复杂结构

多级agent之间串联

![1581651276690](/img/1581651276690.png)

## Flume安装

### 安装以及启动步骤

1. 下载二进制包

   apache-flume-1.6.0-bin.tar.gz

2. 上传到数据源所在节点

3. 解压二进制包

   tar -zxvf apache-flume-1.6.0-bin.tar.gz

4. 修改配置文件

   进入到flume目录， 修改conf下的flume-env.sh, 在里面配置JAVA_HOME

5. 配置采集方案

   根据数据采集的需求配置采集方案，描述在配置文件中(文件名可以任意自定义)

6. 启动flume agent

   指定采集方案配置文件, 在相应的节点上启动flume agent

### 采集方案配置示例

1. 创建采集方案

   现在flume的conf目录下新建一个文件， 以netcat-logger.conf 为例(netcat 为一种socket数据源)

   ```ini
   # 定义这个agent中各组件的名字
   a1.sources = r1
   a1.sinks = k1
   a1.channels = c1
   
   # 描述和配置source组件：r1
   a1.sources.r1.type = netcat
   a1.sources.r1.bind = localhost
   a1.sources.r1.port = 44444
   
   # 描述和配置sink组件：k1
   a1.sinks.k1.type = logger
   
   # 描述和配置channel组件，此处使用是内存缓存的方式
   a1.channels.c1.type = memory
   a1.channels.c1.capacity = 1000
   a1.channels.c1.transactionCapacity = 100
   
   # 描述和配置source  channel   sink之间的连接关系
   a1.sources.r1.channels = c1
   a1.sinks.k1.channel = c1
   ```

2. 启动agent去采集数据

   ```shell
   bin/flume-ng agent -c conf -f conf/netcat-logger.conf -n a1  -Dflume.root.logger=INFO,console
   ```

   备注:

   ​	-c conf   指定flume自身的配置文件所在目录

   ​	-f conf/netcat-logger.con  指定我们所描述的采集方案

   ​	-n a1  指定我们这个agent的名字

3. 测试数据采集效果

   先要往agent采集监听的端口上发送数据， 让agent有数据可以采集

   ```shell
   telnet localhost 44444
   ```

   连接后，发送数据，查看flume的logger界面显示内容

## Flume采集案例

### 目录到HDFS

**采集需求**

某服务器的某特定目录下，会不断产生新的文件，每当有新文件出现，就需要把文件采集到HDFS中去

根据需求，首先定义以下3大要素：

- 采集源，即source——监控文件目录 :  spooldir
- 下沉目标，即sink——HDFS文件系统  :  hdfs sink
- source和sink之间的传递通道——channel，可用file channel 也可以用内存channel

**采集方案编写**

```ini
#定义三大组件的名称
agent1.sources = source1
agent1.sinks = sink1
agent1.channels = channel1

# 配置source组件
agent1.sources.source1.type = spooldir
agent1.sources.source1.spoolDir = /home/hadoop/logs/
agent1.sources.source1.fileHeader = false

#配置拦截器
agent1.sources.source1.interceptors = i1
agent1.sources.source1.interceptors.i1.type = host
agent1.sources.source1.interceptors.i1.hostHeader = hostname

# 配置sink组件
agent1.sinks.sink1.type = hdfs
agent1.sinks.sink1.hdfs.path =hdfs://hdp-node-01:9000/weblog/flume-collection/%y-%m-%d/%H-%M
agent1.sinks.sink1.hdfs.filePrefix = access_log
agent1.sinks.sink1.hdfs.maxOpenFiles = 5000
agent1.sinks.sink1.hdfs.batchSize= 100
agent1.sinks.sink1.hdfs.fileType = DataStream
agent1.sinks.sink1.hdfs.writeFormat =Text
agent1.sinks.sink1.hdfs.rollSize = 102400
agent1.sinks.sink1.hdfs.rollCount = 1000000
agent1.sinks.sink1.hdfs.rollInterval = 60
#agent1.sinks.sink1.hdfs.round = true
#agent1.sinks.sink1.hdfs.roundValue = 10
#agent1.sinks.sink1.hdfs.roundUnit = minute
agent1.sinks.sink1.hdfs.useLocalTimeStamp = true
# Use a channel which buffers events in memory
agent1.channels.channel1.type = memory
agent1.channels.channel1.keep-alive = 120
agent1.channels.channel1.capacity = 500000
agent1.channels.channel1.transactionCapacity = 600

# Bind the source and sink to the channel
agent1.sources.source1.channels = channel1
agent1.sinks.sink1.channel = channel1
```

Channel参数解释：

- capacity：默认该通道中最大的可以存储的event数量
- trasactionCapacity：每次最大可以从source中拿到或者送到sink中的event数量
- keep-alive：event添加到通道中或者移出的允许时间

HDFS SINK参数解释:

- path：hdfs的路径，需要包含文件系统标识，比如：hdfs://namenode/flume/webdata/

- filePrefix：默认值：FlumeData，写入hdfs的文件名前缀

- fileSuffix：写入 hdfs 的文件名后缀，比如：.lzo .log等。

- inUsePrefix：临时文件的文件名前缀，hdfs sink 会先往目标目录中写临时文件，再根据相关规则重命名成最终目标文件；

- inUseSuffix：默认值：.tmp，临时文件的文件名后缀。

- rollInterval：默认值：30：hdfs sink 间隔多长将临时文件滚动成最终目标文件，单位：秒;  如果设置成0，则表示不根据时间来滚动文件； 注：滚动（roll）指的是，hdfs sink 将临时文件重命名成最终目标文件，并新打开一个临时文件来写入数据；

- rollSize：默认值：1024：当临时文件达到多少（单位：bytes）时，滚动成目标文件；如果设置成0，则表示不根据临时文件大小来滚动文件；

- rollCount：默认值：10：当 events 数据达到该数量时候，将临时文件滚动成目标文件；如果设置成0，则表示不根据events数据来滚动文件；

- idleTimeout：默认值：0：当目前被打开的临时文件在该参数指定的时间（秒）内，没有任何数据写入，则将该临时文件关闭并重命名成目标文件；

- batchSize：默认值：100：每个批次刷新到 HDFS 上的 events 数量；

- codeC：文件压缩格式，包括：gzip, bzip2, lzo, lzop, snappy

- fileType：默认值：SequenceFile，文件格式，包括：SequenceFile, DataStream,CompressedStream

​                                   当使用DataStream时候，文件不会被压缩，不需要设置hdfs.codeC;

​                                   当使用CompressedStream时候，必须设置一个正确的hdfs.codeC值；

- maxOpenFiles：默认值：5000：最大允许打开的HDFS文件数，当打开的文件数达到该值，最早打开的文件将会被关闭；

- minBlockReplicas：默认值：HDFS副本数，写入 HDFS 文件块的最小副本数。

​                                     该参数会影响文件的滚动配置，一般将该参数配置成1，才可以按照配置正确滚动文件。

- writeFormat：写 sequence 文件的格式。包含：Text, Writable（默认）

- callTimeout：默认值：10000，执行HDFS操作的超时时间（单位：毫秒）；

- threadsPoolSize：默认值：10，hdfs sink 启动的操作HDFS的线程数。

- rollTimerPoolSize：默认值：1，hdfs sink 启动的根据时间滚动文件的线程数。

- kerberosPrincipal：HDFS安全认证kerberos配置；

- kerberosKeytab：HDFS安全认证kerberos配置；

- proxyUser：代理用户

- round：默认值：false，是否启用时间上的”舍弃”；

- roundValue：默认值：1，时间上进行“舍弃”的值；

- roundUnit：默认值：seconds，时间上进行”舍弃”的单位，包含：second,minute,hour

### 文件到HDFS

**采集需求**

比如业务系统使用log4j生成的日志，日志内容不断增加，需要把追加到日志文件中的数据实时采集到hdfs

根据需求，首先定义以下3大要素：

- 采集源，即source——监控文件内容更新 :  exec  ‘tail -F file’
- 下沉目标，即sink——HDFS文件系统  :  hdfs sink
- Source和sink之间的传递通道——channel，可用file channel 也可以用 内存channel

**采集方案编写**

```ini
agent1.sources = source1
agent1.sinks = sink1
agent1.channels = channel1

# Describe/configure tail -F source1
agent1.sources.source1.type = exec
agent1.sources.source1.command = tail -F /home/hadoop/logs/access_log
agent1.sources.source1.channels = channel1

#configure host for source
agent1.sources.source1.interceptors = i1
agent1.sources.source1.interceptors.i1.type = host
agent1.sources.source1.interceptors.i1.hostHeader = hostname

# Describe sink1
agent1.sinks.sink1.type = hdfs
#a1.sinks.k1.channel = c1
agent1.sinks.sink1.hdfs.path =hdfs://hdp-node-01:9000/weblog/flume-collection/%y-%m-%d/%H-%M
agent1.sinks.sink1.hdfs.filePrefix = access_log
agent1.sinks.sink1.hdfs.maxOpenFiles = 5000
agent1.sinks.sink1.hdfs.batchSize= 100
agent1.sinks.sink1.hdfs.fileType = DataStream
agent1.sinks.sink1.hdfs.writeFormat =Text
agent1.sinks.sink1.hdfs.rollSize = 102400
agent1.sinks.sink1.hdfs.rollCount = 1000000
agent1.sinks.sink1.hdfs.rollInterval = 60
agent1.sinks.sink1.hdfs.round = true
agent1.sinks.sink1.hdfs.roundValue = 10
agent1.sinks.sink1.hdfs.roundUnit = minute
agent1.sinks.sink1.hdfs.useLocalTimeStamp = true

# Use a channel which buffers events in memory
agent1.channels.channel1.type = memory
agent1.channels.channel1.keep-alive = 120
agent1.channels.channel1.capacity = 500000
agent1.channels.channel1.transactionCapacity = 600

# Bind the source and sink to the channel
agent1.sources.source1.channels = channel1
agent1.sinks.sink1.channel = channel1
```

### 跟过source和sink组件

Flume支持众多的source和sink类型，详细手册可参考官方文档

<http://flume.apache.org/FlumeUserGuide.html>