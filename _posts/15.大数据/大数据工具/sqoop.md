---
title: sqoop
date: "2020-02-14 15:00:00"
categories:
- 大数据
- 大数据工具
tags:
- 大数据
toc: true
typora-root-url: ..\..\..
---

## Sqoop 的介绍

### Sqoop概述

sqoop是apache旗下一款**“Hadoop和关系数据库服务器之间传送数据”**的工具。

Apache Sqoop是用来实现结构型数据（如关系数据库）和Hadoop之间进行数据迁移的工具。它充分利用了MapReduce的并行特点以批处理的方式加快数据的传输，同时也借助MapReduce实现了容错。

**导入数据**：MySQL，Oracle导入数据到Hadoop的HDFS、HIVE、HBASE等数据存储系统；

**导出数据：**从Hadoop的文件系统中导出数据到关系数据库

![1581663242162](/img/1581663242162.png)

### Sqoop 支持的数据库

| **Database** | **version** | **--direct** **support?** | **connect string matches**             |
| ------------ | ----------- | ------------------------- | -------------------------------------- |
| HSQLDB       | 1.8.0+      | No                        | [jdbc:hsqldb:*//](http://jdbchsqldb*/) |
| MySQL        | 5.0+        | Yes                       | **jdbc:mysql://**                      |
| Oracle       | 10.2.0+     | No                        | [jdbc:oracle:*//](http://jdbcoracle*/) |
| PostgreSQL   | 8.3+        | Yes (import only)         | **jdbc:postgresql:/**                  |

### 工作机制

将导入或导出命令翻译成mapreduce程序来实现

在翻译出的mapreduce中主要是对inputformat和outputformat进行定制

## Sqoop安装

> 安装sqoop的前提是已经具备java和hadoop的环境

下载并解压: 

   下载地址http://ftp.wayne.edu/apache/sqoop/1.4.6/

修改配置文件

   ```
   cd $SQOOP_HOME/conf
   mv sqoop-env-template.sh sqoop-env.sh
   
   # 打开sqoop-env.sh并编辑下面几行：
   export HADOOP_COMMON_HOME=/home/hadoop/apps/hadoop-2.6.1/ 
   export HADOOP_MAPRED_HOME=/home/hadoop/apps/hadoop-2.6.1/
   export HIVE_HOME=/home/hadoop/apps/hive-1.2.1
   ```

加如mysql的jdbc驱动包

   ```shell
   cp  ~/app/hive/lib/mysql-connector-java-5.1.28.jar   $SQOOP_HOME/lib/
   ```

验证启动

   ```shell
   cd $SQOOP_HOME/bin
   sqoop-version
   ```

预期的输出:

   ```shell
   15/12/17 14:52:32 INFO sqoop.Sqoop: Running Sqoop version: 1.4.6
   
   Sqoop 1.4.6 git commit id 5b34accaca7de251fc91161733f906af2eddbe83
   
   Compiled by abe on Fri Aug 1 11:19:26 PDT 2015
   ```



## Sqoop命令介绍

### 命令详情

```shell
sqoop help
```

```shell
usage: sqoop COMMAND [ARGS]
Available commands:
codegen            Generate code to interact with database records
create-hive-table  Import a table definition into Hive
eval               Evaluate a SQL statement and display the results
export             Export an HDFS directory to a database table
help               List available commands
import             Import a table from a database to HDFS
import-all-tables  Import tables from a database to HDFS
job                Work with saved jobs
list-databases     List available databases on a server
list-tables        List available tables in a database
merge              Merge results of incremental imports
metastore          Run a standalone Sqoop metastore
version            Display version information
```

## 数据导入

### 关系型数据到导入到HDFS

**命令格式**

```shell
sqoop import (generic-args) (import-args) 
```

**命令示例**

```shell
sqoop import --connectjdbc:mysql://192.168.81.176/hivemeta2db --username root -password passwd --table sds
```
- 默认目录是/user/\${user.name}/\${tablename}，可以通过--target-dir设置hdfs上的目标目录。
- 可以通过--m设置并行数据，即map的数据，决定文件的个数。
- 如果想要将整个数据库中的表全部导入到hdfs上，可以使用import-all-tables命令
- 如果想要指定所需的列, 使用--columns 参数, 多个字段有,隔开
- 如果想导出为SequenceFiles, 使用--as-sequencefile参数，为类文件命名使用--class-name参数
- 导出文本可以指定分隔符, --fields-terminated-by,--lines-terminated-by, --optionally-enclosed-by

- 指定过滤条件, 示例： --where "sd_id > 100"

### HDFS导出到关系型数据库

**命令格式**

```shell
sqoop export (generic-args) (export-args)
```

**命令示例**

```shell
sqoop export --connect jdbc:mysql://192.168.81.176/sqoop --username root -password passwd --table sds --export-dir /user/guojian/sds
```

**sqoop事务**

上例中sqoop数据中的sds表需要先把表结构创建出来，否则export操作会直接失败。

由于sqoop是通过map完成数据的导入，各个map过程是独立的，没有事物的概念，可能会有部分map数据导入失败的情况。为了解决这一问题，sqoop中有一个折中的办法，即是指定中间 staging 表，成功后再由中间表导入到结果表。

这一功能是通过 --staging-table  staging-table-name 指定，同时staging表结构也是需要提前创建出来的:

需要说明的是，在使用 --direct ， --update-key 或者--call存储过程的选项时，staging中间表是不可用的。

结果验证:

1. 数据会首先写到sds_tmp表，导入操作成功后，再由sds_tmp表导入到sds结果表中，同时会清除sds_tmp表。

2. 如果有map失败，则成功的map会将数据写入tmp表，export任务失败，同时tmp表的数据会被保留。

3. 如果tmp中已有数据，则此export操作会直接失败，可以使用 --clear-staging-table 指定在执行前清除中间表。

**export 选项**

| 参数                                   | 说明                                                        |
| -------------------------------------- | ----------------------------------------------------------- |
| --direct                               | 直接使用 mysqlimport 工具导入mysql                          |
| --export-dir <dir>                     | 需要export的hdfs数据路径                                    |
| -m,--num-mappers <n>                   | 并行export的map个数n                                        |
| --table <table-name>                   | 导出到的目标表                                              |
| --call <stored-proc-name>              | 调用存储过程                                                |
| --update-key <col-name>                | 指定需要更新的列名，可以将数据库中已经存在的数据进行更新    |
| --update-mode <mode>                   | 更新模式，包括 updateonly (默认）和allowinsert              |
| 前者只允许更新，后者允许新的列数据写入 |                                                             |
| --input-null-string <null-string>      | The string to be interpreted as null for string columns     |
| --input-null-non-string <null-string>  | The string to be interpreted as null for non-string columns |
| --staging-table <staging-table-name>   | 指定中间staging表                                           |
| --clear-staging-table                  | 执行export前将中间staging表数据清除                         |
| --batch                                | Use batch mode for underlying statement execution.          |

### 关系型数据库导入到HIVE中

**命令格式**

```shell
sqoop import (generic-args) (create-args)
```
要指定--hive-import 参数

**命令示例**

```shell
bin/sqoop import --connect jdbc:mysql://hdp-node-01:3306/test --username root --password root --table emp --hive-import --m 1
```

- 在导入表数据到HDFS使用Sqoop导入工具，我们可以指定目标目录。用--target-dir 参数，来创建外部表

## 其他命令

### 列出数据库中所有库

**命令示例**

```shell
sqoop list-databases --connect jdbc:mysql://192.168.81.176/ --username root -password passwd
```

### 列出数据库中所有表

**命令示例**

```shell
sqoop list-tables --connect jdbc:mysql://192.168.81.176/sqoop --username root -password passwd
```

### sqoop  job

**job 相关参数**

| 参数              | 说明                         |
| ----------------- | ---------------------------- |
| --create <job-id> | 创业一个新的sqoop作业.       |
| --delete <job-id> | 删除一个sqoop job            |
| --exec <job-id>   | 执行一个 --create 保存的作业 |
| --show <job-id>   | 显示一个作业的参数           |
| --list            | 显示所有创建的sqoop作业      |

**创建job**

```shell
sqoop job --create myimportjob -- import --connectjdbc:mysql://192.168.81.176/hivemeta2db --username root -password passwd --table TBLS
```

**查看job**

```shell

```

**检查job**

```shell
sqoop job --show myjob
```

**执行job**

```shell
sqoop job --exec myjob
```

