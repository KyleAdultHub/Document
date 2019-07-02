---
title: GlusterFS 原理及使用
date: "2019-07-01 17:00:00"
categories:
- 数据存储
- GlusterFS
tags:
- glusterfs
toc: true
typora-root-url: ..\..\..\
---

### Gluster 简介

#### Gluster 概述

- Glusterfs是一个开源的分布式文件系统； 是Scale存储的核心,能够处理千数量级的客户端.在传统的解决方案中Glusterfs能够灵活的结合物理的,虚拟的和云资源去体现高可用和企业级的性能存储.
- Glusterfs通过TCP/IP或InfiniBand RDMA网络链接将客户端的存储资块源聚集在一起； 使用单一的全局命名空间来管理数据,磁盘和内存资源.
- Glusterfs基于堆叠的用户空间设计,可以为不同的工作负载提供高优的性能.
- Glusterfs支持运行在任何标准IP网络上标准应用程序的标准客户端，用户可以在全局统一的命名空间中使用NFS/CIFS等标准协议来访问应用数据.

#### Gluster 主要特征

##### 扩展性和高性能

GlusterFS利用双重特性来提供几TB至数PB的高扩展存储解决方案。Scale-Out架构允许通过简单地增加资源来提高存储容量和性能，磁盘、计算和I/O资源都可以独立增加，支持10GbE和InfiniBand等高速网络互联。Gluster弹性哈希（Elastic Hash）解除了GlusterFS对元数据服务器的需求，消除了单点故障和性能瓶颈，真正实现了并行化数据访问。

##### 高可用性

GlusterFS可以对文件进行自动复制，如镜像或多次复制，从而确保数据总是可以访问，甚至是在硬件故障的情况下也能正常访问。自我修复功能能够把数据恢复到正确的状态，而且修复是以增量的方式在后台执行，几乎不会产生性能负载。GlusterFS没有设计自己的私有数据文件格式，而是采用操作系统中主流标准的磁盘文件系统（如EXT3、ZFS）来存储文件，因此数据可以使用各种标准工具进行复制和访问。

##### 全局统一命名空间

全局统一命名空间将磁盘和内存资源聚集成一个单一的虚拟存储池，对上层用户和应用屏蔽了底层的物理硬件。存储资源可以根据需要在虚拟存储池中进行弹性扩展，比如扩容或收缩。当存储虚拟机映像时，存储的虚拟映像文件没有数量限制，成千虚拟机均通过单一挂载点进行数据共享。虚拟机I/O可在命名空间内的所有服务器上自动进行负载均衡，消除了SAN环境中经常发生的访问热点和性能瓶颈问题。

##### 弹性哈希算法

GlusterFS采用弹性哈希算法在存储池中定位数据，而不是采用集中式或分布式元数据服务器索引。在其他的Scale-Out存储系统中，元数据服务器通常会导致I/O性能瓶颈和单点故障问题。GlusterFS中，所有在Scale-Out存储配置中的存储系统都可以智能地定位任意数据分片，不需要查看索引或者向其他服务器查询。这种设计机制完全并行化了数据访问，实现了真正的线性性能扩展。

##### 基于标准协议

Gluster存储服务支持NFS, CIFS, HTTP, FTP以及Gluster原生协议，完全与POSIX标准兼容。现有应用程序不需要作任何修改或使用专用API，就可以对Gluster中的数据进行访问。这在公有云环境中部署Gluster时非常有用，Gluster对云服务提供商专用API进行抽象，然后提供标准POSIX接口。

#### 显著优点

- 开源，且网上的文档基本上足够你维护一套简单的存储系统。
- 支持多客户端处理，当前已知的支持千级别及以上规模客户。
- 支持POSIX<简单来说，软件跨平台，也可以看找的其他博客说明，见‘参考文件’>。
- 支持各种低端硬件，当然并不推崇，但是可以实现。我在线上用过8核，32G，依然跑的很溜。
- 支持NFS/SMB/CIFS/GLUSTERFS等行业标准协议访问。
- 提供各种优秀的功能，如磁盘配额、复制式、分布式、快照、性能检测命令等。
- 支持大容量存储，当前已知的支持PB及以上规模存储。
- 可以使用任何支持扩展属性的ondisk文件系统，比如xattr.
- 强大且简单的扩展能力，你可以在任何时候快速扩容。
- 自动配置故障转移，在故障发生时，无需任何操作即可恢复数据，且最新副本依然从仍在运行的节点获取。

#### 术语介绍

**Brick**:    GFS中的存储单元，通常是一个受信存储池中的服务器的一个导出目录。可以通过主机名和目录名来标识，如'SERVER:EXPORT'

**Client**:     挂载了GFS卷的设备

**Extended Attributes**:    xattr是一个文件系统的特性，其支持用户或程序关联文件/目录和元数据。

**FUSE**:    Filesystem Userspace是一个可加载的内核模块，其支持非特权用户创建自己的文件系统而不需要修改内核代码。通过在用户空间运行文件系统的代码通过FUSE代码与内核进行桥接。

**GFID**:    GFS卷中的每个文件或目录都有一个唯一的128位的数据相关联，其用于模拟inode

**Namespace**:    每个Gluster卷都导出单个ns作为POSIX的挂载点

**Node**:    一个拥有若干brick的设备

**RDMA**:    远程直接内存访问，支持不通过双方的OS进行直接内存访问。

**RRDNS**:    round robin DNS是一种通过DNS轮转返回不同的设备以进行负载均衡的方法

**Self-heal**:    用于后台运行检测复本卷中文件和目录的不一致性并解决这些不一致。

**Split-brain**:     脑裂

**Volfile**:    glusterfs进程的配置文件，通常位于/var/lib/glusterd/vols/volname

**Volume**:    一组bricks的逻辑集合

#### 工作原理

![1561981137392](/img/1561981137392.png)

1. 首先是在客户端， 用户通过glusterfs的mount point 来读写数据， 对于用户来说，集群系统的存在对用户是完全透明的，用户感觉不到是操作本地系统还是远端的集群系统。
2. 用户的这个操作被递交给 本地linux系统的VFS来处理。
3. VFS 将数据递交给FUSE 内核文件系统:在启动 glusterfs 客户端以前，需要想系统注册一个实际的文件系统FUSE,如上图所示，该文件系统与ext3在同一个层次上面， ext3 是对实际的磁盘进行处理， 而fuse 文件系统则是将数据通过/dev/fuse 这个设备文件递交给了glusterfs client端。所以， 我们可以将 fuse文件系统理解为一个代理。
4. 数据被fuse 递交给Glusterfs client 后， client 对数据进行一些指定的处理（所谓的指定，是按照client 配置文件据来进行的一系列处理， 我们在启动glusterfs client 时需要指定这个文件。
5. 在glusterfs client的处理末端，通过网络将数据递交给 Glusterfs Server，并且将数据写入到服务器所控制的存储设备上。

#### GlusterFS 整体工作流

![1561985936932](/img/1561985936932.png)

1，只要安装了glusterFS，会创建一个gluster管理守护进程(glusterd)二进制文件，该守护进程应该在集群中的所有设备上运行。

2，启动glusterd后，可以创建一个由所有存储服务器节点组成的受信任的服务器池(TSP可以包含单个节点)

3，作为基本存储单元的brick可以再这些服务器中作为导出目录创建。

4，从这个TSP的任何数量的brick都可以被联合起来形成一个整体。

5，一旦创建了volume，glusterfsd进程就会开始在每个参与的brick中运行，除此之外，将在/var/lib/glusterd/vols/生成为vol文件的配置文件。

6，volume中将会有与每个brick对应的配置文件，文件中包含关于特定brick的所有内容。

7，客户端进程所需的配置文件也将被创建。

8，文件系统完成，挂载使用。

9，挂载的IP/主机名可以是受信任服务器池中创建所需volume的任何节点的IP/主机名。

10，当我们在客户端安装volume时，客户端glusterfs进程与服务器的glusterd进程进行通信。

11，服务器glusterd进程发送一个配置文件(vol)文件，其中包括客户端转换器列表，另一个包含volume中每个brick的信息。

12，借助于该文件，客户端glusterfs进程可以与每个brick的glusterfsd进行通信。

13，当挂载的文件系统中，客户端发出系统调用<文件操作或Fop或打开文件>时，

14，VFS识别文件系统类型为glusterfs，会将请求发送到FUSE内核模块。

15，FUSE内核模块将通过/dev/fuse将其发送到客户机节点的用户空间中的glusterFS。

16，客户端上的glusterFS进程由一堆成为客户端翻译器的翻译器组成。这些翻译器在存储服务器glusterd进程发送的配置文件(vol文件)中定义。

17，这些翻译器中的第一个由FUSE库(libfuse)组成的FUSE翻译器。

18，每个翻译器都具有与每个文件操作对应的功能或glusterfs支持的fop。

19，该请求将在每个翻译器中发挥相应的功能。

#### 常用volume 卷类型

##### 分布（distributed）

默认模式，既DHT, 也叫 分布卷: 将文件已hash算法随机分布到 一台服务器节点中存储。

![1561981331786](/img/1561981331786.png)

##### 复制（replicate）

复制模式，既AFR, 创建volume 时带 replica x 数量: 将文件复制到 replica x 个节点中。

![1561981392564](/img/1561981392564.png)

##### 条带（striped）

条带模式，既Striped, 创建volume 时带 stripe x 数量： 将文件切割成数据块，分别存储到 stripe x 个节点中 ( 类似raid 0 )。

![1561981455113](/img/1561981455113.png)

##### 分布式条带卷(distribute stripe volume)

分布式条带模式（组合型），最少需要4台服务器才能创建。 创建volume 时 stripe 2 server = 4 个节点： 是DHT 与 Striped 的组合型。

![1561981508738](/img/1561981508738.png)

##### 分布式复制卷(distribute replica volume)

分布式复制模式（组合型）, 最少需要4台服务器才能创建。 创建volume 时 replica 2 server = 4 个节点：是DHT 与 AFR 的组合型。

![1561981580578](/img/1561981580578.png)

##### 条带复制卷(stripe replica volume)

条带复制卷模式（组合型）, 最少需要4台服务器才能创建。 创建volume 时 stripe 2 replica 2 server = 4 个节点： 是 Striped 与 AFR 的组合型。

![1561981599397](/img/1561981599397.png)

##### 分布式条带复制卷(distribute stripe replicavolume)

 三种模式混合, 至少需要8台 服务器才能创建。 stripe 2 replica 2 , 每4个节点 组成一个 组。

![1561981659019](/img/1561981659019.png)

### 环境准备

#### 前置条件

1. 至少需要两台设备/节点。
2. 具备正常的网络连接
3. 至少需要两个磁盘，一个用于OS安装，一个用于服务gluster存储；

#### 环境配置

**关闭防火墙**

```shell
systemctl stop firewalld.service       #停止firewalld
systemctl disable firewalld.service    #禁止firewalld开机自启
```

**关闭SELinux**

```shell
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config   #关闭SELinux
setenforce 0
getenforce
```

**配置host文件**

```shell
10.0.0.101  node01
10.0.0.102  node02
10.0.0.103  node03
10.0.0.104  node04
```

**同步时间**

```shell
ntpdate time.windows.com   #同步时间
```

**安装 gluster 源 **

```shell
yum  -y  install centos-release-gluster312.noarch
```

**安装glusterfs**

```shell
yum -y --enablerepo=centos-gluster*-test install glusterfs-server glusterfs-cli glusterfs-geo-replication
```

**启动gluster， 并设置开机自启动**

```shell
glusterfs -v
systemctl start glusterd.service
systemctl enable glusterd.service
systemctl status glusterd.service
netstat -lntup | grep gluster
```

**格式化磁盘(全部gluster )**

在每台主机上创建几块硬盘，做接下来的分布式存储使用

注：创建的硬盘要用xfs格式来格式化硬盘，如果用ext4来格式化硬盘的话，对于大于16TB空间格式化就无法实现了。所以这里要用xfs格式化磁盘(centos7默认的文件格式就是xfs)，并且xfs的文件格式支持PB级的数据量

```shell
fdisk -l   # 查看所有磁盘设备
mkfs.xfs  -i size=512 /dev/sdb    # 格式化磁盘, 一块磁盘相当于一个brick
mkfs.xfs  -i size=512 /dev/sdc
mkfs.xfs  -i size=512 /dev/sdd
mkdir -p /data/brick{1..3}    # 创建挂载块设备的目录
echo '/dev/sdb /data/brick1 xfs defaults 0 0' >> /etc/fstab
echo '/dev/sdc /data/brick2 xfs defaults 0 0' >> /etc/fstab
echo '/dev/sdd /data/brick3 xfs defaults 0 0' >> /etc/fstab
mount -a 
```

**将server 主机加入到信任池**

随便在一个开启glusterfs服务的主机上将其他主机加入到一个信任的主机池里，这里选择node01

```shell
gluster peer probe node01
gluster peer probe node02
gluster peer probe node03
gluster peer probe node04
gluster peer detach xxx # 用来将节点从信任池中删除
gluster peer status  # 查看信任主机池状态
# 当客户端需要挂载卷的时候，也需要加入到信任主机池中
```

### gluster 卷 操作

#### GlusterFS 五种卷的创建注意　　

- Distributed：分布式卷，文件通过 hash 算法随机分布到由 bricks 组成的卷上。
- Replicated: 复制式卷，类似 RAID 1，replica 数必须等于 volume 中 brick 所包含的存储服务器数，可用性高。
- Striped: 条带式卷，类似 RAID 0，stripe 数必须等于 volume 中 brick 所包含的存储服务器数，文件被分成数据块，以 Round Robin 的方式存储在 bricks 中，并发粒度是数据块，大文件性能好。
- Distributed Striped: 分布式的条带卷，volume中 brick 所包含的存储服务器数必须是 stripe 的倍数（>=2倍），兼顾分布式和条带式的功能。
- Distributed Replicated: 分布式的复制卷，volume 中 brick 所包含的存储服务器数必须是 replica 的倍数（>=2倍），兼顾分布式和复制式的功能。

　分布式复制卷的brick顺序决定了文件分布的位置，一般来说，先是两个brick形成一个复制关系，然后两个复制关系形成分布。

　企业一般用后两种，大部分会用分布式复制（可用容量为 总容量/复制份数），通过网络传输的话最好用万兆交换机，万兆网卡来做。这样就会优化一部分性能。它们的数据都是通过网络来传输的。

#### 配置分布式卷

```shell
#在信任的主机池中任意一台设备上创建卷都可以，而且创建好后可在任意设备挂载后都可以查看
[root@node01 ~] gluster volume create gv1 node01:/data/brick1 node02:/data/brick1 force      #创建分布式卷
volume create: gv1: success: please start the volume to access data
[root@node01 ~] gluster volume start gv1    #启动卷gv1
volume start: gv1: success
[root@node01 ~] gluster volume info gv1     #查看gv1的配置信息
 
Volume Name: gv1
Type: Distribute  #分布式卷
Volume ID: 85622964-4b48-47d5-b767-d6c6f1e684cc
Status: Started
Snapshot Count: 0
Number of Bricks: 2
Transport-type: tcp
Bricks:
Brick1: node01:/data/brick1
Brick2: node02:/data/brick1
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
[root@node01 ~] mount -t glusterfs 127.0.0.1:/gv1 /opt   #挂载gv1卷
[root@node01 ~] df -h
文件系统        容量  已用  可用 已用% 挂载点/dev/sdc        5.0G   33M  5.0G    1% /data/brick2
/dev/sdd        5.0G   33M  5.0G    1% /data/brick3
/dev/sdb        5.0G   33M  5.0G    1% /data/brick1
127.0.0.1:/gv1   10G   65M   10G    1% /opt     # 两个设备容量之和
[root@node01 ~] cd /opt/
[root@node01 opt] touch {a..f}      #创建测试文件
[root@node01 opt] ll
总用量 0
-rw-r--r--. 1 root root 0 2月   2 23:59 a
-rw-r--r--. 1 root root 0 2月   2 23:59 b
-rw-r--r--. 1 root root 0 2月   2 23:59 c
-rw-r--r--. 1 root root 0 2月   2 23:59 d
-rw-r--r--. 1 root root 0 2月   2 23:59 e
-rw-r--r--. 1 root root 0 2月   2 23:59 f
# 在node04也可看到新创建的文件，信任存储池中的每一台主机挂载这个卷后都可以看到
[root@node04 ~] mount -t glusterfs 127.0.0.1:/gv1 /opt
[root@node04 ~] ll /opt/
总用量 0
-rw-r--r--. 1 root root 0 2月   2 2018 a
-rw-r--r--. 1 root root 0 2月   2 2018 b
-rw-r--r--. 1 root root 0 2月   2 2018 c
-rw-r--r--. 1 root root 0 2月   2 2018 d
-rw-r--r--. 1 root root 0 2月   2 2018 e
-rw-r--r--. 1 root root 0 2月   2 2018 f
[root@node01 opt] ll /data/brick1/
总用量 0
-rw-r--r--. 2 root root 0 2月   2 23:59 a
-rw-r--r--. 2 root root 0 2月   2 23:59 b
-rw-r--r--. 2 root root 0 2月   2 23:59 c
-rw-r--r--. 2 root root 0 2月   2 23:59 e
[root@node02 ~] ll /data/brick1
总用量 0
-rw-r--r--. 2 root root 0 2月   2 23:59 d
-rw-r--r--. 2 root root 0 2月   2 23:59 f
#文件实际存在位置node01和node02上的/data/brick1目录下,通过hash分别存到node01和node02上的分布式磁盘上
```

#### 配置复制卷

复制模式，既AFR, 创建volume 时带 replica x 数量: 将文件复制到 replica x 个节点中。

```shell
# 这条命令的意思是使用Replicated的方式，建立一个名为gv2的卷(Volume)，存储块(Brick)为2个，分别为node01:/data/brick2  和  node02:/data/brick2；
# fore为强制创建：因为复制卷在双方主机通信有故障再恢复通信时容易发生脑裂。本次为实验环境，生产环境不建议使用。
[root@node01 ~] gluster volume create gv2 replica 2 node01:/data/brick2 node02:/data/brick2 force  
volume create: gv2: success: please start the volume to access data
[root@node01 ~] gluster volume start gv2    #启动gv2卷
volume start: gv2: success
[root@node01 ~] gluster volume info gv2   #查看gv2信息
Volume Name: gv2
Type: Replicate   #复制卷
Volume ID: 9f33bd9a-7096-4749-8d91-1e6de3b50053
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: node01:/data/brick2
Brick2: node02:/data/brick2
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
[root@node01 ~] mount -t glusterfs 127.0.0.1:/gv2 /mnt
[root@node01 ~] df -h
文件系统        容量  已用  可用 已用% 挂载点
/dev/sdc        5.0G   33M  5.0G    1% /data/brick2
/dev/sdd        5.0G   33M  5.0G    1% /data/brick3
/dev/sdb        5.0G   33M  5.0G    1% /data/brick1
127.0.0.1:/gv1   10G   65M   10G    1% /opt
127.0.0.1:/gv2  5.0G   33M  5.0G    1% /mnt    #容量是总容量的一半
[root@node01 ~] cd /mnt/
[root@node01 mnt] touch {1..6}
[root@node01 mnt] ll /data/brick2
总用量 0
-rw-r--r--. 2 root root 0 2月   3 01:06 1
-rw-r--r--. 2 root root 0 2月   3 01:06 2
-rw-r--r--. 2 root root 0 2月   3 01:06 3
-rw-r--r--. 2 root root 0 2月   3 01:06 4
-rw-r--r--. 2 root root 0 2月   3 01:06 5
-rw-r--r--. 2 root root 0 2月   3 01:06 6
[root@node02 ~] ll /data/brick2
总用量 0
-rw-r--r--. 2 root root 0 2月   3 01:06 1
-rw-r--r--. 2 root root 0 2月   3 01:06 2
-rw-r--r--. 2 root root 0 2月   3 01:06 3
-rw-r--r--. 2 root root 0 2月   3 01:06 4
-rw-r--r--. 2 root root 0 2月   3 01:06 5
-rw-r--r--. 2 root root 0 2月   3 01:06 6
# 创建文件的实际存在位置为node01和node02上的/data/brick2目录下，因为是复制卷，这两个目录下的内容是完全一致的。
```

#### 配置条带卷

```shell
[root@node01 ~] gluster volume create gv3 stripe 2 node01:/data/brick3 node02:/data/brick3 force
volume create: gv3: success: please start the volume to access data
[root@node01 ~] gluster volume start gv3
volume start: gv3: success
[root@node01 ~] gluster volume info gv3
 
Volume Name: gv3
Type: Stripe
Volume ID: 54c16832-6bdf-42e2-81a9-6b8d7b547c1a
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: node01:/data/brick3
Brick2: node02:/data/brick3
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
[root@node01 ~] mkdir /data01
[root@node01 ~] mount -t glusterfs 127.0.0.1:/gv3 /data01
[root@node01 ~] df -h
文件系统        容量  已用  可用 已用% 挂载点
127.0.0.1:/gv3   10G   65M   10G    1% /data01
[root@node01 ~] dd if=/dev/zero bs=1024 count=10000 of=/data01/10M.file
[root@node01 ~] dd if=/dev/zero bs=1024 count=20000 of=/data01/20M.file
[root@node01 ~] ll /data01/ -h
总用量 30M
-rw-r--r--. 1 root root 9.8M 2月   3 02:03 10M.file
-rw-r--r--. 1 root root  20M 2月   3 02:04 20M.file
*************************************************************************************
#文件的实际存放位置：
[root@node01 ~] ll -h /data/brick3
总用量 15M
-rw-r--r--. 2 root root 4.9M 2月   3 02:03 10M.file
-rw-r--r--. 2 root root 9.8M 2月   3 02:03 20M.file
[root@node02 ~] ll -h /data/brick3
总用量 15M
-rw-r--r--. 2 root root 4.9M 2月   3 02:03 10M.file
-rw-r--r--. 2 root root 9.8M 2月   3 02:04 20M.file
# 上面可以看到 10M 20M 的文件分别分成了 2 块（这是条带的特点），写入的时候是循环地一点一点在node01和node02的磁盘上.

#上面配置的条带卷在生产环境是很少使用的，因为它会将文件破坏，比如一个图片，它会将图片一份一份地分别存到条带卷中的brick上。
```

#### 配置分布式复制卷(拓展卷)

注意:

  - 块服务器的数量必须是复制的倍数
  - 将按块服务器的排列顺序指定相邻的块服务器成为彼此的复制； 例如，8台服务器：
   - 当复制副本为2时，按照服务器列表的顺序，服务器1和2作为一个复制,3和4作为一个复制,5和6作为一个复制,7和8作为一个复制
   - 当复制副本为4时，按照服务器列表的顺序，服务器1/2/3/4作为一个复制,5/6/7/8作为一个复制


```shell
#将原有的复制卷gv2进行扩容，使其成为分布式复制卷；
#要扩容前需停掉gv2
[root@node01 ~] gluster volume stop gv2
[root@node01 ~] gluster volume add-brick gv2 replica 2 node03:/data/brick1 node04:/data/brick1 force  #添加brick到gv2中
volume add-brick: success
[root@node01 ~] gluster volume start gv2
volume start: gv2: success
[root@node01 ~] gluster volume info gv2
 
Volume Name: gv2
Type: Distributed-Replicate    # 这里显示是分布式复制卷，是在 gv2 复制卷的基础上增加 2 块 brick 形成的
Volume ID: 9f33bd9a-7096-4749-8d91-1e6de3b50053
Status: Started
Snapshot Count: 0
Number of Bricks: 2 x 2 = 4
Transport-type: tcp
Bricks:
Brick1: node01:/data/brick2
Brick2: node02:/data/brick2
Brick3: node03:/data/brick1
Brick4: node04:/data/brick1
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off

注意：当你给分布式复制卷和分布式条带卷增加 bricks 时，你增加的 bricks 数目必须是复制或条带数目的倍数，
例如：你给一个分布式复制卷的 replica 为 2，你在增加 bricks 的时候数量必须为2、4、6、8等。
扩容后进行测试，发现文件都分布在扩容前的卷中。
```

#### 配置分布式条带卷(拓展卷)

```shell
#将原有的复制卷gv3进行扩容，使其成为分布式条带卷
#要扩容前需停掉gv3
[root@node01 ~] gluster volume stop gv3
[root@node01 ~] gluster volume add-brick gv3 stripe 2 node03:/data/brick2 node04:/data/brick2 force  #添加brick到gv3中
[root@node01 ~] gluster volume start gv3
volume start: gv3: success
[root@node01 ~] gluster volume info gv3
 
Volume Name: gv3
Type: Distributed-Stripe   # 这里显示是分布式条带卷，是在 gv3 条带卷的基础上增加 2 块 brick 形成的
Volume ID: 54c16832-6bdf-42e2-81a9-6b8d7b547c1a
Status: Started
Snapshot Count: 0
Number of Bricks: 2 x 2 = 4
Transport-type: tcp
Bricks:
Brick1: node01:/data/brick3
Brick2: node02:/data/brick3
Brick3: node03:/data/brick2
Brick4: node04:/data/brick2
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
```

#### 磁盘存储平衡(拓展卷后操作)

平衡布局是很有必要的，因为布局结构是静态的，当新的 bricks 加入现有卷，新创建的文件会分布到旧的 bricks 中，所以需要平衡布局结构，使新加入的 bricks 生效。布局平衡只是使新布局生效，并不会在新的布局中移动老的数据，如果你想在新布局生效后，重新平衡卷中的数据，还需要对卷中的数据进行平衡。

```shell
#在gv2的分布式复制卷的挂载目录中创建测试文件如下
[root@node01 ~] df -h
文件系统        容量  已用  可用 已用% 挂载点
127.0.0.1:/gv2   10G   65M   10G    1% /mnt
[root@node01 ~] cd /mnt/
[root@node01 mnt] touch {x..z}
#新创建的文件只在老的brick中有，在新加入的brick中是没有的
[root@node01 mnt]# ls /data/brick2
1  2  3  4  5  6  x  y  z
[root@node02 ~] ls /data/brick2
1  2  3  4  5  6  x  y  z
[root@node03 ~] ll -h /data/brick1
总用量 0
[root@node04 ~] ll -h /data/brick1
总用量 0
# 从上面可以看到，新创建的文件还是在之前的 bricks 中，并没有分布中新加的 bricks 中

# 下面进行磁盘存储平衡
[root@node01 ~] gluster volume rebalance gv2 start
[root@node01 ~] gluster volume rebalance gv2 status  #查看平衡存储状态
# 查看磁盘存储平衡后文件在 bricks 中的分布情况
[root@node01 ~] ls /data/brick2
1  5  y
[root@node02 ~] ls /data/brick2
1  5  y
[root@node03 ~] ls /data/brick1
2  3  4  6  x  z
[root@node04 ~] ls /data/brick1
2  3  4  6  x  z
# 从上面可以看出部分文件已经平衡到新加入的brick中了
# 每做一次扩容后都需要做一次磁盘平衡。 磁盘平衡是在万不得已的情况下再做的，一般再创建一个卷就可以了。
```

#### 移除brick(收缩卷)

你可能想在线缩小卷的大小，例如：当硬件损坏或网络故障的时候，你可能想在卷中移除相关的 bricks。

注意：当你移除 bricks 的时候，你在 gluster 的挂载点将不能继续访问数据，只有配置文件中的信息移除后你才能继续访问 bricks 中的数据。当移除分布式复制卷或者分布式条带卷的时候，移除的 bricks 数目必须是 replica 或者 stripe 的倍数。

但是移除brick在生产环境中基本上不做的，如果是硬盘坏掉的话，直接换个好的硬盘即可，然后再对新的硬盘设置卷标识就可以使用了，后面会演示硬件故障或系统故障的解决办法。

```shell
[root@node01 ~] gluster volume stop gv2
# 先将数据迁移到其它可用的Brick，迁移结束后才将该Brick移除：
[root@node01 ~] gluster volume remove-brick gv2 replica 2 node03:/data/brick1 node04:/data/brick1 start
# 在执行了start之后，可以使用status命令查看移除进度
[root@node01 ~] gluster volume remove-brick gv2 replica 2 node03:/data/brick1 node04:/data/brick1 status
# 不进行数据迁移，直接删除该Brick
[root@node01 ~] gluster volume remove-brick gv2 replica 2 node03:/data/brick1 node04:/data/brick1 commit
[root@node01 ~] gluster volume info gv2    
Volume Name: gv2
Type: Replicate
Volume ID: 9f33bd9a-7096-4749-8d91-1e6de3b50053
Status: Stopped
Snapshot Count: 0
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: node01:/data/brick2
Brick2: node02:/data/brick2
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
# 如果误操作删除了后，其实文件还在 /storage/brick1 里面的，加回来就可以了
[root@node01 ~] gluster volume add-brick gv2 replica 2 node03:/data/brick1 node04:/data/brick1 force  
volume add-brick: success
[root@node01 ~] gluster volume info gv2
 
Volume Name: gv2
Type: Distributed-Replicate
Volume ID: 9f33bd9a-7096-4749-8d91-1e6de3b50053
Status: Stopped
Snapshot Count: 0
Number of Bricks: 2 x 2 = 4
Transport-type: tcp
Bricks:
Brick1: node01:/data/brick2
Brick2: node02:/data/brick2
Brick3: node03:/data/brick1
Brick4: node04:/data/brick1
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```

#### 迁移卷(迁移卷)

```shell
[root@node01 ~] gluster volume stop gv2
# 先将数据迁移到其它可用的Brick：
[root@node01 ~] gluster volume replace-brick gv2 node03:/data/brick1 node06:/data/brick1 start
# 在执行了start之后，可以使用status命令查看迁移进度
[root@node01 ~] gluster volume replace-brick gv2 node03:/data/brick1 node06:/data/brick1 status
# 在数据迁移结束后，执行commit命令来进行Brick替换，如果不进行迁移直接commit会造成数据丢失
[root@node01 ~] gluster volume replace-brick gv2 node03:/data/brick1 node06:/data/brick1 commit
[root@node01 ~] gluster volume info gv2    
Volume Name: gv2
Type: Replicate
Volume ID: 9f33bd9a-7096-4749-8d91-1e6de3b50053
Status: Stopped
Snapshot Count: 0
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: node01:/data/brick2
Brick2: node02:/data/brick2
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```

#### 删除卷

一般会用在命名不规范的时候才会删除 

```shell
[root@node01 ~] gluster volume stop gv1
[root@node01 ~] gluster volume delete gv
```

#### 卷误删解决办法

```shell
[root@node01 ~] ls /var/lib/glusterd/vols/
gv2  gv3
[root@node01 ~] rm -rf /var/lib/glusterd/vols/gv3 #删除卷gv3的卷信息
[root@node01 ~]    ls /var/lib/glusterd/vols/     #再查看卷信息情况如下：gv3卷信息被删除了
gv2
[root@node01 ~] gluster volume sync node02      #因为其他节点服务器上的卷信息是完整的，比如从node02上同步所有卷信息如下：
Sync volume may make data inaccessible while the sync is in progress. Do you want to continue? (y/n) y
volume sync: success
[root@node01 ~] ls /var/lib/glusterd/vols/    #验证卷信息是否同步过来
gv2  gv3
```

#### 卷数据不一致解决办法

```shell
[root@node01 ~] ls /data/brick2    #复制卷的存储位置的数据
1  5  y
[root@node01 ~] rm -f /data/brick2/y 
[root@node01 ~] ls /data/brick2
1  5
[root@node02 ~] ls /data/brick2
1  5  y
[root@node01 ~] gluster start gv2   #因为之前关闭了，如果未关闭可以忽略此步。
[root@node01 ~] cat /mnt/y     #通过访问这个复制卷的挂载点的数据来同步数据
[root@node01 ~] ls /data/brick2/     #这时候再看复制卷的数据是否同步成功
1  5  y
```

### glusterfs 分布式存储优化

**优化参数**

```shell
Auth_allow　　#IP访问授权；缺省值（*.allow all）；合法值：Ip地址
Cluster.min-free-disk　　#剩余磁盘空间阀值；缺省值（10%）；合法值：百分比
Cluster.stripe-block-size　　#条带大小；缺省值（128KB）；合法值：字节
Network.frame-timeout　　#请求等待时间；缺省值（1800s）；合法值：1-1800
Network.ping-timeout　　#客户端等待时间；缺省值（42s）；合法值：0-42
Nfs.disabled　　#关闭NFS服务；缺省值（Off）；合法值：Off|on
Performance.io-thread-count　　#IO线程数；缺省值（16）；合法值：0-65
Performance.cache-refresh-timeout　　#缓存校验时间；缺省值（1s）；合法值：0-61
Performance.cache-size　　#读缓存大小；缺省值（32MB）；合法值：字节

Performance.quick-read: #优化读取小文件的性能
Performance.read-ahead: #用预读的方式提高读取的性能，有利于应用频繁持续性的访问文件，当应用完成当前数据块读取的时候，下一个数据块就已经准备好了。
Performance.write-behind:先写入缓存内，在写入硬盘，以提高写入的性能。
Performance.io-cache:缓存已经被读过的、
```

**优化方式**

```shell
命令格式：
gluster.volume set <卷><参数>

例如：
#打开预读方式访问存储
[root@node01 ~]# gluster volume set gv2 performance.read-ahead on

#调整读取缓存的大小
[root@mystorage gv2]# gluster volume set gv2 performance.cache-size 256M
```

### glusterfs 监控及维护

```shell
使用zabbix自带的模板即可，CPU、内存、磁盘空间、主机运行时间、系统load。日常情况要查看服务器监控值，遇到报警要及时处理。
#看下节点有没有在线
gluster volume status nfsp

#启动完全修复
gluster volume heal gv2 full

#查看需要修复的文件
gluster volume heal gv2 info

#查看修复成功的文件
gluster volume heal gv2 info healed

#查看修复失败的文件
gluster volume heal gv2 heal-failed

#查看主机的状态
gluster peer status

#查看脑裂的文件
gluster volume heal gv2 info split-brain

#激活quota功能
gluster volume quota gv2 enable

#关闭quota功能
gulster volume quota gv2 disable

#目录限制（卷中文件夹的大小）
gluster volume quota limit-usage /data/30MB --/gv2/data

#quota信息列表
gluster volume quota gv2 list

#限制目录的quota信息
gluster volume quota gv2 list /data

#设置信息的超时时间
gluster volume set gv2 features.quota-timeout 5

#删除某个目录的quota设置
gluster volume quota gv2 remove /data

备注：quota功能，主要是对挂载点下的某个目录进行空间限额。如：/mnt/gulster/data目录，而不是对组成卷组的空间进行限制。
```

### 故障处理

#### 一台主机故障

一台节点故障的情况包含以下情况：

- 物理故障
- 同时有多块硬盘故障，造成数据丢失
- 系统损坏不可修复

**解决方法：**

​	找一台完全一样的机器，至少要保证硬盘数量和大小一致，安装系统，配置和故障机同样的 IP，安装 gluster 软件，保证配置一样，在其他健康节点上执行命令 gluster peer status，查看故障服务器的 uuid

```shell
[root@mystorage2 ~] gluster peer status
Number of Peers: 3

Hostname: mystorage3
Uuid: 36e4c45c-466f-47b0-b829-dcd4a69ca2e7
State: Peer in Cluster (Connected)

Hostname: mystorage4
Uuid: c607f6c2-bdcb-4768-bc82-4bc2243b1b7a
State: Peer in Cluster (Connected)

Hostname: mystorage1
Uuid: 6e6a84af-ac7a-44eb-85c9-50f1f46acef1
State: Peer in Cluster (Disconnected)
复制代码
修改新加机器的 /var/lib/glusterd/glusterd.info 和 故障机器一样

[root@mystorage1 ~] cat /var/lib/glusterd/glusterd.info
UUID=6e6a84af-ac7a-44eb-85c9-50f1f46acef1
operating-version=30712
在信任存储池中任意节点执行
gluster volume heal gv2 full
就会自动开始同步，但在同步的时候会影响整个系统的性能。

可以查看状态
gluster volume heal gv2 info
```

