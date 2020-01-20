---
title: MAPREDUCE
date: "2020-01-10 15:20:00"
categories:
- 大数据
- hadoop
tags:
- 大数据
- HADOOP
toc: true
typora-root-url: ..\..\..
---

## MAPREDUCE 介绍

### MAPREDUCE简介

Mapreduce是一个分布式运算程序的编程框架，是用户开发“基于hadoop的数据分析应用”的核心框架；

Mapreduce核心功能是将用户编写的业务逻辑代码和自带默认组件整合成一个完整的分布式运算程序，并发运行在一个hadoop集群上；

### 为什么使用MAPREDUCE

1. 海量数据在单机上在单机上处理硬件资源限制，无法胜任

2. 而一旦将单机版程序扩展到集群来分布式运行，将极大增加程序的复杂度和开发难度
3. 引入mapreduce框架后，开发人员可以将绝大部分工作集中在业务逻辑的开发上，而将分布式计算中的复杂性交由框架来处理

可见在程序由单机版扩成分布式时，会引入大量的复杂工作。为了提高开发效率，可以将分布式程序中的公共功能封装成框架，让开发人员可以将精力集中于业务逻辑。

### MAPREDUCE框架设计思想

![1578643919988](/img/1578643919988.png)

## MAPREDUCE结构与运行流程

### MAPREDUCE设计结构

一个完成的MAPREDUCE程序在分布式运行时有三类实例进程:

1. MRAppMaster:  负责整个程序的过程调度及状态协调
2. mapTask:  负责map阶段的整个数据处理流程
3. ReduceTask:  负责 reduce 阶段的整个数据处理流程

### MAPREDUCE运行流程

**示意图**

![1578641943447](/img/1578641943447.png)

![1578651977793](/img/1578651977793.png)

**流程解析**

1. 一个mr程序启动的时候，最先启动的是MRAppMaster，MRAppMaster启动后根据本次job的描述信息，计算出需要的maptask实例数量，然后向集群申请机器启动相应数量的maptask进程

2. maptask进程启动之后，根据给定的数据切片范围进行数据处理，主体流程为：

   a. 利用客户指定的inputformat来获取RecordReader读取数据，形成输入KV对

   b. 将输入KV对传递给客户定义的map()方法，做逻辑运算，并将map()方法输出的KV对收集到缓存

   c. 将缓存中的KV对按照K分区排序后不断溢写到磁盘文件

3. MRAppMaster监控到所有maptask进程任务完成之后，会根据客户指定的参数启动相应数量的reducetask进程，并告知reducetask进程要处理的数据范围（数据分区）

4. Reducetask进程启动之后，根据MRAppMaster告知的待处理数据所在位置，从若干台maptask运行所在机器上获取到若干个maptask输出结果文件，并在本地进行重新归并排序，然后按照相同key的KV为一个组，调用客户定义的reduce()方法进行逻辑运算，并收集运算输出的结果KV，然后调用客户指定的outputformat将结果数据输出到外部存储

### MRclient提交MR程序给MR框架的流程

![1578644229986](/img/1578644229986.png)

## MAPREDUCE并行度机制

maptask的并行度决定map阶段的任务处理并发度，进而影响到整个job的处理速度

那么，mapTask并行实例是否越多越好呢？其并行度又是如何决定呢？

### mapTask 并行度的决定机制

#### mapTask并行度规则

一个job的map阶段并行度由客户端在提交job时决定

**而客户端对map阶段并行度的规划的基本逻辑为：**

将待处理数据执行逻辑切片（即按照一个特定切片大小，将待处理数据划分成逻辑上的多个split），然后每一个split分配一个mapTask并行实例处理

这段逻辑及形成的切片规划描述文件，由FileInputFormat实现类的getSplits()方法完成，其过程如下图：



![1578644404467](/img/1578644404467.png)

#### FileInputFormat 切片机制

**切片机制 (TextInputFormat示例)**

0. FileInputFormat 用来读取数据，其本身为一个抽象类，继承自 InputFormat 抽象类，针对不同的类型的数据有不同的子类来处理； FileInputFormat 常见的接口实现类包括：TextInputFormat、KeyValueTextInputFormat、NLinelnputFormat、CombineTextInputFormat 和自定义 ImputFormat 等。

1. 切片定义在MRclient的FileInputFormat类中的getSplit()方法中

2. FileInputFormat中默认的切片机制:

   a. 简单地按照文件的内容长度进行切片

   b. 切片大小，默认等于block大小

   c. 切片时不考虑数据集整体，而是逐个针对每一个文件单独切片

   比如待处理数据有两个文件:

   ```
   file1.txt    320M
   file2.txt    10M
   ```

   经过FileInputFormat的切片机制运算后，行程的切片信息如下:

   ```
   file1.txt.split1--  0~128
   file1.txt.split2--  128~256
   file1.txt.split3--  256~320
   file2.txt.split1--  0~10M
   ```

3. FileInputFormat 中切片的大小参数配置

   通过分析源码，在FileInputFormat中，计算切片大小的逻辑：

   ```java
   Math.max(minSize, Math.min(maxSize, blockSize));
   ```

   切片主要由这几个值来运算决定：

   **minsize**:   默认值：1, 配置参数： mapreduce.input.fileinputformat.split.minsize    

   **maxsize**：默认值：Long.MAXValue, 配置参数：mapreduce.input.fileinputformat.split.maxsize

   因此，默认情况下，切片大小=blocksize

   **maxsize（切片最大值）：**

   参数如果调得比blocksize小，则会让切片变小，而且就等于配置的这个参数的值

   **minsize （切片最小值）：**

   参数调的比blockSize大，则可以让切片变得比blocksize还大

   **配置并发数的影响因素:**

   1. 运算节点的硬件配置
   2. 运算任务的类型: CPU密集型还是IO密集型
   3. 运算任务的数量

**小文件切片优化**

  如果是小文件，就会产生大量的小切片，造成大量的maptask运行

  解决办法:

  	 1. 从源头上解决问题，文件合并后再上传处理
    	 2. 可以用另外一种InputFormat:  CombineInputFormat（可以将多个文件划分到一个切片中）, 可以设置每个技巧篇的最小容量和最大容量;

#### MAPTASK 并行度建议设置

- 如果硬件配置为2*12core + 64G，恰当的map并行度是大约每个节点20-100个map，**最好每个map的执行时间至少一分钟。**

- 如果job的每个map或者 reduce task的运行时间都只有30-40秒钟，那么就减少该job的map或者reduce数，每一个task(map|reduce)的setup和加入到调度器中进行调度，这个中间的过程可能都要花费几秒钟，所以如果每个task都非常快就跑完了，就会在task的开始和结束的时候浪费太多的时间。

- 配置task的JVM重用可以改善该问题：（mapred.job.reuse.jvm.num.tasks，默认是1，表示一个JVM上最多可以顺序执行的task数目（属于同一个Job）是1。也就是说一个task启一个JVM）

- 如果input的文件非常的大，比如1TB，可以考虑将hdfs上的每个block size设大，比如设成256MB或者512MB

### ReduceTask并行度的决定

#### ReduceTask 设置方式

reducetask的并行度同样影响整个job的执行并发度和执行效率，但与maptask的并发数由切片数决定不同，Reducetask数量的决定是可以直接手动设置：

```java
//默认值是1，手动设置为4
job.setNumReduceTasks(4);
```

如果数据分布不均匀，就有可能在reduce阶段产生数据倾斜

> 注意： reducetask数量并不是任意设置，还要考虑业务逻辑需求，有些情况下，需要计算全局汇总结果，就只能有1个reducetask，尽量不要运行太多的reduce task。
>
> 当设置reduceTask数量为0的时候，mapreduce 不会执行shuffle以及后面的操作，会直接将map的结果输出到outputPath目录，此时有多少个mapTask就有多少个文件。
>
> 如果不设置reduceTask数量，mapreduce将会默认启动一个reduceTask， 用来将所有map的结果进行整理，并输出到统一文件到输出目录。

## MAPREDUCE 编程规范

### 编写规范

1. 用户编写的程序分成三个部分：Mapper，Reducer，Driver(提交运行mr程序的客户端)
2. Mapper的输入数据是KV对的形式（KV的类型可自定义）
3. Mapper的输出数据是KV对的形式（KV的类型可自定义）
4. Mapper中的业务逻辑写在map()方法中
5. map()方法（maptask进程）对每一个<K,V>调用一次
6. Reducer的输入数据类型对应Mapper的输出数据类型，也是KV
7. Reducer的业务逻辑写在reduce()方法中
8. Reducetask进程对每一组相同k的<k,v>组调用一次reduce()方法
9. 用户自定义的Mapper和Reducer都要继承各自的父类
10. 整个程序需要一个Drvier来进行提交，提交的是一个描述了各种必要信息的job对象

### MAPREDUCE示例

> 需求: 在一堆给定的文本文件中统计输出每一个单词出现的总次数

**定义mapper类**

```java
//首先要定义四个泛型的类型
//keyin:  LongWritable    valuein: Text
//keyout: Text            valueout:IntWritable

public class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable>{
	//map方法的生命周期：  框架每传一行数据就被调用一次
	//key :  这一行的起始点在文件中的偏移量
	//value: 这一行的内容
	@Override
	protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
		//拿到一行数据转换为string
		String line = value.toString();
		//将这一行切分出各个单词
		String[] words = line.split(" ");
		//遍历数组，输出<单词，1>
		for(String word:words){
			context.write(new Text(word), new IntWritable(1));
		}
	}
}
```

**定义reduce类**

```java
//生命周期：框架每传递进来一个kv 组，reduce方法被调用一次
	@Override
	protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
		//定义一个计数器
		int count = 0;
		//遍历这一组kv的所有v，累加到count中
		for(IntWritable value:values){
			count += value.get();
		}
		context.write(key, new IntWritable(count));
	}
}
```

**定义主类，描述并提交job**

```java
public class WordCountRunner {
	//把业务逻辑相关的信息（哪个是mapper，哪个是reducer，要处理的数据在哪里，输出的结果放哪里……）描述成一个job对象
	//把这个描述好的job提交给集群去运行
	public static void main(String[] args) throws Exception {
		Configuration conf = new Configuration();
		Job wcjob = Job.getInstance(conf);
		//指定我这个job所在的jar包
//		wcjob.setJar("/home/hadoop/wordcount.jar");
		wcjob.setJarByClass(WordCountRunner.class);
		
		wcjob.setMapperClass(WordCountMapper.class);
		wcjob.setReducerClass(WordCountReducer.class);
		//设置我们的业务逻辑Mapper类的输出key和value的数据类型
		wcjob.setMapOutputKeyClass(Text.class);
		wcjob.setMapOutputValueClass(IntWritable.class);
		//设置我们的业务逻辑Reducer类的输出key和value的数据类型
		wcjob.setOutputKeyClass(Text.class);
		wcjob.setOutputValueClass(IntWritable.class);
		
		//指定要处理的数据所在的位置
		FileInputFormat.setInputPaths(wcjob, "hdfs://hdp-server01:9000/wordcount/data/big.txt");
		//指定处理完成之后的结果所保存的位置
		FileOutputFormat.setOutputPath(wcjob, new Path("hdfs://hdp-server01:9000/wordcount/output/"));
		//向yarn集群提交这个job
		boolean res = wcjob.waitForCompletion(true);
		System.exit(res?0:1);
	}
```

## MAPREDUCE 之YARN

### 概述

Yarn是一个资源调度平台，负责为运算程序提供服务器运算资源，相当于一个分布式的操作系统平台，而mapreduce等运算程序则相当于运行于操作系统之上的应用程序

### YARN的重要概念

1. yarn并不清楚用户提交的程序的运行机制
2. yarn只提供运算资源的调度（用户程序向yarn申请资源，yarn就负责分配资源）
3. yarn中的主管角色叫ResourceManager
4. yarn中具体提供运算资源的角色叫NodeManager
5. 这样一来，yarn其实就与运行的用户程序完全解耦，就意味着yarn上可以运行各种类型的分布式运算程序（mapreduce只是其中的一种），比如mapreduce、storm程序，spark程序，tez ……
6. 所以，spark、storm等运算框架都可以整合在yarn上运行，只要他们各自的框架中有符合yarn规范的资源请求机制即可
7. Yarn就成为一个通用的资源调度平台，从此，企业中以前存在的各种运算集群都可以整合在一个物理集群上，提高资源利用率，方便数据共享

**MR YARN集群运行机制**

![1578897882446](/img/1578897882446.png)

## MAPREDUCE程序运行机制

### 本地运行模式

1. mapreduce程序是被提交给LocalJobRunner在本地以单进程的形式运行
2. 而处理的数据及输出结果可以在本地文件系统，也可以在hdfs上
3. 怎样实现本地运行？写一个程序，不要带集群的配置文件（本质是你的mr程序的conf中是否有mapreduce.framework.name=local以及yarn.resourcemanager.hostname参数）
4. 本地模式非常便于进行业务逻辑的debug，只要在eclipse中打断点即可

> 如果在windows下想运行本地模式来测试程序逻辑，需要在windows中配置环境变量：
>
> ％HADOOP_HOME％  =  d:/hadoop-2.6.1
>
> %PATH% =  ％HADOOP_HOME％\bin
>
> 并且要将d:/hadoop-2.6.1的lib和bin目录替换成windows平台编译的版本

### 集群运行模式

**什么是集群模式**

1. 将mapreduce程序提交给yarn集群resourcemanager，分发到很多的节点上并发执行
2. 处理的数据和输出结果应该位于hdfs文件系统

**提交集群的实现步骤**

1. 将程序打成JAR包，然后在集群的任意一个节点上用hadoop命令启动

   ```shell
   hadoop jar wordcount.jar cn.itcast.bigdata.mrsimple.WordCountDriver inputpath outputpath
   ```

2. 直接在linux的eclipse中运行main方法

   **（项目中要带参数：mapreduce.framework.name=yarn以及yarn的两个基本配置）**

3. 如果要在windows的eclipse中提交job给集群，则要修改YarnRunner类

   >  在windows平台上访问hadoop时, 应该带上参数  -DHADOOP_USER_NAME=hadoop

## MAPREDUCE运行原理

### MAPREDUCE运行原理图

![1578900985611](/img/1578900985611.png)

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

## MAPREDUCE 各种类

### Map类和Reduce类

**作用**

MR主要的工作机制就是通过map对数据行进行递归的操作， 通过reduce对处理的结果进行汇总操作

具体实现步骤：

1. 自定义Map 和 Reduce类 并实现map和reduce方法；

2. 在job实例中，引用Map和Reduce的类: 

   **job.setJarByClass(FlowCount.class);**
   **job.setMapperClass(FlowCountMapper.class);**
   **job.setReducerClass(FlowCountReducer.class);**

**Map类示例**

```java
public class WordcountMapper extends Mapper<LongWritable, Text, Text, IntWritable>{
    
    HashMap<String,String> cache = new HashMap<String, String>();
	
    // 是在map任务初始化的时候调用一次，然后再对文本的每一行进行map操作
    @Override
    protected void setup(Context context) {
		cache.put("some_key", "some_value")
    }
    
	/**
	 * map阶段的业务逻辑就写在自定义的map()方法中
	 * maptask会对每一行输入数据调用一次我们自定义的map()方法
	 */
	@Override
	protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
		//将maptask传给我们的文本内容先转换成String
		String line = value.toString();
		//根据空格将这一行切分成单词
		String[] words = line.split(" ");
		
		//将单词输出为<单词，1>
		for(String word:words){
			//将单词作为key，将次数1作为value，以便于后续的数据分发，可以根据单词分发，以便于相同单词会到相同的reduce task
			context.write(new Text(word), new IntWritable(1));
		}
	}
}
```

**Reduce类示例**

```java
public class WordcountReducer extends Reducer<Text, IntWritable, Text, IntWritable>{
	
    HashMap<String,String> cache = new HashMap<String, String>();
	
    // 是在map任务初始化的时候调用一次，然后再对文本的每一行进行map操作
    @Override
    protected void setup(Context context) {
		cache.put("some_key", "some_value")
    }
	
    /**
	 * <angelababy,1><angelababy,1><angelababy,1><angelababy,1><angelababy,1>
	 * <hello,1><hello,1><hello,1><hello,1><hello,1><hello,1>
	 * <banana,1><banana,1><banana,1><banana,1><banana,1><banana,1>
	 * 入参key，是一组相同单词kv对的key
	 */
	@Override
	protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
		int count=0;
		/*Iterator<IntWritable> iterator = values.iterator();
		while(iterator.hasNext()){
			count += iterator.next().get();
		}*/
		for(IntWritable value:values){
		
			count += value.get();
		}
		context.write(key, new IntWritable(count));
	}
}
```

### Bean类

**作用**

Java的序列化是一个重量级序列化框架（Serializable），一个对象被序列化后，会附带很多额外的信息（各种校验信息，header，继承体系。。。。），不便于在网络中高效传输；

hadoop自己开发了一套序列化机制（Writable），精简，高效，用来进行高效方便的传输

MAPREDUCE就可以借助hadoop内置的或者自定义的Bean对象(继承Writable)， 来作为mapTask和reduceTask的输入输出对象

如果需要将自定义的bean放在key中传输(当需要对key进行排序的时候)，则还需要实现comparable接口，因为mapreduce框中的shuffle过程一定会对key进行排序；

具体实现步骤：

1. 自定义Writable接口，并重写readFields、write、toString等方法

2. 在map 和 reduce方法中，输出和接受自定义的Bean； 

3. 在job实例中，指定keyclass和valueclass:

   **job.setMapOutputKeyClass(Text.class);**
   **job.setMapOutputValueClass(FlowBean.class);**
   **job.setOutputKeyClass(Text.class);**
   **job.setOutputValueClass(FlowBean.class);**

**Writable接口方法**

```java
/**
* 反序列化的方法，反序列化时，从流中读取到的各个字段的顺序应该与序列化时写出去的顺序保持一致
*/
@Override
public void readFields(DataInput in) throws IOException {
    upflow = in.readLong();
    dflow = in.readLong();
    sumflow = in.readLong();
}

/**
* 传输过程中的序列化的方法
*/
@Override
public void write(DataOutput out) throws IOException {
    out.writeLong(upflow);
    out.writeLong(dflow);
    //可以考虑不序列化总流量，因为总流量是可以通过上行流量和下行流量计算出来的
    out.writeLong(sumflow);
}

/**
* shuffle过程中对key进行排序的规则
*/
@Override
public int compareTo(FlowBean o) {
    //实现按照sumflow的大小倒序排序
    return sumflow>o.getSumflow()?-1:1;
}

/**
* 最终reduce将bean输出到文本的时候，显示的内容
*/
@Override
public String toString() {
    return upFlow + "\t" + dFlow + "\t" + sumFlow;
}
```

### Partitioner类

**作用**

Mapreduce中会将map输出的kv对，按照相同key分组，将数据分为不同的分区, 然后分发给不同的reducetask

默认的分发规则为：根据key的 **hashcode%reducetask** 数来分发

所以, 如果要按照我们自己的需求进行分组，则需要改写数据分发（分组）组件Partitioner

具体实现步骤：

1. 自定义一个CustomPartitioner继承抽象类：Partitioner

2. 然后在job对象中，设置自定义partitioner： **job.setPartitionerClass(CustomPartitioner.class)**

**示例**

```java
/**
 * 定义自己的从map到reduce之间的数据（分组）分发规则 按照手机号所属的省份来分发（分组）ProvincePartitioner
 * 默认的分组组件是HashPartitioner
 * 
 * @author
 * 
 */
public class ProvincePartitioner extends Partitioner<Text, FlowBean> {

	static HashMap<String, Integer> provinceMap = new HashMap<String, Integer>();

	static {
		provinceMap.put("135", 0);
		provinceMap.put("136", 1);
		provinceMap.put("137", 2);
		provinceMap.put("138", 3);
		provinceMap.put("139", 4);
	}

	@Override
    public int getPartition(Text key, FlowBean value, int numPartitions) {
		Integer code = provinceMap.get(key.toString().substring(0, 3));
		return code == null ? 5 : code;
	}

}
```

### Combine类

**作用**

combiner是MR程序中Mapper和Reducer之外的一种组件, combiner组件的父类就是Reducer

combiner和reducer的区别在于运行的位置：Combiner是在每一个maptask所在的节点运行, Reducer是接收全局所有Mapper的

输出结果；

combiner的意义就是对每一个maptask的输出进行局部汇总，以减小网络传输量

具体实现步骤：

1. 自定义一个combiner继承Reducer，重写reduce方法

2. 在job中设置：  **job.setCombinerClass(CustomCombiner.class)**

> combiner能够应用的前提是不能影响最终的业务逻辑。 而且，combiner的输出kv应该跟reducer的输入kv类型要对应起来

**示例**

```java
/**
  * 对象同key的value进行累加合并
 */
public class WordcountCombiner extends Reducer<Text, IntWritable, Text, IntWritable>{
	@Override
	protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
		int count=0;
		for(IntWritable v: values){
			
			count += v.get();
		}
		context.write(key, new IntWritable(count));
	}
}
```

### 主类(job对象)

**作用**

是与YARN集群通讯的客户端；

用来指定MR的Map，Reduce类等信息(如Map、Reduce、Partitioner、Combiner)，并设置启动参数等；

并启动客户端来与YARN通讯，并提交jar包等文件，请求启动task等；

**示例**

```java
public class FlowCount {
	
	public static void main(String[] args) throws Exception {
		Configuration conf = new Configuration();
		/*conf.set("mapreduce.framework.name", "yarn");
		conf.set("yarn.resoucemanager.hostname", "mini1");*/
		Job job = Job.getInstance(conf);
		
		/*job.setJar("/home/hadoop/wc.jar");*/
		//指定本程序的jar包所在的本地路径
		job.setJarByClass(FlowCount.class);
		
		//指定本业务job要使用的mapper/Reducer业务类
		job.setMapperClass(FlowCountMapper.class);
		job.setReducerClass(FlowCountReducer.class);
		
		//指定我们自定义的数据分区器
		job.setPartitionerClass(ProvincePartitioner.class);
		//同时指定相应“分区”数量的reducetask
		job.setNumReduceTasks(5);
		
		//指定mapper输出数据的kv类型
		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(FlowBean.class);
		
		//指定最终输出的数据的kv类型
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(FlowBean.class);
		
		//指定job的输入原始文件所在目录
		FileInputFormat.setInputPaths(job, new Path(args[0]));
		//指定job的输出结果所在目录
		FileOutputFormat.setOutputPath(job, new Path(args[1]));
        
        // 指定需要缓存一个文件到所有的maptask运行节点工作目录
		/* job.addArchiveToClassPath(archive); */// 缓存jar包到task运行节点的classpath中
		/* job.addFileToClassPath(file); */// 缓存普通文件到task运行节点的classpath中
		/* job.addCacheArchive(uri); */// 缓存压缩包文件到task运行节点的工作目录
		/* job.addCacheFile(uri) */// 缓存普通文件到task运行节点的工作目录
		// 将产品表文件缓存到task工作节点的工作目录中去, 在map和reduce运行阶段可以直接从工作目录里读取该文件
        // 在Mapper或者Reducer中可以使用context.getLocalCacheFiles() 获取缓存文件,或者直接从工作目录获取
		job.addCacheFile(new URI("file:/D:/srcdata/mapjoincache/pdts.txt"));
        
		//将job中配置的相关参数，以及job所用的java类所在的jar包，提交给yarn去运行
		/*job.submit();*/
		boolean res = job.waitForCompletion(true);
		System.exit(res?0:1);
	}
}
```

### GroupingComparator 类

**作用**

map 输出的K, V 如果K是自定义Bean对象，那么MAPREDUCE不能识别相同的K的K,V 为一组

这时，如果想要让相同的自定义K作为一组输入到reduce方法，需要设置一个GroupingComparator类，来指定对自定义对象分组的依据

原理是K,V对在传递给REDUCE方法的时候，reduce会先判断下一个K是否与当前K相同，如果相同，继续向下判断，直到获得

**使用示例**

```java
/* 
* 利用reduce端的GroupingComparator来实现将一组bean看成相同的key
*/
public class ItemidGroupingComparator extends WritableComparator {
	//传入作为key的bean的class类型，以及制定需要让框架做反射获取实例对象
	protected ItemidGroupingComparator() {
		super(OrderBean.class, true);
	}
	// 通过compare方法来判断key是否属于一组
	@Override
	public int compare(WritableComparable a, WritableComparable b) {
		OrderBean abean = (OrderBean) a;
		OrderBean bbean = (OrderBean) b;
		
		//比较两个bean时，指定只比较bean中的orderid
		return abean.getItemid().compareTo(bbean.getItemid());
	}
}
// 在主类中设置GroupingComparatorClass
//job.setGroupingComparatorClass(ItemidGroupingComparator.class);

```

### OutputFormat类

#### 作用

MapReduce 只一共了一些标准的OutputFormat，这样当我们想要自定义Reduce结果的保存方式，和保存位置时，标准的OutputFormat就不能满足我们的需求；

比如我们想要在运营商的流量日志中，增加一列标记为访问目标网站的类型（依赖已经存在的网站分类数据库表）, 并且把分类成功统计到一个文件，分类失败的统计到另外一个文件中；

这时候原始的OutputFormat就不能满足需求，因为它只能根据分区结果去生成文件， 因此我们可以用自定义OutputFormat解决；

#### 示例

**需求**

1、 从原始日志文件中读取数据

2、 根据日志中的一个URL字段到外部知识库中获取信息增强到原始日志

3、 如果成功增强，则输出到增强结果目录；如果增强失败，则抽取原始数据中URL字段输出到待爬清单目录

**分析**

程序的关键点是要在一个mapreduce程序中根据数据的不同输出两类结果到不同目录，这类灵活的输出需求可以通过自定义outputformat来实现

**实现**

1. 在mapreduce中访问外部资源
2. 自定义outputformat，改写其中的recordwriter，改写具体输出数据的方法write()

OutputFormat类

```java
public class LogEnhancerOutputFormat extends FileOutputFormat<Text, NullWritable>{

	
	@Override
	public RecordWriter<Text, NullWritable> getRecordWriter(TaskAttemptContext context) throws IOException, InterruptedException {


		FileSystem fs = FileSystem.get(context.getConfiguration());
		Path enhancePath = new Path("hdfs://hdp-node01:9000/flow/enhancelog/enhanced.log");
		Path toCrawlPath = new Path("hdfs://hdp-node01:9000/flow/tocrawl/tocrawl.log");
		
		FSDataOutputStream enhanceOut = fs.create(enhancePath);
		FSDataOutputStream toCrawlOut = fs.create(toCrawlPath);
		
		
		return new MyRecordWriter(enhanceOut,toCrawlOut);
	}
	
	
	static class MyRecordWriter extends RecordWriter<Text, NullWritable>{
		
		FSDataOutputStream enhanceOut = null;
		FSDataOutputStream toCrawlOut = null;
		
		public MyRecordWriter(FSDataOutputStream enhanceOut, FSDataOutputStream toCrawlOut) {
            super();
			this.enhanceOut = enhanceOut;
			this.toCrawlOut = toCrawlOut;
		}

		@Override
		public void write(Text key, NullWritable value) throws IOException, InterruptedException {
			 
			//有了数据，你来负责写到目的地  —— hdfs
			//判断，进来内容如果是带tocrawl的，就往待爬清单输出流中写 toCrawlOut
			if(key.toString().contains("tocrawl")){
				toCrawlOut.write(key.toString().getBytes());
			}else{
				enhanceOut.write(key.toString().getBytes());
			}
				
		}

		@Override
		public void close(TaskAttemptContext context) throws IOException, InterruptedException {
			 
			if(toCrawlOut!=null){
				toCrawlOut.close();
			}
			if(enhanceOut!=null){
				enhanceOut.close();
			}
		}
	}
}
// 要将自定义的输出格式组件设置到job中
//job.setOutputFormatClass(LogEnhancerOutputFormat.class);
```

### OutputFormat类

#### 作用

更改文件的读取方式，或者对读取的内容提前做一些处理等

#### 示例

**需求**

无论hdfs还是mapreduce，对于小文件都有损效率，实践中，又难免面临处理大量小文件的场景，此时，就需要有相应解决方案

**分析**

1、 在数据采集的时候，就将小文件或小批数据合成大文件再上传HDFS

2、 在业务处理之前，在HDFS上使用mapreduce程序对小文件进行合并

3、 在mapreduce处理时，可采用combineInputFormat提高效率

**实现**

实现的是上述第二种方式

程序的核心机制：

自定义一个InputFormat

改写RecordReader，实现一次读取一个完整文件封装为KV

在输出时使用SequenceFileOutPutFormat输出合并文件

IutputFormat类

```java
public class WholeFileInputFormat extends
	FileInputFormat<NullWritable, BytesWritable> {
	//设置每个小文件不可分片,保证一个小文件生成一个key-value键值对
	@Override
	protected boolean isSplitable(JobContext context, Path file) {
		return false;
	}

	@Override
	public RecordReader<NullWritable, BytesWritable> createRecordReader(
			InputSplit split, TaskAttemptContext context) throws IOException,
			InterruptedException {
		WholeFileRecordReader reader = new WholeFileRecordReader();
		reader.initialize(split, context);
		return reader;
    }
}
```

RecordReader类

```java
class WholeFileRecordReader extends RecordReader<NullWritable, BytesWritable> {
	private FileSplit fileSplit;
	private Configuration conf;
	private BytesWritable value = new BytesWritable();
	private boolean processed = false;

	@Override
	public void initialize(InputSplit split, TaskAttemptContext context)
			throws IOException, InterruptedException {
		this.fileSplit = (FileSplit) split;
		this.conf = context.getConfiguration();
	}

	@Override
	public boolean nextKeyValue() throws IOException, InterruptedException {
		if (!processed) {
			byte[] contents = new byte[(int) fileSplit.getLength()];
			Path file = fileSplit.getPath();
			FileSystem fs = file.getFileSystem(conf);
			FSDataInputStream in = null;
			try {
				in = fs.open(file);
				IOUtils.readFully(in, contents, 0, contents.length);
				value.set(contents, 0, contents.length);
			} finally {
				IOUtils.closeStream(in);
			}
			processed = true;
			return true;
		}
		return false;
	}

	@Override
	public NullWritable getCurrentKey() throws IOException,
			InterruptedException {
		return NullWritable.get();
	}
	@Override
	public BytesWritable getCurrentValue() throws IOException,
			InterruptedException {
		return value;
	}

	@Override
	public float getProgress() throws IOException {
		return processed ? 1.0f : 0.0f;
	}

	@Override
	public void close() throws IOException {
		// do nothing
	}
}
```

mapreduce处理流程

```java
public class SmallFilesToSequenceFileConverter extends Configured implements Tool {
	static class SequenceFileMapper extends
			Mapper<NullWritable, BytesWritable, Text, BytesWritable> {
		private Text filenameKey;

		@Override
		protected void setup(Context context) throws IOException,
				InterruptedException {
			InputSplit split = context.getInputSplit();
			Path path = ((FileSplit) split).getPath();
			filenameKey = new Text(path.toString());
		}

		@Override
		protected void map(NullWritable key, BytesWritable value,
				Context context) throws IOException, InterruptedException {
			context.write(filenameKey, value);
		}
	}

	@Override
	public int run(String[] args) throws Exception {
		Configuration conf = new Configuration();
		/*System.setProperty("HADOOP_USER_NAME", "hadoop");*/
		String[] otherArgs = new GenericOptionsParser(conf, args)
				.getRemainingArgs();
		if (otherArgs.length != 2) {
			System.err.println("Usage: combinefiles <in> <out>");
			System.exit(2);
		}
		
		Job job = Job.getInstance(conf,"combine small files to sequencefile");
		job.setJarByClass(SmallFilesToSequenceFileConverter.class);
		
		job.setInputFormatClass(WholeFileInputFormat.class);
		job.setOutputFormatClass(SequenceFileOutputFormat.class);
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(BytesWritable.class);
		job.setMapperClass(SequenceFileMapper.class);
		
		FileInputFormat.setInputPaths(job, new Path(args[0]));
		FileOutputFormat.setOutputPath(job, new Path(args[1]));
		
		return job.waitForCompletion(true) ? 0 : 1;
	}

	public static void main(String[] args) throws Exception {
		args=new String[]{"c:/wordcount/smallinput","c:/wordcount/smallout"};
		int exitCode = ToolRunner.run(new SmallFilesToSequenceFileConverter(),
				args);
		System.exit(exitCode);
		
	}
}
```

### 计数器

**作用**

在实际生产代码中，常常需要将数据处理过程中遇到的不合规数据行进行全局计数，类似这种需求可以借助mapreduce框架中提供的全局计数器来实现

**示例**

```java
public class MultiOutputs {
	//通过枚举形式定义自定义计数器
	enum MyCounter{MALFORORMED,NORMAL}

	static class CommaMapper extends Mapper<LongWritable, Text, Text, LongWritable> {

		@Override
		protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
			String[] words = value.toString().split(",");

			for (String word : words) {
				context.write(new Text(word), new LongWritable(1));
			}
			//对枚举定义的自定义计数器加1
			context.getCounter(MyCounter.MALFORORMED).increment(1);
			//通过动态设置自定义计数器加1
			context.getCounter("counterGroupa", "countera").increment(1);
		}
	}
```

## MAPREDUCE 数据压缩

### 数据压缩配置

在配置参数或在代码中都可以设置reduce的输出压缩

**Reduce输出数据压缩**

1. 在配置参数中设置

   ```shell
   mapreduce.output.fileoutputformat.compress=false
   mapreduce.output.fileoutputformat.compress.codec=org.apache.hadoop.io.compress.DefaultCodec
   mapreduce.output.fileoutputformat.compress.type=RECORD
   ```

2. 在代码中配置

   ```java
   Job job = Job.getInstance(conf);
   FileOutputFormat.setCompressOutput(job, true);
   FileOutputFormat.setOutputCompressorClass(job, (Class<? extends CompressionCodec>) Class.forName(""));
   ```

**Mapper数据压缩**

1. 在配置参数中设置

   ```shell
   mapreduce.map.output.compress=false
   mapreduce.map.output.compress.codec=org.apache.hadoop.io.compress.DefaultCodec
   ```

2. 在代码中设置

   ```java
   conf.setBoolean(Job.MAP_OUTPUT_COMPRESS, true);
   conf.setClass(Job.MAP_OUTPUT_COMPRESS_CODEC, GzipCodec.class, CompressionCodec.class);
   ```

### 压缩文件的读取

Hadoop自带的InputFormat类内置支持压缩文件的读取，比如TextInputformat类，在其initialize方法中：

```java
 public void initialize(InputSplit genericSplit,
                         TaskAttemptContext context) throws IOException {
    FileSplit split = (FileSplit) genericSplit;
    Configuration job = context.getConfiguration();
    this.maxLineLength = job.getInt(MAX_LINE_LENGTH, Integer.MAX_VALUE);
    start = split.getStart();
    end = start + split.getLength();
    final Path file = split.getPath();

    // open the file and seek to the start of the split
    final FileSystem fs = file.getFileSystem(job);
    fileIn = fs.open(file);
    //根据文件后缀名创建相应压缩编码的codec
    CompressionCodec codec = new CompressionCodecFactory(job).getCodec(file);
    if (null!=codec) {
      isCompressedInput = true;	
      decompressor = CodecPool.getDecompressor(codec);
	  //判断是否属于可切片压缩编码类型
      if (codec instanceof SplittableCompressionCodec) {
        final SplitCompressionInputStream cIn =
          ((SplittableCompressionCodec)codec).createInputStream(
            fileIn, decompressor, start, end,
            SplittableCompressionCodec.READ_MODE.BYBLOCK);
		 //如果是可切片压缩编码，则创建一个CompressedSplitLineReader读取压缩数据
        in = new CompressedSplitLineReader(cIn, job,
            this.recordDelimiterBytes);
        start = cIn.getAdjustedStart();
        end = cIn.getAdjustedEnd();
        filePosition = cIn;
      } else {
		//如果是不可切片压缩编码，则创建一个SplitLineReader读取压缩数据，并将文件输入流转换成解压数据流传递给普通SplitLineReader读取
        in = new SplitLineReader(codec.createInputStream(fileIn,
            decompressor), job, this.recordDelimiterBytes);
        filePosition = fileIn;
      }
    } else {
      fileIn.seek(start);
	   //如果不是压缩文件，则创建普通SplitLineReader读取数据
      in = new SplitLineReader(fileIn, job, this.recordDelimiterBytes);
      filePosition = fileIn;
    }
```

## MAPREDUCE 参数优化

### 资源相关参数

> **以下参数是在用户自己的mr应用程序中配置就可以生效**

1. mapreduce.map.memory.mb: 一个Map Task可使用的资源上限（单位:MB），默认为1024。如果Map Task实际使用的资源量超过该值，则会被强制杀死。
2. mapreduce.reduce.memory.mb: 一个Reduce Task可使用的资源上限（单位:MB），默认为1024。如果Reduce Task实际使用的资源量超过该值，则会被强制杀死。

3. mapreduce.map.java.opts: Map Task的JVM参数，你可以在此配置默认的java heap size等参数, e.g：`-Xmx1024m -verbose:gc -Xloggc:/tmp/@taskid@.gc` （@taskid@会被Hadoop框架自动换为相应的taskid）, 默认值: ""

4. mapreduce.reduce.java.opts: Reduce Task的JVM参数，你可以在此配置默认的java heap size等参数, e.g: `-Xmx1024m -verbose:gc -Xloggc:/tmp/@taskid@.gc`, 默认值: ""

5. mapreduce.map.cpu.vcores: 每个Map task可使用的最多cpu core数目, 默认值: 1

6. mapreduce.reduce.cpu.vcores: 每个Reduce task可使用的最多cpu core数目, 默认值: 1

> 以下参数应该在yarn启动之前就配置在服务器的配置文件中才能生效

1. yarn.scheduler.minimum-allocation-mb	  1024   给应用程序container分配的最小内存

2. yarn.scheduler.maximum-allocation-mb	  8192	给应用程序container分配的最大内存

3. yarn.scheduler.minimum-allocation-vcores	1	 最小cpu

4. yarn.scheduler.maximum-allocation-vcores	32    最大cpu

5. yarn.nodemanager.resource.memory-mb   8192   一台nodemanager 可用的总内存 

> shuffle性能优化的关键参数，应在yarn启动之前就配置好

1. mapreduce.task.io.sort.mb   100         //shuffle的环形缓冲区大小，默认100m

2. mapreduce.map.sort.spill.percent   0.8    //环形缓冲区溢出的阈值，默认80%

### 容错相关参数

1. mapreduce.map.maxattempts: 每个Map Task最大重试次数，一旦重试参数超过该值，则认为Map Task运行失败，默认值：4。

2. mapreduce.reduce.maxattempts: 每个Reduce Task最大重试次数，一旦重试参数超过该值，则认为Map Task运行失败，默认值：4。
3. mapreduce.map.failures.maxpercent: 当失败的Map Task失败比例超过该值为，整个作业则失败，默认值为0. 如果你的应用程序允许丢弃部分输入数据，则该该值设为一个大于0的值，比如5，表示如果有低于5%的Map Task失败（如果一个Map Task重试次数超过mapreduce.map.maxattempts，则认为这个Map Task失败，其对应的输入数据将不会产生任何结果），整个作业扔认为成功。
4. mapreduce.reduce.failures.maxpercent: 当失败的Reduce Task失败比例超过该值为，整个作业则失败，默认值为0.
5. mapreduce.task.timeout: Task超时时间，经常需要设置的一个参数，该参数表达的意思为：如果一个task在一定时间内没有任何进入，即不会读取新的数据，也没有输出数据，则认为该task处于block状态，可能是卡住了，也许永远会卡主，为了防止因为用户程序永远block住不退出，则强制设置了一个该超时时间（单位毫秒），默认是300000。如果你的程序对每条输入数据的处理时间过长（比如会访问数据库，通过网络拉取数据等），建议将该参数调大，该参数过小常出现的错误提示是“AttemptID:attempt_14267829456721_123456_m_000224_0 Timed out after 300 secsContainer killed by the ApplicationMaster.”。

### 本地运行mapreduce 作业

设置以下几个参数:

1. mapreduce.framework.name=local

2. mapreduce.jobtracker.address=local

3. fs.defaultFS=local

### 效率和稳定性相关参数

1. mapreduce.map.speculative: 是否为Map Task打开推测执行机制，默认为false

2. mapreduce.reduce.speculative: 是否为Reduce Task打开推测执行机制，默认为false

3. mapreduce.job.user.classpath.first & mapreduce.task.classpath.user.precedence：当同一个class同时出现在用户jar包和hadoop jar中时，优先使用哪个jar包中的class，默认为false，表示优先使用hadoop jar中的class。

4. mapreduce.input.fileinputformat.split.minsize: FileInputFormat做切片时的最小切片大小，(5)mapreduce.input.fileinputformat.split.maxsize:  FileInputFormat做切片时的最大切片大小 (切片的默认大小就等于blocksize，即 134217728)