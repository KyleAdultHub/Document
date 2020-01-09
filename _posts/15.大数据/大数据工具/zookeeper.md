---
title: zookeeper
date: "2019-12-21 14:00:00"
categories:
- 大数据
- 大数据工具
tags:
- 大数据
toc: true
typora-root-url: ..\..\..
---

## Zookeeper 的简介

### zookeeper的作用

Zookeeper是一个分布式协调服务；就是为用户的分布式应用程序提供协调服务

主要用来给分布式节点提供数据保管和节点监听功能

### zookeeper特点

- zookeeper是为别的分布式程序服务的
- Zookeeper本身就是一个分布式程序（只要有半数以上节点存活，zk就能正常服务，一般要部署奇数台，最好大于等于三台）
- Zookeeper所提供的服务涵盖：主从协调、服务器节点动态上下线、统一配置管理、分布式共享锁、统一名称服务……
- 虽然说可以提供各种服务，但是zookeeper在底层其实只提供了两个功能：1. 管理(存储，读取)用户程序提交的数据；2. 为用户程序提供数据节点监听服务；

### zookeeper集群的角色：  

- Leader 
- follower  （Observer）

### zookeeper集群的选举机制（全新集群）

假设有五台服务器组成的zookeeper集群,它们的id从1-5,同时它们都是最新启动的,也就是没有历史数据,在存放数据量这一点上,都是一样的.假设这些服务器依序启动；

1. 服务器1启动,此时只有它一台服务器启动了,它发出去的报没有任何响应,所以它的选举状态一直是LOOKING状态
2. 服务器2启动,它与最开始启动的服务器1进行通信,互相交换自己的选举结果,由于两者都没有历史数据,所以id值较大的服务器2胜出,但是由于没有达到超过半数以上的服务器都同意选举它(这个例子中的半数以上是3),所以服务器1,2还是继续保持LOOKING状态.
3. 服务器3启动,根据前面的理论分析,服务器3成为服务器1,2,3中的老大,而与上面不同的是,此时有三台服务器选举了它,所以它成为了这次选举的leader.
4. 服务器4启动,根据前面的分析,理论上服务器4应该是服务器1,2,3,4中最大的,但是由于前面已经有半数以上的服务器选举了服务器3,所以它只能接收当小弟的命了.
5. 服务器5启动,同4一样,当小弟.

### zookeeper集群的选举机制（运行中集群）

运行中集群的选举机制需要加入数据id、leader id和逻辑时钟的概念。

**数据id**：数据新的id就大，数据每次更新都会更新id。

**Leader id**：就是我们配置的myid中的值，每个机器一个。

**逻辑时钟**：这个值从0开始递增,每次选举对应一个值,也就是说:  如果在同一次选举中,那么这个值应该是一致的 ;  逻辑时钟值越大,说明这一次选举leader的进程更新.

选举的标准就变成：

​		1、逻辑时钟小的选举结果被忽略，重新投票

​		2、统一逻辑时钟后，数据id大的胜出

​		3、数据id相同的情况下，leader id大的胜出

### zookeeper集群特性

- Zookeeper：一个leader，多个follower组成的集群
- 全局数据一致：每个server保存一份相同的数据副本，client无论连接到哪个server，数据都是一致的
- 分布式读写，更新请求转发，由leader实施
- 更新请求顺序进行，来自同一个client的更新请求按其发送顺序依次执行
- 数据更新原子性，一次数据更新要么成功，要么失败
- 实时性，在一定时间范围内，client能读到最新数据

### zookeeper的监听工作机制

![1577675698050](/img/1577675698050.png)

**原理:**  

监听器是一个接口，我们的代码中可以实现Wather这个接口，实现其中的process方法，方法中即我们自己的业务逻辑，当zookeeper检测到有事件发生时，就会进行事件回调，出发process方法

**监听器的注册是在获取数据的操作中实现：** 

getData(path,watch?)监听的事件是：节点数据变化事件

getChildren(path,watch?)监听的事件是：节点下的子节点增减变化事件

## zookeeper 的部署步骤

> 前提: 每台服务器安装好jdk, 3台以上的部署机

1. 下载zookeeper的压缩包(每台执行)

   ```shell
   wget  ....../zookeeper-3.4.5.tar.gz
   ```

2. 解压缩(每台执行)

   ```shell
   tar -zxvf -C /opt/apps  zookeeper-3.4.5.tar.gz
   mv /opt/apps/zookeeper-3.4.5 /opt/apps/zookeeper
   ```

3. 修改环境变量(每台执行)

   ```shell
   echo "export ZOOKEEEPER_HOME=/opt/apps/zookeeper"
   echo "export PATH=$PATH:$ZOOKEEPER_HOME/bin"
   source /etc/profile
   ```

4. 修改集群配置文件(每台执行)

   ```shell
   cp /opt/apps/zookeeper/conf/zoo_sample.cfg /opt/apps/zookeeper/conf/zoo.cfg
   echo "server.1=ip1:2888:3888"  # 1.2.3 代表zookeeper集群中的id
   echo "server.2=ip2:2888:3888"
   echo "server.3=ip3:2888:3888"
   
   vi /opt/apps/zookeeper/conf/zoo.cfg
   dataDir=/opt/apps/zookeeper/data  # 配置zookeeper数据目录
   dataLogDir=/opt/apps/zookeeper/log  # 配置zookeeper log目录
   ```

5. 创建数据文件夹(每台执行)

   ```shell
   mkdir -m 755 /opt/apps/zookeeper/data
   mkdir  -m 755 /opt/apps/zookeeper/
   ```

6. 创建myid文件(每台执行)

   ```shell
   cd data
   vi myid
   1       # 添加每台机器的id文件，对应cfg文件中的server.x
   ```

7. 启动zookeeper

   ```shell
   zkServer.sh start
   ```

8. 查看集群的状态

   ```shell
   jps（查看进程）
   zkServer.sh status（查看集群状态，主从信息）
   ```

## zookeeper数据结构

### 数据结构图

zookeeper是按照树状结构对数据进行存储的

![1576996483672](/img/1576996483672.png)

### 数据结构特点

- 层次化的目录结构，命名符合常规文件系统规范(见下图)
- 每个节点在zookeeper中叫做znode,并且其有一个唯一的路径标识
- 节点Znode可以包含数据和子节点（但是EPHEMERAL类型的节点不能有子节点，下一页详细讲解）
- 客户端应用可以在节点上设置监视器（后续详细讲解）	

### 节点类型

**znode的两种类型**

- 短暂(ephemeral) : 客户端断开后节点自动删除，节点不能有子节点
- 持久(persistent)：客户端断开后不删除节点

**znode四种目录形式(默认persistent)**

- persistent  :   持久节点
- persistent_sequential:  持久序列节点，znode名称后会附加一个值，顺序号是一个单调递增的计数器，由父节点维护(例: /test0000000019)， 
- ephemeral:        短暂节点
- ephemera_sequential:     短暂序列节点

**序列节点的作用**

在分布式系统中，顺序号可以被用于为所有的事件进行全局排序，这样客户端可以通过顺序号推断事件的顺序

## zookeeper 命令行操作

**启动客户端**

```shell
zkCli.sh  -server  <ip>  # 连接本机可以省略-server ip
```

**查看当前节点包含的所有节点**

```shell
[zk: 202.115.36.251:2181(CONNECTED) 1] ls /
```

**创建一个新节点**

```shell
# /zk 是节点名称   myData  是节点中保存的数据
# -e 创建短暂节点， -s 创建序列节点
[zk: 202.115.36.251:2181(CONNECTED) 1] create [-s] [-e] /zk "myData"
```

**获取节点数据**

```shell
# 当使用watch后缀时， 监听这个节点的变化,当另外一个客户端改变/zk时,会出发watch事件，客户端它会打印watch时间信息
[zk: 202.115.36.251:2181(CONNECTED) 3] get /zk  [watch]
```

**更改节点数据**

```shell
[zk: 202.115.36.251:2181(CONNECTED) 4] set /zk "zsl“
```

**删除节点**

```shell
[zk: 202.115.36.251:2181(CONNECTED) 5] delete /zk
[zk: 202.115.36.251:2181(CONNECTED) 5] rmr /zk   # 递归删除节点
```

## zookeeper java api 使用

org.apache.zookeeper.Zookeeper 是zookeeper客户端入口的主要类，负责建立与zk  server 的会话

### 主要方法

| create      | 在本地目录树中创建一个节点          |
| ----------- | ----------------------------------- |
| delete      | 删除一个节点                        |
| exists      | 测试本地是否存在目标节点            |
| get/setData | 从目标节点上读取 / 写数据           |
| get/setACL  | 获取 / 设置目标节点访问控制列表信息 |
| getChildren | 检索一个子节点上的列表              |
| sync        | 等待要被传送的数据                  |

### zookeeper简单使用代码示例

实现简单的增删改查和监听操作

```java
package com.bigdata.utils.zookeeper;

import org.apache.zookeeper.*;
import org.apache.zookeeper.ZooDefs.Ids;
import org.apache.zookeeper.data.Stat;
import org.junit.Before;
import org.junit.Test;

import java.util.List;

public class SimpleZkClient {

	private static final String connectString = "mini1:2181,mini2:2181,mini3:2181";
	private static final int sessionTimeout = 2000;

	private ZooKeeper zkClient = null;

	@Before
	public void init() throws Exception {
		zkClient = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
			@Override
			public void process(WatchedEvent event) {
				// 收到事件通知后的回调函数（应该是我们自己的事件处理逻辑）
				System.out.println(event.getType() + "---" + event.getPath());
				try {
					zkClient.getChildren("/", true);
				} catch (Exception e) {
				}
			}
		});

	}

	/**
	 * 数据的增删改查
	 * 
	 * @throws InterruptedException
	 * @throws KeeperException
	 */

	// 创建数据节点到zk中
	public void testCreate() throws KeeperException, InterruptedException {
		// 参数1：要创建的节点的路径 参数2：节点大数据 参数3：节点的权限 参数4：节点的类型
		String nodeCreated = zkClient.create("/eclipse", "hellozk".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
		//上传的数据可以是任何类型，但都要转成byte[]
	}

	//判断znode是否存在
	@Test	
	public void testExist() throws Exception{
		Stat stat = zkClient.exists("/eclipse", false);
		System.out.println(stat==null?"not exist":"exist");
		
		
	}
	
	// 获取子节点
	@Test
	public void getChildren() throws Exception {
		List<String> children = zkClient.getChildren("/", true);
		for (String child : children) {
			System.out.println(child);
		}
		Thread.sleep(Long.MAX_VALUE);
	}

	//获取znode的数据
	@Test
	public void getData() throws Exception {
		
		byte[] data = zkClient.getData("/eclipse", false, null);
		System.out.println(new String(data));
		
	}
	
	//删除znode
	@Test
	public void deleteZnode() throws Exception {
		
		//参数2：指定要删除的版本，-1表示删除所有版本
		zkClient.delete("/eclipse", -1);
		
		
	}
	//删除znode
	@Test
	public void setData() throws Exception {
		// -1 代表删除所有节点版本
		zkClient.setData("/app1", "imissyou angelababy".getBytes(), -1);
		
		byte[] data = zkClient.getData("/app1", false, null);
		System.out.println(new String(data));
		
	}
}
```

### 分布式系统使用zookeeper进行协调示例

实现功能: 分布式系统中，有多台服务器和客户端，服务端可以动态的上下线，要实现客户端可以动态的感知服务节点的变化和最新状态

**服务端代码示例**

```java
package com.bigdata.utils.zookeeper;

import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooDefs.Ids;
import org.apache.zookeeper.ZooKeeper;

public class DistributedServer {
	private static final String connectString = "mini1:2181,mini2:2181,mini3:2181";
	private static final int sessionTimeout = 2000;
	private static final String parentNode = "/servers";

	private ZooKeeper zk = null;

	/**
	 * 创建到zk的客户端连接
	 * 
	 * @throws Exception
	 */
	public void getConnect() throws Exception {

		zk = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
			@Override
			public void process(WatchedEvent event) {
				// 收到事件通知后的回调函数（应该是我们自己的事件处理逻辑）
				System.out.println(event.getType() + "---" + event.getPath());
				try {
					zk.getChildren("/", true);
				} catch (Exception e) {
				}
			}
		});

	}

	/**
	 * 向zk集群注册服务器信息
	 * 
	 * @param hostname
	 * @throws Exception
	 */
	public void registerServer(String hostname) throws Exception {

		String create = zk.create(parentNode + "/server", hostname.getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
		System.out.println(hostname + "is online.." + create);

	}

	/**
	 * 业务功能
	 * 
	 * @throws InterruptedException
	 */
	public void handleBussiness(String hostname) throws InterruptedException {
		System.out.println(hostname + "start working.....");
		Thread.sleep(Long.MAX_VALUE);
	}

	public static void main(String[] args) throws Exception {

		// 获取zk连接
		DistributedServer server = new DistributedServer();
		server.getConnect();

		// 利用zk连接注册服务器信息
		server.registerServer(args[0]);

		// 启动业务功能
		server.handleBussiness(args[0]);

	}
}
```

**客户端代码示例**

```java
package com.bigdata.utils.zookeeper;

import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;

import java.util.ArrayList;
import java.util.List;

public class DistributedClient {

	private static final String connectString = "mini1:2181,mini2:2181,mini3:2181";
	private static final int sessionTimeout = 2000;
	private static final String parentNode = "/servers";
	// 注意:加volatile的意义何在？
	private volatile List<String> serverList;
	private ZooKeeper zk = null;

	/**
	 * 创建到zk的客户端连接
	 * 
	 * @throws Exception
	 */
	public void getConnect() throws Exception {

		zk = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
			@Override
			public void process(WatchedEvent event) {
				// 收到事件通知后的回调函数（应该是我们自己的事件处理逻辑）
				try {
					//重新更新服务器列表，并且注册了监听
					getServerList();

				} catch (Exception e) {
				}
			}
		});

	}

	/**
	 * 获取服务器信息列表
	 * 
	 * @throws Exception
	 */
	public void getServerList() throws Exception {

		// 获取服务器子节点信息，并且对父节点进行监听
		List<String> children = zk.getChildren(parentNode, true);

		// 先创建一个局部的list来存服务器信息
		List<String> servers = new ArrayList<String>();
		for (String child : children) {
			// child只是子节点的节点名
			byte[] data = zk.getData(parentNode + "/" + child, false, null);
			servers.add(new String(data));
		}
		// 把servers赋值给成员变量serverList，已提供给各业务线程使用
		serverList = servers;

		//打印服务器列表
		System.out.println(serverList);
		
	}

	/**
	 * 业务功能
	 * 
	 * @throws InterruptedException
	 */
	public void handleBussiness() throws InterruptedException {
		System.out.println("client start working.....");
		Thread.sleep(Long.MAX_VALUE);
	}
	
	public static void main(String[] args) throws Exception {

		// 获取zk连接
		DistributedClient client = new DistributedClient();
		client.getConnect();
		// 获取servers的子节点信息（并监听），从中获取服务器信息列表
		client.getServerList();

		// 业务线程启动
		client.handleBussiness();
		
	}
}
```

