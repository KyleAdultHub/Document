---
title: hadoop 介绍、安装
date: "2019-12-29 14:00:00"
categories:
- 大数据
- hadoop
tags:
- 大数据
- HADOOP
toc: true
typora-root-url: ..\..\..
---

## hadoop 介绍

### 什么是hadoop

- HADOOP是apache旗下的一套开源**软件平台**

- HADOOP提供的功能：利用服务器集群，根据用户的自定义业务逻辑，**对海量数据进行分布式处理**

###  hadoop 的核心组件

1. HDFS（分布式文件系统）
2. YARN（运算资源调度系统）
3. MAPREDUCE（分布式运算编程框架）
4. 广义上来说，HADOOP通常是指一个更广泛的概念——HADOOP生态圈

### hadoop 的产生背景

1. HADOOP**最早起源于Nutch**。Nutch的设计目标是构建一个大型的全网搜索引擎，包括网页抓取、索引、查询等功能，但随着抓取网页数量的增加，**遇到了严重的可扩展性问题——**如何解决数十亿网页的存储和索引问题。
2. 2003年、2004年**谷歌发表的两篇论文为该问题提供了可行的解决方案**。
   - 分布式文件系统（GFS），可用于处理海量网页的**存储**
   - 分布式计算框架MAPREDUCE(YARN+MAPREDUCE)，可用于处理海量网页的**索引计算**问题。

3. Nutch的开发人员完成了相应的**开源实现HDFS和MAPREDUCE**，并从Nutch中剥离成为独立项目HADOOP，到2008年1月，HADOOP成为Apache顶级项目，迎来了它的快速发展期。

### hadoop生态圈组件

![1577615377073](/img/1577615377073.png)

**HDFS**: 分布式文件系统

**MAPREDUCE**: 分布式运算程序开发框架

**HIVE**： 基于大数据技术(文件系统/运算框架)的SQL数据仓库工具

**HBASE**： 基于HADOOP的分布式海量数据库

**ZOOKEEPER**: 分布式协调服务基础组件

**Mahout**:  基于mapreduce/spark/flink  等分布式运算框架的机器学习算法库

**Oozie**:  工作流调度框架

**Sqoop**：数据导入导出工具

**Flume**： 日志数据采集框架

## 离线大数据分析流程介绍

### 流程图分析

![1577615860118](/img/1577615860118.png)

1) 数据采集：定制开发采集程序，或使用开源框架FLUME

2) 数据预处理：定制开发mapreduce程序运行于hadoop集群

3) 数据仓库技术：基于hadoop之上的Hive

4) 数据导出：基于hadoop的sqoop数据导入导出工具

5) 数据可视化：定制开发web程序或使用kettle等产品

6) 整个过程的流程调度：hadoop生态圈中的oozie工具或其他类似开源产品

### 项目技术架构图

![1577615959629](/img/1577615959629.png)

### 推荐系统架构示例

![1577615996895](/img/1577615996895.png)

## hadoop集群搭建

### 集群简介

HADOOP集群具体来说包含两个集群：HDFS集群和YARN集群，两者逻辑上分离，但物理上常在一起，因为YARN需要分配资源对数据进行处理，所以最好的方式也是将YARN和HDFS保持物理地址统一

**HDFS集群：**

负责海量数据的存储，集群中的角色主要有 NameNode / DataNode

**YARN集群：**

负责海量数据运算时的资源调度，集群中的角色主要有 ResourceManager /NodeManager

**mapredice**：

它其实是一个应用程序开发包)

接下来的集群示例，以3节点为例进行搭建:  hosts 文件示例

```shell
hdp-node-01    	NameNode  SecondaryNameNode
hdp-node-02    	ResourceManager 
hdp-node-03		DataNode    NodeManager
```

### 环境准备

**服务器**

Centos  6.5  64bit

**网络**

网络环境需要互通

**同步时间**

设置时间同步服务器ntp  或者机器之间同步执行  date -s "2019-06-06 00:00:00"

**服务器系统设置**

1. 添加hadoop用户,  并给sudo权限

   ```shell
   useradd hadoop
   passwd hadoop
   vi /etc/sudoers   
   # 添加  hadoop   ALL=(ALL)  ALL 
   ```

2. 设置主机名

   ```shell
   hdp-node-01
   hdp-node-02
   hdp-node-03
   ```

3. 配置域名映射, 修改host文件

   ```shell
   192.168.33.101          hdp-node-01
   192.168.33.102          hdp-node-02
   192.168.33.103          hdp-node-03
   ```

4. 配置ssh免密登录

   ```shell
   ssh-keygen
   ssh-copy-id hdp-node-01
   ssh-copy-id hdp-node-02
   ssh-copy-id hdp-node-03
   ```

5. 关闭防火墙

   ```shell
   systemctl stop firewalld
   systemctl disable firewalld
   ```

**jdk安装**

上传jdk安装包

规划安装目录  /home/hadoop/apps/jdk_1.7.65

解压安装包

配置环境变量 /etc/profile， 添加

```shell
export JAVA_HOME=/usr/local/java/jdk1.8.0_141
export JRE_HOME=${JAVA_HOME}/jre 
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib 
export  PATH=${JAVA_HOME}/bin:$PATH
```

### hadoop集群安装

1. **上传安装包并解压**

   - 上传hadoop安装包, 并解压到一个自定义目录（该安装包通过编译生成，编译方法参考**hadoop编译**）

     ```shell
     tar -xzvf hadoop-2.6.1.tar.gz -C /home/hadoop/apps
     ```

2. **修改配置文件**

   **etc/hadoop/hadoop-env.sh**  hadoop 环境变量配置

   修改JAVA_HOME 的位置, 如果不通过ssh连接执行hadoop命令可以不改

   ```shell
   # The java implementation to use.
   export JAVA_HOME=/home/hadoop/apps/jdk1.7.0_51
   ```

   **core-site.xml**  hadoop基础配置

   配置使用文件系统，和nameNode的地址，还有hadoop的系统数据目录

   ```xml
   <configuration>
   <property>
   <name>fs.defaultFS</name>
   <value>hdfs://hdp-node-01:9000</value>
   </property>
   <property>
   <name>hadoop.tmp.dir</name>
   <value>/home/HADOOP/apps/hadoop-2.6.1/tmp</value>
   </property>
   </configuration>
   ```

   **hdfs-site.xml**  hdfs相关配置

   配置hdfs的数据目录, 副本数量, secondaryNameNode 的http地址等

   ```xml
   <configuration>
   <property>
   <name>dfs.namenode.name.dir</name>
   <value>/home/hadoop/data/name</value>
   </property>
   <property>
   <name>dfs.datanode.data.dir</name>
   <value>/home/hadoop/data/data</value>
   </property>
   
   <property>
   <name>dfs.replication</name>
   <value>3</value>
   </property>
   
   <property>
   <name>dfs.secondary.http.address</name>
   <value>hdp-node-01:50090</value>
   </property>
   </configuration>
   ```

   **mapreduce.framework.name**  mapreduce计算框架相关配置

   配置使用的资源调度工具配置等

   ```xml
   <configuration>
   <property>
   <name>mapreduce.framework.name</name>
   <value>yarn</value>
   </property>
   </configuration>
   ```

   **yarn-site.xml**   yarn资源调度器相关配置

   配置yarn的resource Manager和nodeManager

   ```xml
   <configuration>
   <property>
   <name>yarn.resourcemanager.hostname</name>
   <value>hadoop01</value>
   </property>
   
   <property>
   <name>yarn.nodemanager.aux-services</name>
   <value>mapreduce_shuffle</value>
   </property>
   </configuration>
   ```

   **slaves**  集群配置文件

   ```shell
   hdp-node-01
   hdp-node-02
   hdp-node-03
   ```

3. **启动集群**

   **格式化hdfs**

   格式化的作用是初始化namenode 的工作目录

```shell
bin/hadoop  namenode  -format
```

​	**启动HDFS和YARN**

​	第一种方法:  hadoop-daemon  启动

​	每次只会启动一个组件，不自动化，比较麻烦

```shell
sbin/hadoop-daemon.sh start namenode           # 在namenode执行
sbin/hadoop-daemon.sh start datanode           # 在datanode执行
sbin/hadoop-daemon.sh start secondarynamenode  # 启动secondarynamenode
sbin/yarn-daemon.sh start resourcemanager   # 在resourcemanager节点启动
sbin/yarn-daemon.sh start nodemanager   # 在nodemanager节点启动
```

​	第二种方法:  start-dfs  / start-yarn 启动

```shell
sbin/start-dfs.sh      # 会把dfs集群都启动起来
sbin/start-yarn.sh        # 直接把yarn集群都启动起来
```

​	第三种方法: start-all 启动

```shell
sbin/start-all.sh    # 直接把dfs 和yarn 集群都启动起来
```

## HDFS 集群状态查询

1. 查看集群状态

   命令

   ```
   hdfs  dfsadmin  –report 
   ```

   示例

   ![1577619270058](/img/1577619270058.png)

   ​		可以看出，集群共有3个datanode可用

2. 也可打开web控制台查看HDFS集群信息

   在浏览器打开<http://hdp-node-01:50070/>

   示例

   ![1577619307817](/img/1577619307817.png)