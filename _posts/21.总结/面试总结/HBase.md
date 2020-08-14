---
title: HBase 面试题整理
date: "2020-06-13 11:00:00"
categories:
- 总结
- 面试总结
tags:
- 面试
toc: true
typora-root-url: ..\..\..
---

## HBase 原理

![image-20200807105906341](/img/image-20200807105906341.png)

### 存储原理

1. HBase中的每张表都通过行键按照一定的范围被分割成多个子表（HRegion），默认一个HRegion超过256M就要被分割成两个，由HRegionServer管理，管理哪些HRegion由HMaster分配。

2. HRegionServer存取一个子表时，会创建一个HRegion对象，表的Region信息存在.meta表中，该表信息可通过Zookeeper进行追踪。然后对表的每个列族(Column Family)创建一个Store实例，每个Store都会有0个或多个StoreFile与之对应，每个StoreFile都会对应一个HFile， HFile就是实际的存储文件。因此，一个HRegion有多少个列族就有多少个Store。另外，每个HRegion还拥有一个MemStore实例。memStore存储在内存中，StoreFile存储在HDFS上。

3. Region虽然是分布式存储的最小单元，但并不是存储的最小单元。Region由一个或者多个Store组成，每个store保存一个columns family；

4. 本质上MemStore就是一个内存里放着一个保存KEY/VALUE的MAP，当MemStore（默认64MB）写满之后，会开始刷磁盘操作。

5. 在进行表增加时，会先写memStore，当memStore文件大小达到一定大小时，会flush到StoreFile中。当Region中的StoreFile文件过多时，会进行Compact操作，将StoreFile合并。

### 写流程

ZooKeeper---meta--regionserver--Hlog|MemStore--storefile

1、 通过zookeeper的-ROOT-  找到表 .META. 对应的hregionserver。

2、通过META表rowkey，表名等信息找到数据对应的regine。

3、通过zookerpeer 的信息找到对应的regioneserver。

4、 hregionserver将数据写到hlog（write ahead log）。为了数据的持久化和恢复。

5、 hregionserver将数据写到内存（memstore）

6、 反馈client写成功。

### 数据flush过程

1、 当memstore数据达到阈值（默认是64M），将内存中的数据删除。

2、 将数据刷到硬盘(storefile文件)。

3、将内存中数据删除

4、删除Hlog中的历史数据，并在hlog中做标记点。

### 数据合并过程

1、 当storefile数据块达到4块，hmaster将数据块加载到本地，进行合并

2、 当合并的数据超过256M，进行拆分，将拆分后的region分配给不同的hregionserver管理

3、 当hregionser宕机后，将hregionserver上的hlog拆分，然后分配给不同的hregionserver加载，修改.META.	

4、 注意：hlog会同步到hdfs做持久化

### hbase 读流程

ZooKeeper---meta--regionserver--region--memstore--storefile

1、 通过zookeeper的-ROOT-  找到表 .META. 对应的hregionserver。

2、通过META表rowkey，表名等信息找到数据对应的regine。

3、通过zookerpeer 的信息找到对应的regioneserver。

4、连接到对应regineserver上的regine。

5、先从Memstore找数据，如果没有，再到StoreFile上读

6、 数据块会缓存

### 