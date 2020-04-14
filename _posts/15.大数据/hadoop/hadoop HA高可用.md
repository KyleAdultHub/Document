---
title: hadoop HA高可用
date: "2020-01-17 11:00:00"
categories:
- 大数据
- HADOOP
tags:
- 大数据
- HADOOP
toc: true
typora-root-url: ..\..\..
---

## HADOOP的HA机制

正式引入HA机制是从hadoop2.0开始，之前的版本中没有HA机制

### HA的运作机制介绍

所谓HA，即高可用（7*24小时不中断服务）

实现高可用最关键的是消除单点故障

hadoop-ha严格来说应该分成各个组件的HA机制——**HDFS**的HA、**YARN**的HA

### HDFS的HA机制详解

**HDFS怎么实现HA机制**

> 通过双namenode消除单点故障

**双namenode的工作方式**

1. 元数据管理
   - 元数据管理方式与传统方式不同，在每个namenode内存中各自保存一份元数据
   - Edits日志只能有一份，只有Active状态的namenode节点可以做写操作， 两个namenode都可以读取edits
   - 共享的edits放在一个共享存储中管理（qjournal和NFS两个主流实现）

2. 状态管理功能模块

   - 实现了一个zkfailover，常驻在每一个namenode所在的节点
   - 每一个zkfailover负责监控自己所在namenode节点，利用zk进行状态标识
   - 当需要进行状态切换时，由zkfailover来负责切换
   - 切换时需要防止brain split现象的发生，即程序假死现象，会导致脑裂发生，所以需要在切换程序的时候，强制停止异常程序

3. 服务名称

   双namenode 对外提供服务会用一个逻辑名称来表示，所以client只需要请求namenode的逻辑名称即可，不用关注哪个namenode正在对外提供服务

**HDFS的HA图解**

![1579490818287](/img/1579490818287.png)

### YARN 的HA机制

**YARN怎么实现HA机制**

YARN-resource manager 也是通过双节点的方式，来实现HA的

**双resource manager的工作方式**

1. 状态管理

   - Yarn的高可用状态管理相对于HDFS非常简单，双resource通过zookeeper来同步每个节点的状态，根据对方的状态来更改自己的服务状态
   - 如果客户端正在运行程序，这是resource发生了崩溃，这时resource会自动切换到另外一台机器上，MAPREDUCE会自动重试

2. 服务名称

   双resource 对外提供服务会用一个逻辑名称来表示，client不用关注哪个namenode正在对外提供服务

**YARN高可用图解**

![1579491516933](/img/1579491516933.png)

### HDFS联邦机制(namenode水平扩展)

**HDFS HA 还存在的问题**

1. 系统扩展性方面，元数据存储在NN内存中，会受到NN内存上限的制约。
2. 整体性能方面，吞吐量受单个NN的影响。
3. 隔离性方面，一个程序可能会影响其他运行的程序，如一个程序消耗过多资源导致其他程序无法顺利运行。HDFS HA本质上还是单名称节点。

**HDFS联邦的作用**

1. 在HDFS联邦中，设计了多个相互独立的NN，使得HDFS的命名服务能够水平扩展，这些NN分别进行各自命名空间和块的管理，不需要彼此协调。每个DN要向集群中所有的NN注册，并周期性的发送心跳信息和块信息，报告自己的状态。
2. HDFS联邦拥有多个独立的命名空间，其中，每一个命名空间管理属于自己的一组块，这些属于同一个命名空间的块组成一个“块池”。每个DN会为多个块池提供块的存储，块池中的各个块实际上是存储在不同DN中的。

**HDFS联邦图解**

![1579491564512](/img/1579491564512.png)

### NameNode的安全模式

**什么是namenode的安全模式**

1. 在name刚启动的时候，内存中只有文件和文件的块id以及副本数量，但是并不知道所有块在哪个datanode上
2. namenode这个时候需要等待所有的datanode向他汇报自身持有的块信息，namenode才能再元数据中补全文件块信息中的位置信息
3. 只有当namenode找到99.8%的块位置信息，才会退出安全模式，正常对外提供服务

**namenode 安全模式图解**

![1579491589379](/img/1579491589379.png)

## HA集群安装部署

### 前期准备

1. 准备7台主机(以7台来举例 mini1, mini2, mini3, mini4, mini5, mini6, mini7)
2. 修改linux主机名
3. 修改/etc/hosts 域名映射
4. 关闭防火墙
5. 配置ssh免密登录
6. 安装JDK，配置JAVA环境变量
7. 编译HADOOP安装包hadoop-2.6.4

### 集群规划

```shell
mini1	192.168.1.200	jdk、hadoop					 NameNode、DFSZKFailoverController(zkfc)
mini2	192.168.1.201	jdk、hadoop					 NameNode、DFSZKFailoverController(zkfc)
mini3	192.168.1.202	jdk、hadoop					 ResourceManager 
mini4	192.168.1.203	jdk、hadoop					 ResourceManager
mini5	192.168.1.205	jdk、hadoop、zookeeper		DataNode、NodeManager、JournalNode、QuorumPeerMain
mini6	192.168.1.206	jdk、hadoop、zookeeper		DataNode、NodeManager、JournalNode、QuorumPeerMain
mini7	192.168.1.207	jdk、hadoop、zookeeper		DataNode、NodeManager、JournalNode、QuorumPeerMain
```

说明:

1. 在hadoop2.0中通常由两个NameNode组成，一个处于active状态，另一个处于standby状态。Active NameNode对外提供服务，而Standby NameNode则不对外提供服务，仅同步active namenode的状态，以便能够在它失败时快速进行切换。
   hadoop2.0官方提供了两种HDFS HA的解决方案，一种是NFS，另一种是QJM。这里我们使用简单的QJM。在该方案中，主备NameNode之间通过一组JournalNode同步元数据信息，一条数据只要成功写入多数JournalNode即认为写入成功。通常配置奇数个JournalNode
   这里还配置了一个zookeeper集群，用于ZKFC（DFSZKFailoverController）故障转移，当Active NameNode挂掉了，会自动切换Standby NameNode为standby状态
2. hadoop-2.2.0中依然存在一个问题，就是ResourceManager只有一个，存在单点故障，hadoop-2.6.4解决了这个问题，有两个ResourceManager，一个是Active，一个是Standby，状态由zookeeper进行协调

### 安装步骤

#### 一. 配置zookeeper集群

1. 解压zookeeper

   ```shell
   tar -zxvf zookeeper-3.4.5.tar.gz -C /home/hadoop/app/
   ```

2. 修改配置

   ```shell
   cd /home/hadoop/app/zookeeper-3.4.5/conf/
   cp zoo_sample.cfg zoo.cfg
   vim zoo.cfg
   # 修改   dataDir=/home/hadoop/app/zookeeper-3.4.5/tmp
   # 添加
   server.1=mini5:2888:3888
   server.2=mini6:2888:3888
   server.3=mini7:2888:3888
   # 创建tmp文件夹
   mkdir /home/hadoop/app/zookeeper-3.4.5/tmp
   echo 1 > /home/hadoop/app/zookeeper-3.4.5/tmp/myid
   ```

3. 拷贝zk到集群其他节点

   ```shell
   scp -r /home/hadoop/app/zookeeper-3.4.5/ mini6:/home/hadoop/app/
   scp -r /home/hadoop/app/zookeeper-3.4.5/ mini7:/home/hadoop/app/
   ```

   > 注意要修改其他节点的myid内容
   > hadoop06：  echo 2 > /home/hadoop/app/zookeeper-3.4.5/tmp/myid
   > hadoop07：  echo 3 > /home/hadoop/app/zookeeper-3.4.5/tmp/myid

#### 二. 配置hadoop集群

1. 解压hadoop

   ```shell
   tar -zxvf hadoop-2.6.4.tar.gz -C /home/hadoop/app/
   ```

2. 配置HDFS

   **添加环境变量**

   ```shell
   #将hadoop添加到环境变量中
   vim /etc/profile
   export JAVA_HOME=/usr/java/jdk1.7.0_55
   export HADOOP_HOME=/hadoop/hadoop-2.6.4
   export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin
   
   #hadoop2.0的配置文件全部在$HADOOP_HOME/etc/hadoop下
   cd /home/hadoop/app/hadoop-2.6.4/etc/hadoop
   ```

   **修改hadoop-env.sh**

   ```shell
   vi hadoop-env.sh
   # 修改  export JAVA_HOME=/home/hadoop/app/jdk1.7.0_55
   ```

3. 修改配置文件

   **core-site.xml**

   ```xml
   <configuration>
   <!-- 指定hdfs的nameservice为ns1 -->
   <property>
   <name>fs.defaultFS</name>
   <value>hdfs://bi/</value>
   </property>
   <!-- 指定hadoop临时目录 -->
   <property>
   <name>hadoop.tmp.dir</name>
   <value>/home/hadoop/app/hdpdata/</value>
   </property>
   
   <!-- 指定zookeeper地址 -->
   <property>
   <name>ha.zookeeper.quorum</name>
   <value>mini5:2181,mini6:2181,mini7:2181</value>
   </property>
   </configuration>
   ```

   **hdfs-site.xml**

   ```xml
   <configuration>
   <!--指定hdfs的nameservice为bi，需要和core-site.xml中的保持一致 -->
   <property>
   <name>dfs.nameservices</name>
   <value>bi</value>
   </property>
   <!-- bi下面有两个NameNode，分别是nn1，nn2 -->
   <property>
   <name>dfs.ha.namenodes.bi</name>
   <value>nn1,nn2</value>
   </property>
   <!-- nn1的RPC通信地址 -->
   <property>
   <name>dfs.namenode.rpc-address.bi.nn1</name>
   <value>mini1:9000</value>
   </property>
   <!-- nn1的http通信地址 -->
   <property>
   <name>dfs.namenode.http-address.bi.nn1</name>
   <value>mini1:50070</value>
   </property>
   <!-- nn2的RPC通信地址 -->
   <property>
   <name>dfs.namenode.rpc-address.bi.nn2</name>
   <value>mini2:9000</value>
   </property>
   <!-- nn2的http通信地址 -->
   <property>
   <name>dfs.namenode.http-address.bi.nn2</name>
   <value>mini2:50070</value>
   </property>
   <!-- 指定NameNode的edits元数据在JournalNode上的存放位置 -->
   <property>
   <name>dfs.namenode.shared.edits.dir</name>
   <value>qjournal://mini5:8485;mini6:8485;mini7:8485/bi</value>
   </property>
   <!-- 指定JournalNode在本地磁盘存放数据的位置 -->
   <property>
   <name>dfs.journalnode.edits.dir</name>
   <value>/home/hadoop/journaldata</value>
   </property>
   <!-- 开启NameNode失败自动切换 -->
   <property>
   <name>dfs.ha.automatic-failover.enabled</name>
   <value>true</value>
   </property>
   <!-- 配置失败自动切换实现方式 -->
   <property>
   <name>dfs.client.failover.proxy.provider.bi</name>
   <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
   </property>
   <!-- 配置隔离机制方法，多个机制用换行分割，即每个机制暂用一行-->
   <property>
   <name>dfs.ha.fencing.methods</name>
   <value>
   sshfence
   shell(/bin/true)
   </value>
   </property>
   <!-- 使用sshfence隔离机制时需要ssh免登陆 -->
   <property>
   <name>dfs.ha.fencing.ssh.private-key-files</name>
   <value>/home/hadoop/.ssh/id_rsa</value>
   </property>
   <!-- 配置sshfence隔离机制超时时间 -->
   <property>
   <name>dfs.ha.fencing.ssh.connect-timeout</name>
   <value>30000</value>
   </property>
   </configuration>
   ```

   **mapred-site.xml**

   ```xml
   <configuration>
   <!-- 指定mr框架为yarn方式 -->
   <property>
   <name>mapreduce.framework.name</name>
   <value>yarn</value>
   </property>
   </configuration>	
   ```

   **yarn-site.xml**

   ```xml
   <configuration>
   <!-- 开启RM高可用 -->
   <property>
   <name>yarn.resourcemanager.ha.enabled</name>
   <value>true</value>
   </property>
   <!-- 指定RM的cluster id -->
   <property>
   <name>yarn.resourcemanager.cluster-id</name>
   <value>yrc</value>
   </property>
   <!-- 指定RM的名字 -->
   <property>
   <name>yarn.resourcemanager.ha.rm-ids</name>
   <value>rm1,rm2</value>
   </property>
   <!-- 分别指定RM的地址 -->
   <property>
   <name>yarn.resourcemanager.hostname.rm1</name>
   <value>mini3</value>
   </property>
   <property>
   <name>yarn.resourcemanager.hostname.rm2</name>
   <value>mini4</value>
   </property>
   <!-- 指定zk集群地址 -->
   <property>
   <name>yarn.resourcemanager.zk-address</name>
   <value>mini5:2181,mini6:2181,mini7:2181</value>
   </property>
   <property>
   <name>yarn.nodemanager.aux-services</name>
   <value>mapreduce_shuffle</value>
   </property>
   </configuration>
   ```

   **slave文件**

   ```shell
   # slaves是指定子节点的位置，因为要在hadoop01上启动HDFS、在hadoop03启动yarn，所以hadoop01上的slaves文件指定的是datanode的位置，hadoop03上的slaves文件指定的是nodemanager的位置
   
   mini5
   mini6
   mini7
   ```

4. 拷贝项目到其他节点

   **配置免密登录**

   ```
   #首先要配置mini1到mini2、mini3、mini4、mini5、mini6、mini7的免密码登陆
   #在mini1上生产一对钥匙
   ssh-keygen -t rsa
   #将公钥拷贝到其他节点，包括自己
   ssh-coyp-id mini1
   ssh-coyp-id mini2
   ssh-coyp-id mini3
   ssh-coyp-id mini4
   ssh-coyp-id mini5
   ssh-coyp-id mini6
   ssh-coyp-id mini7
   #配置mini3到mini4、mini5、mini6、mini7的免密码登陆
   #在mini3上生产一对钥匙
   ssh-keygen -t rsa
   #将公钥拷贝到其他节点
   ssh-coyp-id mini3				
   ssh-coyp-id mini4
   ssh-coyp-id mini5
   ssh-coyp-id mini6
   ssh-coyp-id mini7
   #注意：两个namenode之间要配置ssh免密码登陆，别忘了配置hadoop02到hadoop01的免登陆
   在mini2上生产一对钥匙
   ssh-keygen -t rsa
   ssh-coyp-id -i mini1				
   ```

   **将配置好的hadoop拷贝到其他节点**

   ```
   scp -r  /home/hadoop/app/hadoop-2.6.4 mini2:/home/hadoop/app/hadoop-2.6.4
   scp -r  /home/hadoop/app/hadoop-2.6.4 mini3:/home/hadoop/app/hadoop-2.6.4
   scp -r  /home/hadoop/app/hadoop-2.6.4 mini4:/home/hadoop/app/hadoop-2.6.4
   scp -r  /home/hadoop/app/hadoop-2.6.4 mini5:/home/hadoop/app/hadoop-2.6.4
   scp -r  /home/hadoop/app/hadoop-2.6.4 mini6:/home/hadoop/app/hadoop-2.6.4
   scp -r  /home/hadoop/app/hadoop-2.6.4 mini7:/home/hadoop/app/hadoop-2.6.4
   ```

#### 三.启动Hadoop HA

1. 启动zookeeper集群（分别在mini5、mini6、mini7上启动zk）

   ```shell
   cd /hadoop/zookeeper-3.4.5/bin/
   ./zkServer.sh start
   # 查看状态：一个leader，两个follower
   ./zkServer.sh status
   ```

2. 启动journalnode（分别在在mini5、mini6、mini7上执行)

   ```shell
   cd /hadoop/hadoop-2.6.4
   sbin/hadoop-daemon.sh start journalnode
   # 运行jps命令检验，hadoop05、hadoop06、hadoop07上多了JournalNode进程
   ```

3. 格式化HDFS

   ```shell
   # 在mini1上执行命令:
   hdfs namenode -format
   # 格式化后会在根据core-site.xml中的hadoop.tmp.dir配置生成个文件，这里我配置的是/hadoop/hadoop-2.6.4/tmp，然后将/hadoop/hadoop-2.6.4/tmp拷贝到hadoop02的/hadoop/hadoop-2.6.4/下。
   scp -r tmp/ hadoop02:/home/hadoop/app/hadoop-2.6.4/
   # 也可以这样，建议hdfs namenode -bootstrapStandby
   ```

4. 格式化ZKFC(在mini1上执行一次即可)

   ```shell
   hdfs zkfc -formatZK
   ```

5. 启动HDFS(在mini1上执行)

   ```shell
   sbin/start-dfs.sh
   ```

6. 启动YARN

   ```shell
   # 启动YARN，是在hadoop02上执行start-yarn.sh，把namenode和resourcemanager分开是因为性能问题，因为他们都要占用大量资源，所以把他们分开了，他们分开了就要分别在不同的机器上启动)
   sbin/start-yarn.sh
   ```

7. 验证集群

   到此，hadoop-2.6.4配置并安装完毕

   **浏览器访问验证**

   ```shell
   http://hadoop00:50070
   NameNode 'hadoop01:9000' (active)
   http://hadoop01:50070
   NameNode 'hadoop02:9000' (standby)
   ```

   **验证HDFS HA**

   ```shell
   # 首先向hdfs上传一个文件
   hadoop fs -put /etc/profile /profile
   hadoop fs -ls /
   # 然后再kill掉active的NameNode
   kill -9 <pid of NN>
   # 通过浏览器访问：http://192.168.1.202:50070
   # NameNode 'hadoop02:9000' (active)
   # 这个时候hadoop02上的NameNode变成了active
   # 在执行命令：
   hadoop fs -ls /
   # -rw-r--r--   3 root supergroup       1926 2014-02-06 15:36 /profile
   # 刚才上传的文件依然存在！！！
   # 手动启动那个挂掉的NameNode
   sbin/hadoop-daemon.sh start namenode
   # 通过浏览器访问：http://192.168.1.201:50070
   # NameNode 'hadoop01:9000' (standby)
   ```

### 集群工作状态查询

**查看hdfs的各节点状态信息**

```shell
bin/hdfs dfsadmin -report	 
```

**获取一个namenode节点的HA状态**

```shell
bin/hdfs haadmin -getServiceState nn1	
```

**单独启动一个namenode进程**

```shell

```

**单独启动一个zkfc进程**

```shell
./hadoop-daemon.sh start zkfc 
```

## 集群运维测试

#### Datanode 动态上下线

Datanode动态上下线很简单，步骤如下：

1. 准备一台服务器，设置好环境

2. 部署hadoop的安装包，并同步集群配置

3. 联网上线，新datanode会自动加入集群

4. 如果是一次增加大批datanode，还应该做集群负载重均衡

#### Namenode 状态切换管理

**使用的命令**

```shell
hdfs  haadmin
```

**查看namenode工作状态**   

```xml
hdfs haadmin -getServiceState nn1
```

**将standby状态namenode切换到active**

```shell
hdfs haadmin –transitionToActive nn1
```

**将active状态namenode切换到standby**

```shell
hdfs haadmin –transitionToStandby nn2
```

#### 数据块的balance

启动balancer的命令：

```shell
start-balancer.sh -threshold 8
```

运行之后，会有Balancer进程出现：

![1579491296500](/img/1579491296500.png)

上述命令设置了Threshold为8%，那么执行balancer命令的时候，首先统计所有DataNode的磁盘利用率的均值，然后判断如果某一个DataNode的磁盘利用率超过这个均值Threshold，那么将会把这个DataNode的block转移到磁盘利用率低的DataNode，这对于新节点的加入来说十分有用。Threshold的值为1到100之间，不显示的进行参数设置的话，默认是10。

## HA 下的hdfs-api变化

客户端需要nameservice的配置信息，其他不变

```java
/**
 * 如果访问的是一个ha机制的集群
 * 则一定要把core-site.xml和hdfs-site.xml配置文件放在客户端程序的classpath下
 * 以让客户端能够理解hdfs://ns1/中  “ns1”是一个ha机制中的namenode对——nameservice
 * 以及知道ns1下具体的namenode通信地址
 * @author
 *
 */
public class UploadFile {
	
	public static void main(String[] args) throws Exception  {
		
		Configuration conf = new Configuration();
		conf.set("fs.defaultFS", "hdfs://ns1/");
		
		FileSystem fs = FileSystem.get(new URI("hdfs://ns1/"),conf,"hadoop");
		
		fs.copyFromLocalFile(new Path("g:/eclipse-jee-luna-SR1-linux-gtk.tar.gz"), new Path("hdfs://ns1/"));
		
		fs.close();
	}
}
```

