---
title: hive
date: "2020-02-11 14:00:00"
categories:
- 大数据
- Hive
tags:
- 大数据
toc: true
typora-root-url: ..\..\..
---

## Hive基本概念

### 什么是Hive

Hive是基于Hadoop的一个数据仓库工具，可以将结构化的数据文件映射为一张数据库表，并提供类似SQL查询的功能；

### 为什么使用Hive

- 直接使用hadoop需要掌握大量的知识，上手难度较大，mapReduce开发难度太大且复杂；
- Hive接口采用类似SQL的语法，加快了开发的速度；
- 避免重复的去写MapReduce，减少了开发学习成本；
- 扩展功能比较方便；

### Hive的特点

- 可扩展

  Hive可以自由的拓展集群的规模，一般情况下不需要重启服务；

- 延展性

  Hive支持用户自定义函数，用户可以根据自己的需求来实现自己的函数；

- 容错

  良好的容错性，节点出现问题SQL仍可完成执行；

### Hive与Hadoop的关系

Hive利用HDFS存储数据，利用MapReduce查询数据

![1581400203063](/img/1581400203063.png)

### Hive与传统数据库的对比

![1581400252842](/img/1581400252842.png)

> hive具有sql数据库的外表，但应用场景完全不同，hive只适合用来做批量数据统计分析

### Hive的数据存储

1. Hive中所有的数据都存储在 HDFS 中，没有专门的数据存储格式（可支持Text，SequenceFile，ParquetFile，RCFILE等）

2. 只需要在创建表的时候告诉 Hive 数据中的列分隔符和行分隔符，Hive 就可以解析数据。

3. Hive 中包含以下数据模型：DB、Table，External Table，Partition，Bucket。

   - db：在hdfs中表现为${hive.metastore.warehouse.dir}目录下一个文件夹

   - table：在hdfs中表现所属db目录下一个文件夹

   - external table：外部表, 与table类似，不过其数据存放位置可以在任意指定路径

     > 普通表: 删除表后, hdfs上的文件都删了
     >
     > External外部表: 删除后, hdfs上的文件没有删除, 只是把文件删除了

   - partition：在hdfs中表现为table目录下的子目录

   - bucket：桶, 在hdfs中表现为同一个表目录下根据hash散列之后的多个文件, 会根据不同的文件把数据放到不同的文件中 

## Hive架构

### Hive架构图

![1581399555525](/img/1581399555525.png)

Jobtracker： 是hadoop1.x中的组件，它的功能相当于： Resourcemanager+AppMaster

TaskTracker：  相当于 Nodemanager  +  yarnchild

### 基本组成

- 用户接口: CLI、JDBC/ODBC、WebGUI

  CLI为shell命令行；JDBC/ODBC是Hive的JAVA实现，与传统数据库JDBC类似；WebGUI是通过浏览器访问Hive。

- 元数据存储：一般存储在关系型数据库中如 mysql；

  Hive 将元数据存储在数据库中。Hive 中的元数据包括表的名字，表的列和分区及其属性，表的属性（是否为外部表等），表的数据所在目录等。

- 解释器、编译器、优化器、执行器

  解释器、编译器、优化器完成 HQL 查询语句从词法分析、语法分析、编译、优化以及查询计划的生成。生成的查询计划存储在 HDFS 中，并在随后有 MapReduce 调用执行。

### Hive SQl 语句执行过程

Hql写出来以后只是一些字符串的拼接，所以要经过一系列的解析处理，才能最终变成集群上的执行的作业

1. Parser：将sql解析为AST（抽象语法树），会进行语法校验，AST本质还是字符串
2. Analyzer：语法解析，生成QB（query block）
3. Logicl Plan：逻辑执行计划解析，生成一堆Opertator Tree
4. Logical optimizer:进行逻辑执行计划优化，生成一堆优化后的Opertator Tree
5. Phsical plan：物理执行计划解析，生成tasktree
6. Phsical Optimizer：进行物理执行计划优化，生成优化后的tasktree，该任务即是集群上的执行的作业

- 结论：经过以上的六步，普通的字符串sql被解析映射成了集群上的执行任务，最重要的两步是 逻辑执行计划优化和物理执行计划优化（图中红线圈画）

## Hive 安装部署

### Hive三种模式

- 内嵌模式：元数据保持在内嵌的derby模式，只允许一个会话连接
- 本地独立模式：在本地安装Mysql，把元数据放到mySql内
- 远程模式：元数据放置在远程的Mysql数据库

### 安装步骤

1. 下载Hive安装包

   地址: <http://hive.apache.org/downloads.html>

2. 将hive安装包上传到hadoop集群，并解压

   ```shell
   tar -zxvf apache-hive-1.2.1-bin.tar.gz -C /export/servers/
   cd /export/servers/
   ln -s apache-hive-1.2.1-bin hive
   ```

3. 配置环境变量，编辑/etc/profile

   ```shell
   vi /etc/profile
   # 新增行
   # export HIVE_HOME=/export/servers/hive
   # export PATH=${HIVE_HOME}/bin:$PATH
   # 保存退出
   
   source /etc/profile
   ```

4. 修改hive配置文件

   ```shell
   cd /export/servers/hive/conf
   
   # 修改hive-env.sh文件， 将如下内容写到hive-env.sh文件中
   cp hive-env.sh template hive-env.sh
   # export JAVA_HOME=/export/servers/jdk
   # export HADOOP_HOME=/export/servers/hadoop
   # export HIVE_HOME=/export/servers/hive
   
   # 将如下信息写到hive-site.xml文件中
   touch hive-site.xml
   
   ```

   ```xml
   <configuration>
           <property>
                   <name>javax.jdo.option.ConnectionURL</name>
                   <value>jdbc:mysql://hadoop02:3306/hivedb?createDatabaseIfNotExist=true</value>
           </property>
           <property>
                   <name>javax.jdo.option.ConnectionDriverName</name>
                   <value>com.mysql.jdbc.Driver</value>
           </property>
           <property>
                   <name>javax.jdo.option.ConnectionUserName</name>
                   <value>root</value>
           </property>
           <property>
                   <name>javax.jdo.option.ConnectionPassword</name>
                   <value>root</value>
           </property>
   </configuration>
   ```

5. 安装mysql并配置hive数据库及权限

   - 安装mysql客户端

     ```shell
     yum install mysql-server
     yum install mysql
     service mysqld start
     ```

   - 配置hive元数据库

     ```shell
     create database hivedb
     ```

   - 对hive源数据库及逆行赋权，开放远程连接

     ```shell
     GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root' WITH GRANT OPTION;
     FLUSH PRIVILEGES;
     ```

6. 启动hive

   ```shell
   hive
   ```

   备注: 

   - 如果报错 Terminal initialization failed; falling back to unsupported, 将/export/servers/hive/lib 里面的jline2.12替换了hadoop 中hadoop-2.6.1/share/hadoop/yarn/lib/jline-0.09*.jar

   - 如果报错 Hive metastore database is not initialized.  执行: schematool  -dbType  mysql  -initSchema

### 使用方式

1. **交互式Shell**

   直接输入命令，启动hive客户端

   ```shell
   bin/hive
   ```

2. **Hive thrift服务**

   ![1581413799174](/img/1581413799174.png)

   **启动服务**

   启动方式（假如是在hadoop01上）:

   启动为前台: bin/hiveserver2

   启动为后台: nohup bin/hiveserver2 1>/var/log/hiveserver.log 2>/var/log/hiveserver.err &

   启动成功后, 可以在别的节点上用beeline去连接

   **客户端连接**

   方式一:

   hive/bin/beeline

   beeline>  !connet jdbc:hive2//mini1:10000

   方式二:

   bin/beeline -u jdbc:hive2://mini1:10000 -n hadoop

3. **直接执行SQL**

   直接通过hive -e 执行命令

   ```shell
   hive  -e  'sql'
   ```

## Hive基本操作

### 创建表

**建表语法**

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

- EXTERNAL关键字可以让用户创建一个外部表，在建表的同时指定一个指向实际数据的路径（LOCATION），Hive 创建内部表时，会将数据移动到数据仓库指向的路径；若创建外部表，仅记录数据所在的路径，不对数据的位置做任何改变。在删除表的时候，内部表的元数据和数据会被一起删除，而外部表只删除元数据，不删除数据。

- LIKE 允许用户复制现有的表结构，但是不复制数据。

- ROW FORMAT DELIMITED \[FIELDS TERMINATED BY char\]\[COLLECTION ITEMS TERMINATED BY char\] \[MAP KEYS TERMINATED BY char\]\[LINES TERMINATED BY char\] | SERDE serde_name [WITH SERDEPROPERTIES (property_name=property_value, property_name=property_value, ...)]

  用户在建表的时候可以自定义 SerDe 或者使用自带的 SerDe。如果没有指定 ROW FORMAT 或者 ROW FORMAT DELIMITED，将会使用自带的 SerDe。在建表的时候，用户还需要为表指定列，用户在指定表的列的同时也会指定自定义的 SerDe，Hive通过 SerDe 确定表的具体的列的数据。

- STORED AS    SEQUENCEFILE|TEXTFILE|RCFILE     如果文件数据是纯文本，可以使用 STORED AS TEXTFILE(默认)。如果数据需要压缩，使用 STORED AS SEQUENCEFILE。

- CLUSTERED BY      对于每一个表（table）或者分区， Hive可以进一步组织成桶，也就是说桶是更为细粒度的数据范围划分。Hive也是 针对某一列进行桶的组织。Hive采用对列值哈希，然后除以桶的个数求余的方式决定该条记录存放在哪个桶当中。 

  把表（或者分区）组织成桶（Bucket）有两个理由：

  （1）获得更高的查询处理效率。桶为表加上了额外的结构，Hive 在处理有些查询时能利用这个结构。具体而言，连接两个在（包含连接列的）相同列上划分了桶的表，可以使用 Map 端连接 （Map-side join）高效的实现。比如JOIN操作。对于JOIN操作两个表有一个相同的列，如果对这两个表都进行了桶操作。那么将保存相同列值的桶进行JOIN操作就可以，可以大大较少JOIN的数据量。

  （2）使取样（sampling）更高效。在处理大规模数据集时，在开发和修改查询的阶段，如果能在数据集的一小部分数据上试运行查询，会带来很多方便。

**建表实例**

创建内部表mytable

```sql
create table if not exists mytable(sid int, sname string) row format delimited fields terminated by ',' stored as textfile;
```

创建外部表pageview

```sql
create external table if not exists pageview(pageid int, page_url string comment 'The page URL')  row format delimited fields terminated by ',' location 'hdfs://192.168.11.191.9000/user/hibe/warehouse/';
```

创建分区表student_p

```sql
create table student_p(Sno int,Sname string,Sex string,Sage int,Sdept string) partitioned by(part string) row format delimited fields terminated by ','stored as textfile;
```

创建带桶的表student

```sql
create table student(id int, age int, name string) partitioned by(stat_date string)  clustered by(id) sorted by(age) into 2 buckets row format delimited fields terminated by',';
```

### 修改表

#### 增加/删除分区

**语法结构**

```shell
# 增加分区
ALTER TABLE table_name ADD [IF NOT EXISTS] partition_spec [ LOCATION 'location1' ] partition_spec [ LOCATION 'location2' ] ...

# 删除分区
ALTER TABLE table_name DROP partition_spec, partition_spec,...
```

**实例**

添加分区

```shell
alter table student_p add partition(part='a') partition(part='b');
```

删除分区

```shell
alter table student drop partition(stat_date='20140101'), partition(stat_date=20140102);
```

#### 重命名表

**语法结构**

```shell
alter table table_name RENAME TO new_table_name
```

**实例**

```shell
alter table student rename to student1;
```

#### 增加/更新列

**语法结构**

```shell
ALTER TABLE table_name ADD|REPLACE COLUMNS (col_name data_type [COMMENT col_comment], ...) 
```

```shell
ALTER TABLE table_name CHANGE [COLUMN] col_old_name col_new_name column_type [COMMENT col_comment] [FIRST|AFTER column_name]
```

**实例**

```shell
alter table student replace columns (id int, age int, name string);
```

### 显示命令

show tables:   查看所有护具库表

show databases:  查看所有数据库

show partitions  t_name:   查看所有分区

show functions:   查看所有function

desc extended t_name:   查看表的详细信息

desc formatted t_name:    格式化显示表的详细信息

### 数据查询

**语法结构**

```sql
SELECT [ALL | DISTINCT] select_expr, select_expr, ... 
FROM table_reference
[WHERE where_condition] 
[GROUP BY col_list [HAVING condition]] 
[ [CLUSTER BY col_list] | [DISTRIBUTE BY col_list] [SORT BY| ORDER BY col_list] ] 
[LIMIT number]
```

备注:

1. order by 会对输入做全局排序，因此只有一个reducer，会导致当输入规模较大时，需要较长的计算时间。

2. sort by不是全局排序，其在数据进入reducer前完成排序。因此，如果用sort by进行排序，并且设置mapred.reduce.tasks>1，则sort by只保证每个reducer的输出有序，不保证全局有序。

3. 根据distribute by指定的字段内容将数据分到同一个reducer。

4. Cluster by 除了具有Distribute by的功能外，还会对该字段进行排序。因此，常常认为cluster by = distribute by + sort by

**实例**

获取三个最大的学生数据

```sql
select id, age, name from stuent where stat_date='20140101' order by age desc limit 3;
```

查询学生信息，按照年龄降序排序

```sql
set mapred.reduce.tasks=4;
select id, age, name from stuent distribute by age sort by age desc;
```

按照学生名称汇总学生年龄

```sql
select name, sum(age) from student group by name;
```

### 数据导入导出

#### Load

**语法结构**

```shell
LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE] INTO 
TABLE tablename [PARTITION (partcol1=val1, partcol2=val2 ...)]
```

备注: 

1. Load 操作只是单纯的复制/移动操作，将数据文件移动到 Hive 表对应的位置。

2. LOCAL关键字

   如果指定了 LOCAL， load 命令会去查找本地文件系统中的 filepath。

   如果没有指定 LOCAL 关键字，则根据inpath中的uri查找文件

3. OVERWRITE关键字

   如果使用了 OVERWRITE 关键字，则目标表（或者分区）中的内容会被删除，然后再将 filepath 指向的文件/目录中的内容添加到表/分区中。 

   如果目标表（分区）已经有一个文件，并且文件名和 filepath 中的文件名冲突，那么现有的文件会被新文件所替代。

**实例**

加载相对路径的数据

```shell
load data local inpath 'buckets.txt' into table student partition(stat_date='20131231');
```

加载绝对路径数据

```shell
load data local inpath '/root/app/datafile/buckets.txt' into table student partition(stat_date='20131230');
```

加载包含模式数据

```shell
load data local inpath 'hdfs://192.168.11.11:9000/user/hibe/warehouse/student/stat_date=20131230/buckets.txt' into table student partition(stat_date='20131229');
```

overwrite 关键字使用

```sql
load data local inpath 'buckets.txt' overwrite into table student partition(stat_date='20131229');
```

#### Insert

**语法结构**

```sql
# 基本模式插入
INSERT OVERWRITE/INTO TABLE tablename [PARTITION (partcol1=val1, partcol2=val2 ...)] select_statement FROM from_statement

# 多插入模式
FROM from_statement 
INSERT OVERWRITE/INTO TABLE tablename1 [PARTITION (partcol1=val1, partcol2=val2 ...)] select_statement1 
[INSERT OVERWRITE/INTO TABLE tablename2 [PARTITION ...] select_statement2] ...

# 自动分区模式(查询结果有分区字段)
INSERT OVERWRITE/INTO TABLE tablename PARTITION (partcol1[=val1], partcol2[=val2] ...) select_statement FROM from_statement

# 分桶数据导入
set mapreduce.job.reduces=4;
insert OVERWRITE/INTO TABLE tablename
select_statement FROM tablename distribute by(var1) sort by(var2 asc);
insert OVERWRITE/INTO table tablename
select_statement cluster by(var1);

# 基本数据导出
INSERT OVERWRITE/INTO [LOCAL] DIRECTORY directory1 SELECT ... FROM ...

# 多导出模式
FROM from_statement
INSERT OVERWRITE/INTO [LOCAL] DIRECTORY directory1 select_statement1
[INSERT OVERWRITE/INTO [LOCAL] DIRECTORY directory2 select_statement2] ...
```

备注:

1.  insert into 与 insert overwrite 都可以向hive表中插入数据，但是insert into直接追加到表中数据的尾部，而insert overwrite会重写数据，既先进行删除，再写入。如果存在分区的情况，insert overwrite会只重写当前分区数据。
2. LOCAL: LOCAL关键字是将数据导入到本地文件系统，如果不加LOCAL关键字是将数据导出到HDFS文件系统

**实例**

基本模式插入

```sql
insert overwrite table student partition(stat_date='20140101')
select id, age, name from student where stat_date='20131229';
```

多插入模式

```sql
from student 
insert overwrite table student partition(stat_date='20140102')
select id, age, name where stat_date='20131229'
insert overwrite table stu dent partition(stat_date='20140103')
select id, age, name where stat_date='20131229';
```

自动分区模式

```sql
insert overwrite table student1 partition(stat_date)
select id, age, stat_date from student where stat_date='20140101';
```

分通表数据导入

```sql
set mapreduce.job.reduces=4;
insert into table stu_buck
select Sno,Sname,Sex,Sage,Sdept from student distribute by(Sno) sort by(Sno asc);

insert overwrite table stu_buck
select * from student distribute by(Sno) sort by(Sno asc);

insert overwrite table stu_buck
select * from student cluster by(Sno);
```

导出文件到本地

```sql
insert overwrite local directory '/root/app/datafile/student1'
select * from student1
```

> 数据写入到文件系统时进行文本序列化，且每列用^A来区分，\n为换行符。用more命令查看时不容易看出分割符，可以使用: sed -e 's/\x01/|/g' filename来查看。

导出数据到HDFS

```sql
insert overwrite directory 'hdfs://192.168.11.191:9000/user/hive/warehouse/mystudent'
select * from student1;
```

### Join操作

**语法结构**

```sql
table1 JOIN table2 [join_condition]
  | table1 {LEFT|RIGHT|FULL} [OUTER] JOIN table2 join_condition
  | table1 LEFT SEMI JOIN table2 join_condition
```

备注:

​	Hive 支持等值连接（equality joins）、外连接（outer joins）和（left/right joins）。Hive **不支持非等值的连接**，因为非等值连接非常难转化到 map/reduce 任务。另外，Hive 支持多于 2 个表的连接。

#### Join操作的说明

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

#### Join实例

获取已经分配班级的学生姓名

```sql
select name, classname from student a join class b on (a.name = b.std_name);
```

获取尚未分配班级学生的姓名

```sql
select name, classname from student a left join class b on (a.name = b.std_name) where b.std_name = null;
```

left semi join 是 IN/EXISTS 的高效实现

只会打印左表的字段

```sql
select id, name from stuent a left semi join class b on (a.name = b.std_name)
```

## Hive Shell

### Hive命令行

**语法结构**

```shell
hive [-hiveconf x=y]* [<-i filename>]* [<-f filename>|<-e query-string>] [-S]
```

说明:

1.  -i 从文件初始化HQL。
2.  -e从命令行执行指定的HQL 

3.  -f 执行HQL脚本 
4.  -v 输出执行的HQL语句到控制台 
5.  -p <port> connect to Hive Server on port number 
6.  -hiveconf x=y Use this to set hive/hadoop configuration variables.

,**实例**

1、执行一次查询

```shell
hive -e 'select count(*) from student';
```

2、运行一个文件

```sql
hive -f query.hql
```

3、运行参数文件

```shell
hive -i initHQl.conf
# 文件内容
# set mapred.reduce.tasks = 4
```

## Hive 配置参数

### Hive参数

文档地址: *https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties*

开发Hive应用时，不可避免地需要设定Hive的参数。设定Hive的参数可以调优HQL代码的执行效率，或帮助定位问题。然而实践中经常遇到的一个问题是，为什么设定的参数没有起作用？这通常是错误的设定方式导致的。

**对于一般参数，有以下三种设定方式：**

- 配置文件 

- 命令行参数 

- 参数声明 

### 配置文件

**Hive的配置文件包括**

- 用户自定义配置文件：$HIVE_CONF_DIR/hive-site.xml 
- 默认配置文件：$HIVE_CONF_DIR/hive-default.xml 

用户自定义配置会覆盖默认配置。

另外，Hive也会读入Hadoop的配置，因为Hive是作为Hadoop的客户端启动的，Hive的配置会覆盖Hadoop的配置。

配置文件的设定对本机启动的所有Hive进程都有效。

### 命令行参数

启动Hive（客户端或Server方式）时，可以在命令行添加-hiveconf param=value来设定参数，例如：

```shell
bin/hive -hiveconf hive.root.logger=INFO,console
```

这一设定对本次启动的Session（对于Server方式启动，则是所有请求的Sessions）有效

### 参数声明

可以在HQL中使用SET关键字设定参数，例如：

```shell
set mapred.reduce.tasks=100;
```

这一设定的作用域也是session级的。

> 上述三种设定方式的优先级依次递增。即参数声明覆盖命令行参数，命令行参数覆盖配置文件设定。注意某些系统级的参数，例如log4j相关的设定，必须用前两种方式设定，因为那些参数的读取在Session建立以前已经完成了。

## Hive 自定义函数、Transform

当Hive提供的内置函数无法满足你的业务处理需要时，此时就可以考虑使用用户自定义函数（UDF：user-defined function）。

### 自定义函数分类

- UDF  作用于单个数据行，产生一个数据行作为输出。（数学函数，字符串函数）

- UDAF（用户定义聚集函数）：接收多个输入数据行，并产生一个输出数据行。（count，max）

### UDF开发步骤

1. 先开发一个java类，继承UDF， 并重载evaluate方法

   ```java
   package cn.itcast.bigdata.udf
   import org.apache.hadoop.hive.ql.exec.UDF;
   import org.apache.hadoop.io.Text;
   
   public final class Lower extends UDF{
   	public Text evaluate(final Text s){
   		if(s==null){return null;}
   		return new Text(s.toString().toLowerCase());
   	}
   }
   ```

2. 打成jar包上传到服务器

3. 将jar包添加到hive的classpath

   ```shell
   hive>add JAR /home/hadoop/udf.jar;
   ```

4. 创建临时函数与开发好的java class关联

   ```shell
   Hive>create temporary function toprovince as 'cn.itcast.bigdata.udf.ToProvince';
   ```

5. 即可在hql中使用自定义函数strip

   ```sql
   Select strip(name),age from t_test;
   ```

### Transform实现步骤

Hive的 TRANSFORM 关键字**提供了在SQL中调用自写脚本的功能**

适合实现Hive中没有的功能又不想写UDF的情况

1. 编辑weekday_mapper.py内容

   ```python
   #!/bin/python
   import sys
   import datetime
   
   for line in sys.stdin:
     line = line.strip()
     movieid, rating, unixtime,userid = line.split('\t')
     weekday = datetime.datetime.fromtimestamp(float(unixtime)).isoweekday()
     print '\t'.join([movieid, rating, str(weekday),userid])
   ```

1. 添加transform方法

   ```shell
   hive>add FILE weekday_mapper.py;
   ```

2. 使用transform进行查询

   ```
   INSERT OVERWRITE TABLE u_data_new
   SELECT
     TRANSFORM (movieid, rating, unixtime,userid)
     USING 'python weekday_mapper.py'
     AS (movieid, rating, weekday,userid)
   FROM u_data;
   ```

## Hive实战例子

### 需求

有如下访客访问次数统计表 t_access_times

| 访客 | 月份    | 访问次数 |
| ---- | ------- | -------- |
| A    | 2015-01 | 5        |
| A    | 2015-01 | 15       |
| B    | 2015-01 | 5        |
| A    | 2015-01 | 8        |
| B    | 2015-01 | 25       |
| A    | 2015-01 | 5        |
| A    | 2015-02 | 4        |
| A    | 2015-02 | 6        |
| B    | 2015-02 | 10       |
| B    | 2015-02 | 5        |
| ……   | ……      | ……       |

需要输出报表：t_access_times_accumulate

| 访客 | 月份    | 月访问总计 | 累计访问总计 |
| ---- | ------- | ---------- | ------------ |
| A    | 2015-01 | 33         | 33           |
| A    | 2015-02 | 10         | 43           |
| …….  | …….     | …….        | …….          |
| B    | 2015-01 | 30         | 30           |
| B    | 2015-02 | 15         | 45           |
| …….  | …….     | …….        | …….          |

### 实现

```sql
select A.username,A.month,max(A.salary) as salary,sum(B.salary) as accumulate
from 
(select username,month,sum(salary) as salary from t_access_times group by username,month) A 
inner join 
(select username,month,sum(salary) as salary from t_access_times group by username,month) B
on
A.username=B.username
where B.month <= A.month
group by A.username,A.month
order by A.username,A.month;
```

## HQL示例

### 表信息查询

```sql
show databases;
show tables;
desc test;
```

### 表分桶示例

```sql
set mapreduce.job.reduces=4;

drop table stu_buck;
create table stu_buck(Sno int,Sname string,Sex string,Sage int,Sdept string)
clustered by(Sno) 
sorted by(Sno DESC)
into 4 buckets
row format delimited
fields terminated by ',';


insert overwrite table student_buck
select * from student cluster by(Sno) sort by(Sage);  报错,cluster 和 sort 不能共存



insert into table stu_buck
select Sno,Sname,Sex,Sage,Sdept from student distribute by(Sno) sort by(Sno asc);

insert overwrite table stu_buck
select * from student distribute by(Sno) sort by(Sno asc);

insert overwrite table stu_buck
select * from student cluster by(Sno);
```

### 多重插入

```sql
from student
insert into table student_p partition(part='a')
select * where Sno<95011;
insert into table student_p partition(part='a')
select * where Sno<95011;
```

### 导出数据到本地

```sql
insert overwrite local directory '/home/hadoop/student.txt'
select * from student;
```

