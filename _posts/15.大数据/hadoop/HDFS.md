---
title: HDFS
date: "2019-12-30 14:00:00"
categories:
- 大数据
- hadoop
tags:
- HADOOP
- 大数据
toc: true
typora-root-url: ..\..\..
---

## HDFS 基本概念

### HDFS 思想

#### 设计思想

分而治之: 将大文件、大批量文件， 分布式存放在大量服务器上，以便于采取分而治之的方式对海量数据进行运算分析；

#### 在大数据系统中的应用

为各类分布式运算框架(如: mapreduce, spark, tez, ...) 提供数据存储服务

#### 重点概念

文件切块， 副本存放， 元数据管理

### HDFS的概念和特性

#### 概念

**首先，它是一个文件系统**，用于存储文件，通过统一的命名空间——目录树来定位文件

**其次，它是分布式的**，由很多服务器联合起来实现其功能，集群中的服务器有各自的角色；

#### 重要特征

1. DFS中的文件在物理上是**分块存储（block）**，块的大小可以通过配置参数( dfs.blocksize)来规定，默认大小在hadoop2.x版本中是128M，老版本中是64M
2. HDFS文件系统会给客户端提供一个**统一的抽象目录树**，客户端通过路径来访问文件，形如：hdfs://namenode:port/dir-a/dir-b/dir-c/file.data
3. 目录结构及文件分块信息(元数据)的管理由namenode节点承担， namenode是HDFS集群主节点，负责维护整个hdfs文件系统的目录树，以及每一个路径（文件）所对应的block块信息（block的id，及所在的datanode服务器）
4. 文件的各个block的存储管理由datanode节点承担， datanode是HDFS集群从节点，每一个block都可以在多个datanode上存储多个副本（副本数量也可以通过参数设置dfs.replication）

5. HDFS是设计成适应一次写入，多次读出的场景，且不支持文件的修改， 适合用来做数据分析，并不适合用来做网盘应用，因为，不便修改，延迟大，网络开销大，成本太高

## HDFS的工作机制

### HDFS 工作机制介绍

1. HDFS集群分为两大角色：NameNode、DataNode  (Secondary Namenode)
2. NameNode负责管理整个文件系统的元数据
3. DataNode 负责管理用户的文件数据块
4. 文件会按照固定的大小（blocksize）切成若干块后分布式存储在若干台datanode上
5. 每一个文件块可以有多个副本，并存放在不同的datanode上
6. Datanode会定期向Namenode汇报自身所保存的文件block信息，而namenode则会负责保持文件的副本数量
7. HDFS的内部工作机制对客户端保持透明，客户端请求访问HDFS都是通过向namenode申请来进行

### HDFS 写数据的流程

![1577692810378](/img/1577692810378.png)

1. 客户端与namenode通信请求上传文件，namenode检查目标文件是否已存在，父目录是否存在
2. namenode返回是否可以上传
3. client请求第一个 block该传输到哪些datanode服务器上
4. namenode返回可以上传的节点, 示例3个datanode服务器ABC
5. client请求3台dn中的一台A上传数据（本质上是一个RPC调用，建立pipeline），A收到请求会继续调用B，然后B调用C，将整个pipeline建立完成，逐级返回客户端
6. client开始往A上传第一个block（先从磁盘读取数据放到一个本地内存缓存），以packet为单位(chunk为校验单位)，A收到一个packet就会传给B，B传给C；A每传一个packet会放入一个应答队列等待应答
7. 当一个block传输完成之后(只要有一个节点上传成功，就算成功)，client再次请求namenode上传第二个block的服务器。

### HDFS读数据流程

![1577692929653](/img/1577692929653.png)

1. client跟namenode通信查询元数据，找到文件块所在的datanode服务器

2. cilent挑选一台datanode（就近原则，然后随机）服务器，请求建立socket流

3. datanode开始发送数据（从磁盘里面读取数据放入流，以packet为传输单位，chunk为校验单位）

4. 客户端以packet为单位接收，先在本地缓存，然后写入目标文件

### block、packet与chunk

在DFSClient写HDFS的过程中，有三个需要搞清楚的单位：block、packet与chunk；

- block是最大的一个单位，它是最终存储于DataNode上的数据粒度，由dfs.block.size参数决定，默认是64M；注：这个参数由客户端配置决定；
- packet是中等的一个单位，它是数据由DFSClient流向DataNode的粒度，以dfs.write.packet.size参数为参考值，默认是64K；注：这个参数为参考值，是指真正在进行数据传输时，会以它为基准进行调整，调整的原因是一个packet有特定的结构，调整的目标是这个packet的大小刚好包含结构中的所有成员，同时也保证写到DataNode后当前block的大小不超过设定值；
- chunk是最小的一个单位，它是DFSClient到DataNode数据传输中进行数据校验的粒度，由io.bytes.per.checksum参数决定，默认是512B；注：事实上一个chunk还包含4B的校验值，因而chunk写入packet时是516B；数据与检验值的比值为128:1，所以对于一个128M的block会有一个1M的校验文件与之对应；

## NAMENODE 工作机制

### NAMENODE 的职责

- 负责客户端请求的响应

- 元数据的管理（查询，修改）

### 元数据管理形式

- 内存元数据(NameSystem)
- 磁盘元数据镜像文件(fsimage)
- 数据操作日志文件（edits文件， 可通过日志运算出元数据）

### 元数据的存储机制

1. 内存中有一份完整的原数据(内存metadate)
2. 磁盘中有一个"准完整"的原数据镜像(fsimage)文件(在namenode的工作目录中)
3. 用于衔接metadata和持久化元数据镜像的fsimage之间的操作日志(edits文件), 当客户端对hdfs中的文件进行新增或者修改操作，操作记录会首先被记录到edits日志文件中，当客户端操作成功后，相应的原数据会更新到内存meta.data中， 并且每隔一定的间隔hdfs会将当前的metadata同步到fsimage镜像文件中

### 元数据手动查看

hdfs命令

```shell
hdfs oev -i edits -o edits.xml
hdfs oiv -i fsimage_0000000000000000087 -p XML -o fsimage.xml
```

### 元数据的checkpoint

由于在数据备份的时候会占用计算资源，所以为了减轻namenode的负载，通常可以将数据备份的工作交给另外一个专门用来做数据备份的namenode--> sencondary namenode

每隔一段时间，会由secondary namenode 将namenode上积累的所有edits和一个最新的fsimage下载到本地(只有第一次merge才会下载fsimage)，并加载到内存进行merge(这个过程称之为checkpoint)

![1577693922715](/img/1577693922715.png)

### namenode的一些情况

**namenode如果宕机，hdfs是否还能正常提供服务**

不能，secondarynamenode虽然有元数据信息，但是不能更新元数据， 不能充当namenode使用

**如果namenode的硬盘损坏，元数据是否能回复，能恢复多少?**

可以恢复最后一次merge之前的数据， 只需要将secondarynamenode的数据目录替换成namenode的数据目录

**配置namenode的工作目录时，有哪些可以注意的事项**

可以将namenode的元数据保存到多块物理磁盘上例如如下的namenode配置

```xml
<property>
<name>dfs.name.dir</name>
<value>/home/hadoop/name1,/home/hadoop/name2</value>
</property>
```

### checkpoint 的触发条件相关配置

```xml
dfs.namenode.checkpoint.check.period=60  #检查触发条件是否满足的频率，60秒
dfs.namenode.checkpoint.dir=file://${hadoop.tmp.dir}/dfs/namesecondary
#以上两个参数做checkpoint操作时，secondary namenode的本地工作目录
dfs.namenode.checkpoint.edits.dir=${dfs.namenode.checkpoint.dir}

dfs.namenode.checkpoint.max-retries=3  #最大重试次数
dfs.namenode.checkpoint.period=3600  #两次checkpoint之间的时间间隔3600秒
dfs.namenode.checkpoint.txns=1000000 #两次checkpoint之间最大的操作记录
```

### 元数据目录说明

在第一次部署好Hadoop集群的时候，我们需要在NameNode（NN）节点上格式化磁盘：

```shell
hadoop namenode -format
```

#### 元数据目录介绍

格式化完成之后，将会在$dfs.namenode.name.dir/current目录下如下的文件结构

```shell
current/
|-- VERSION
|-- edits_*
|-- fsimage_0000000000008547077
|-- fsimage_0000000000008547077.md5
|-- seen_txid
```

其中的dfs.name.dir是在hdfs-site.xml文件中配置的，默认值如下：

```xml
<property>
  <name>dfs.name.dir</name>
  <value>file://${hadoop.tmp.dir}/dfs/name</value>
</property>

# hadoop.tmp.dir是在core-site.xml中配置的，默认值如下
<property>
  <name>hadoop.tmp.dir</name>
  <value>/tmp/hadoop-${user.name}</value>
  <description>A base for other temporary directories.</description>
</property>
```

dfs.namenode.name.dir属性可以配置多个，如/data1/dfs/name,/data2/dfs/name,/data3/dfs/name,....。

各个目录存储的文件结构和内容都完全一样，相当于备份，这样做的好处是当其中一个目录损坏了，也不会影响到Hadoop的元数据，特别是当其中一个目录是NFS（网络文件系统Network File System，NFS）之上，即使你这台机器损坏了，元数据也得到保存。

#### 元数据目录文件介绍

**VERSION文件**

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

3. cTime表示NameNode存储时间的创建时间，由于我的NameNode没有更新过，所以这里的记录值为0，以后对NameNode升级之后，cTime将会记录更新时间戳；

4. layoutVersion表示HDFS永久性数据结构的版本信息， 只要数据结构变更，版本号也要递减，此时的HDFS也需要升级，否则磁盘仍旧是使用旧版本的数据结构，这会导致新版本的NameNode无法使用

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

文件中记录的是edits滚动的序号，每次重启namenode时，namenode就知道要将哪些edits进行加载edits

**fsimage文件和edits文件**

fsimage: 元数据的镜像文件

edits: 元数据的滚动日志文件，每次merge之后会对之前的日志文件进行清除

## DATANODE工作机制

### DATANODE 工作职责

- 存储管理用户的文件块数据
- 定期向namenode汇报自身所持有的block信息(通过心跳上报)， 当集群中的节点失效，或者block存在丢失的时候，集群可以根据汇报信息恢复block初始副本数量的问题

### DATANODE 汇报间隔设置参数

```xml
<property>
	<name>dfs.blockreport.intervalMsec</name>
	<value>3600000</value>
	<description>Determines block reporting interval in milliseconds.</description>
</property>
```

### DATANODE掉线判断时限参数

datanode进程死亡或者网络故障造成datanode无法与namenode通信，namenode不会立即把该节点判定为死亡，要经过一段时间，这段时间暂称作超时时长。HDFS默认的超时时长为10分钟+30秒。如果定义超时时间为timeout，则超时时长的计算公式为：

​	**timeout  = 2 * heartbeat.recheck.interval + 10 * dfs.heartbeat.interval。**

.默认的heartbeat.recheck.interval 大小为5分钟，dfs.heartbeat.interval默认为3秒。

heartbeat.recheck.interval的单位为毫秒，dfs.heartbeat.interval的单位为秒

dfs配置参数

```xml
<property>
        <name>heartbeat.recheck.interval</name>
        <value>2000</value>
</property>
<property>
        <name>dfs.heartbeat.interval</name>
        <value>1</value>
</property>
```

## HDFS客户端操作

### 命令行客户端

**命令格式**

```shell
hadoop  fs  -ls  /
```

**命令行参数**

```shell
      [-appendToFile <localsrc> ... <dst>]
      [-cat [-ignoreCrc] <src> ...]
      [-checksum <src> ...]
      [-chgrp [-R] GROUP PATH...]
      [-chmod [-R] <MODE[,MODE]... | OCTALMODE> PATH...]
      [-chown [-R] [OWNER][:[GROUP]] PATH...]
      [-copyFromLocal [-f] [-p] <localsrc> ... <dst>]
      [-copyToLocal [-p] [-ignoreCrc] [-crc] <src> ... <localdst>]
      [-count [-q] <path> ...]
      [-cp [-f] [-p] <src> ... <dst>]
      [-createSnapshot <snapshotDir> [<snapshotName>]]
      [-deleteSnapshot <snapshotDir> <snapshotName>]
      [-df [-h] [<path> ...]]
      [-du [-s] [-h] <path> ...]
      [-expunge]
      [-get [-p] [-ignoreCrc] [-crc] <src> ... <localdst>]
      [-getfacl [-R] <path>]
      [-getmerge [-nl] <src> <localdst>]
      [-help [cmd ...]]
      [-ls [-d] [-h] [-R] [<path> ...]]
      [-mkdir [-p] <path> ...]
      [-moveFromLocal <localsrc> ... <dst>]
      [-moveToLocal <src> <localdst>]
      [-mv <src> ... <dst>]
      [-put [-f] [-p] <localsrc> ... <dst>]
      [-renameSnapshot <snapshotDir> <oldName> <newName>]
      [-rm [-f] [-r|-R] [-skipTrash] <src> ...]
      [-rmdir [--ignore-fail-on-non-empty] <dir> ...]
      [-setfacl [-R] [{-b|-k} {-m|-x <acl_spec>} <path>]|[--set <acl_spec> <path>]]
      [-setrep [-R] [-w] <rep> <path> ...]
      [-stat [format] <path> ...]
      [-tail [-f] <file>]
      [-test -[defsz] <path>]
      [-text [-ignoreCrc] <src> ...]
      [-touchz <path> ...]
      [-usage [cmd ...]]
```

**常用命令介绍**

- **-help**             

  功能：输出这个命令参数手册

- **--ls**                  

	功能：显示目录信息

	示例：hadoop fs -ls hdfs://hadoop-server01:9000/

	备注：这些参数中，所有的hdfs路径都可以简写,   hadoop fs -ls /   等同于上一条命令的效果

- **-mkdir**              

	功能：在hdfs上创建目录

	示例：hadoop fs  -mkdir  -p  /aaa/bbb/cc/dd

- **-moveFromLocal**            

	功能：从本地剪切粘贴到hdfs

	示例：hadoop  fs  - moveFromLocal  /home/hadoop/a.txt  /aaa/bbb/cc/dd

- **-moveToLocal**              

	功能：从hdfs剪切粘贴到本地

	示例：hadoop  fs  - moveToLocal   /aaa/bbb/cc/dd  /home/hadoop/a.txt*

- **--appendToFile**  

	功能：追加一个文件到已经存在的文件末尾

	示例：hadoop  fs  -appendToFile  ./hello.txt  hdfs://hadoop-server01:9000/hello.txt

	可以简写为： hadoop  fs  -appendToFile  ./hello.txt  /hello.txt

- **-cat**

	功能：显示文件内容

	示例：hadoop fs -cat  /hello.txt

- **-tail**                 

	功能：显示一个文件的末尾

	示例：hadoop  fs  -tail  /weblog/access_log.1

- **-text**                  

	功能：以字符形式打印一个文件的内容

	示例：hadoop  fs  -text  /weblog/access_log.1

- **-chgrp** ,  **-chmod**,  **-chown**

	功能：linux文件系统中的用法一样，对文件所属权限

	示例：

	hadoop  fs  -chmod  666  /hello.txt

	hadoop  fs  -chown  someuser:somegrp   /hello.txt

- **-copyFromLocal**    

	功能：从本地文件系统中拷贝文件到hdfs路径去

	示例：hadoop  fs  -copyFromLocal  ./jdk.tar.gz  /aaa/

- **-copyToLocal**      

	功能：从hdfs拷贝到本地

	示例：hadoop fs -copyToLocal /aaa/jdk.tar.gz

- **-cp**              

  功能：从hdfs的一个路径拷贝hdfs的另一个路径

  示例：hadoop  fs  -cp  /aaa/jdk.tar.gz  /bbb/jdk.tar.gz.2

- **-mv**                     

	功能：在hdfs目录中移动文件

	示例：hadoop  fs  -mv  /aaa/jdk.tar.gz  /

- **-get**              

	功能：等同于copyToLocal，就是从hdfs下载文件到本地

	示例：hadoop fs -get  /aaa/jdk.tar.gz

- **-getmerge**             

	功能：合并下载多个文件

	示例：比如hdfs的目录 /aaa/下有多个文件:log.1, log.2,log.3,...

	hadoop fs -getmerge /aaa/log.* ./log.sum

- **-put**                

	功能：等同于copyFromLocal

	示例：hadoop  fs  -put  /aaa/jdk.tar.gz  /bbb/jdk.tar.gz.2

- **-rm**                

	功能：删除文件或文件夹

	示例：hadoop fs -rm -r /aaa/bbb/

- **-rmdir**                 

	功能：删除空目录

	示例：hadoop  fs  -rmdir   /aaa/bbb/ccc

- **-df**               

	功能：统计文件系统的可用空间信息*

	示例：hadoop  fs  -df  -h  /

- **-du** 

	功能：统计文件夹的大小信息

	示例：hadoop  fs  -du  -s  -h /aaa/*

- **-count**         

	功能：统计一个指定目录下的文件节点数量

	示例：hadoop fs -count /aaa/

- **-setrep**                

	功能：设置hdfs中文件的副本数量

	示例：hadoop fs -setrep 3 /aaa/jdk.tar.gz

### java api 使用

#### 引入依赖(maven)

```xml
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-client</artifactId>
    <version>2.6.1</version>
</dependency>
```

#### window 下开发需要注意

建议在linux下进行hadoop应用的开发，不会存在兼容性问题。如在window上做客户端应用开发，需要设置以下环境：

A、在windows的某个目录下解压一个hadoop的安装包

B、将安装包下的lib和bin目录用对应windows版本平台编译的本地库替换(下载地址: **https://github.com/steveloughran/winutils**,下载之后直接解压,将bin目录里的内容直接覆盖到hadoop的bin )

C、在window系统中配置HADOOP_HOME指向你解压的安装包

D、在windows系统的path变量中加入hadoop的bin目录

#### java 客户端的配置

在java中操作hdfs，首先要获得一个客户端实例

```java
Configuration conf = new Configuration()  
fs = FileSystem.get(new URI("hdfs://master:9000"),conf,"hadoop"); 
```

而我们的操作目标是HDFS，所以获取到的fs对象应该是DistributedFileSystem的实例；

get方法是从何处判断具体实例化那种客户端类呢？

**—从conf中的一个参数 fs.defaultFS的配置值判断；**

如果我们的代码中没有指定fs.defaultFS，并且工程classpath下也没有给定相应的配置，conf中的默认值就来自于hadoop的jar包中的core-default.xml，默认值为： file:///，则获取的将不是一个DistributedFileSystem的实例，而是一个本地文件系统的客户端对象

**配置方法**

```java
conf = new Configuration();
conf.set("fs.defaultFS", "hdfs://master:9000");  // 或者conf.addResource, 直接添加配置文件
fs = FileSystem.get(new URI("hdfs://master:9000"),conf,"hadoop");
```

#### java 客户端 HDFS增删改查示例

```java
package com.bigdata.utils.hdfs;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.*;
import org.junit.Before;
import org.junit.Test;

import java.net.URI;
import java.util.Iterator;
import java.util.Map.Entry;
/**
 *
 * 客户端去操作hdfs时，是有一个用户身份的
 * 默认情况下，hdfs客户端api会从jvm中获取一个参数来作为自己的用户身份：-DHADOOP_USER_NAME=hadoop
 *
 * 也可以在构造客户端fs对象时，通过参数传递进去
 * @author
 *
 */
public class HdfsClientDemo {
	FileSystem fs = null;
	Configuration conf = null;
	@Before
	public void init() throws Exception{

		conf = new Configuration();
//		conf.set("fs.defaultFS", "hdfs://mini1:9000");
		conf.set("dfs.replication", "5");

		//拿到一个文件系统操作的客户端实例对象
		fs = FileSystem.get(conf);
		//可以直接传入 uri和用户身份
		fs = FileSystem.get(new URI("hdfs://mini1:9000"),conf,"hadoop");
	}

	/**
	 * 上传文件
	 * @throws Exception
	 */
	@Test
	public void testUpload() throws Exception {

		fs.copyFromLocalFile(new Path("c:/access.log"), new Path("/access.log.copy"));
		fs.close();
	}


	/**
	 * 下载文件
	 * @throws Exception
	 */
	@Test
	public void testDownload() throws Exception {

		fs.copyToLocalFile(new Path("/access.log.copy"), new Path("d:/"));
	}


	/**
	 * 打印参数
	 */
	@Test
	public void testConf(){

		Iterator<Entry<String, String>> it = conf.iterator();
		while(it.hasNext()){
			Entry<String, String> ent = it.next();
			System.out.println(ent.getKey() + " : " + ent.getValue());

		}

	}


	@Test
	public void testMkdir() throws Exception {
		boolean mkdirs = fs.mkdirs(new Path("/testmkdir/aaa/bbb"));
		System.out.println(mkdirs);

	}


	@Test
	public void testDelete() throws Exception {

		boolean flag = fs.delete(new Path("/testmkdir/aaa"), true);
		System.out.println(flag);

	}


	/**
	 * 递归列出指定目录下所有子文件夹中的文件
	 * @throws Exception
	 */
	@Test
	public void testLs() throws Exception {

		RemoteIterator<LocatedFileStatus> listFiles = fs.listFiles(new Path("/"), true);

		while(listFiles.hasNext()){
			LocatedFileStatus fileStatus = listFiles.next();
			System.out.println("blocksize: " +fileStatus.getBlockSize());
			System.out.println("owner: " +fileStatus.getOwner());
			System.out.println("Replication: " +fileStatus.getReplication());
			System.out.println("Permission: " +fileStatus.getPermission());
			System.out.println("Name: " +fileStatus.getPath().getName());
			System.out.println("------------------");
			BlockLocation[] blockLocations = fileStatus.getBlockLocations();
			for(BlockLocation b:blockLocations){
				System.out.println("块起始偏移量: " +b.getOffset());
				System.out.println("块长度:" + b.getLength());
				//块所在的datanode节点
				String[] datanodes = b.getHosts();
				for(String dn:datanodes){
					System.out.println("datanode:" + dn);
				}
			}



		}

	}

	@Test
	public void testLs2() throws Exception {

		FileStatus[] listStatus = fs.listStatus(new Path("/"));
		for(FileStatus file :listStatus){

			System.out.println("name: " + file.getPath().getName());
			System.out.println((file.isFile()?"file":"directory"));

		}

	}

	public static void main(String[] args) throws Exception {
		Configuration conf = new Configuration();
		conf.set("fs.defaultFS", "hdfs://mini1:9000");
		//拿到一个文件系统操作的客户端实例对象
		FileSystem fs = FileSystem.get(conf);

		fs.copyFromLocalFile(new Path("c:/access.log"), new Path("/access.log.copy"));
		fs.close();
	}
}
```

### java 客户端 HDFS通过流的方式上传下载文件

```java
package com.bigdata.utils.hdfs;

import org.apache.commons.io.IOUtils;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FSDataOutputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.junit.Before;
import org.junit.Test;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.net.URI;


/**
 * 用流的方式来操作hdfs上的文件
 * 可以实现读取指定偏移量范围的数据
 * @author
 *
 */
public class HdfsStreamAccess {

    FileSystem fs = null;
    Configuration conf = null;

    @Before
    public void init() throws Exception{

        conf = new Configuration();
        //拿到一个文件系统操作的客户端实例对象
//		fs = FileSystem.get(conf);
        //可以直接传入 uri和用户身份
        fs = FileSystem.get(new URI("hdfs://mini1:9000"),conf,"hadoop");
    }

    /**
     * 通过流的方式上传文件到hdfs
     * @throws Exception
     */
    @Test
    public void testUpload() throws Exception {

        FSDataOutputStream outputStream = fs.create(new Path("/angelababy.love"), true);
        FileInputStream inputStream = new FileInputStream("c:/angelababy.love");

        IOUtils.copy(inputStream, outputStream);

    }

    @Test
    public void testDownLoad() throws Exception {

        FSDataInputStream inputStream = fs.open(new Path("/angelababy.love"));

        FileOutputStream outputStream = new FileOutputStream("d:/angelababy.love");

        IOUtils.copy(inputStream, outputStream);

    }


    @Test
    public void testRandomAccess() throws Exception{

        FSDataInputStream inputStream = fs.open(new Path("/angelababy.love"));

        inputStream.seek(12);

        FileOutputStream outputStream = new FileOutputStream("d:/angelababy.love.part2");

        IOUtils.copy(inputStream, outputStream);


    }

    /**
     * 显示hdfs上文件的内容
     * @throws IOException
     * @throws IllegalArgumentException
     */
    @Test
    public void testCat() throws IllegalArgumentException, IOException{

        FSDataInputStream in = fs.open(new Path("/angelababy.love"));

        IOUtils.copy(in, System.out);
    }
}
```

### Java客户端获取文件block信息并读取指定block内容

```java
@Test
	public void testCat() throws IllegalArgumentException, IOException{
		FSDataInputStream in = fs.open(new Path("/weblog/input/access.log.10"));
		//拿到文件信息
		FileStatus[] listStatus = fs.listStatus(new Path("/weblog/input/access.log.10"));
		//获取这个文件的所有block的信息
		BlockLocation[] fileBlockLocations = fs.getFileBlockLocations(listStatus[0], 0L, listStatus[0].getLen());
		//第一个block的长度
		long length = fileBlockLocations[0].getLength();
		//第一个block的起始偏移量
		long offset = fileBlockLocations[0].getOffset();
		
		System.out.println(length);
		System.out.println(offset);
		
		//获取第一个block写入输出流
//		IOUtils.copyBytes(in, System.out, (int)length);
		byte[] b = new byte[4096];
		
		FileOutputStream os = new FileOutputStream(new File("d:/block0"));
		while(in.read(offset, b, 0, 4096)!=-1){
			os.write(b);
			offset += 4096;
			if(offset>=length) return;
		};
		os.flush();
		os.close();
		in.close();
	}
```

