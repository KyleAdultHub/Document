---
title: azkaban
date: "2020-02-14 13:00:00"
categories:
- 大数据
- 大数据工具
tags:
- 大数据
toc: true
typora-root-url: ..\..\..
---

## azkaban 介绍

### 为甚么使用azkaban工作流调度

- 一个完整的数据分析系统通常都是由大量任务单元组成：

  shell脚本程序，java程序，mapreduce程序、hive脚本等

- 各任务单元之间存在时间先后及前后依赖关系

- 为了很好地组织起这样的复杂执行计划，需要一个工作流调度系统来调度执行；

例如，我们可能有这样一个需求，某个业务系统每天产生20G原始数据，我们每天都要对其进行处理，处理步骤如下所示：

1. 通过Hadoop先将原始数据同步到HDFS上；
2. 借助MapReduce计算框架对原始数据进行转换，生成的数据以分区表的形式存储到多张Hive表中；
3. 需要对Hive中多个表的数据进行JOIN处理，得到一个明细数据Hive大表；
4. 将明细数据进行复杂的统计分析，得到结果报表信息；
5. 需要将统计分析得到的结果数据同步到业务系统中，供业务调用使用。

### 工作流调度实现方式

- 简单的任务调度： 直接使用linux的crontab来定义
- 负载的任务调度: 1. 开发调度平台; 2. 使用贤臣的开源调度系统, 比如ooize、azkaban等；

### Azkaban简介

Azkaban是由Linkedin开源的一歌批量工作流任务调度器。用于在一个工作流内以一个特定的顺序运行一组工作和流程。Azkaban定义了一种KV文件格式来建立任务之间的依赖关系， 并提供一个易于使用的web用户界面维护和跟踪你的工作流；

其有如下功能特点：

- Web用户界面
- 方便上传工作流
- 方便设置任务之间的关系
- 调度工作流
- 认证/授权(权限的工作)
- 能够杀死并重新启动工作流
- 模块化和可插拔的插件机制
- 项目工作区
- 工作流和任务的日志记录和审计

## Azkaban安装

将安装文件上传到集群，最好上传到安装了hive、sqoop的机器上，方便命令的执行

在当前用户目录下新建azkabantools目录，用于存放源安装文件，新建azkaban目录，用于存放azkaban运行程序；

### azkaban web 服务器安装

1. 下载安装包: azkaban-web-server-2.5.0.tar.gz

2. 解压安装包: tar -zxvf azkaban-web-server-2.5.0.tar.gz

3. 将压缩后的 azkaban-web-server-2.5.0  移动到 azkaban目录中, 并重新命名为webserver

    mv azkaban-web-server-2.5.0 ../azkaban/server

### azkaban 执行服务器安装

1. 下载安装包: azkaban-executor-server-2.5.0.tar.gz
2. 解压安装包: tar -zxvf  azkaban-executor-server-2.5.0.tar.gz

3. 将解压后的azkaban-executor-server-2.5.0 移动到 azkaban目录中,并重新命名 executor

   mv azkaban-executor-server-2.5.0  ../azkaban/executor

### azkaban 脚本导入

azkaban 执行计划是通过配置在数据库中的任务来进行调度的，所以我们需要初始化数据库的任务相关的表结构

1. 下载脚本: azkaban-sql-script-2.5.0.tar.gz
2. 解压数据库脚本:  tar –zxvf azkaban-sql-script-2.5.0.tar.gz

3. 将解压后的mysql脚本，导入到mysql中:

   ```sql
   mysql> create database azkaban;
   mysql> use azkaban;
   Database changed
   mysql> source /home/hadoop/azkaban-2.5.0/create-all-sql-2.5.0.sql;
   ```

### 创建SSL配置

参考地址: http://docs.codehaus.org/display/JETTY/How+to+configure+SSL

命令： keytool -keystore keystore -alias jetty -genkey -keyalg RSA

运行此命令后,会提示输入当前生成 keystor的密码及相应信息,输入的密码请劳记,信息如下:

 

输入keystore密码： 

再次输入新密码:

您的名字与姓氏是什么？

  [Unknown]： 

您的组织单位名称是什么？

  [Unknown]： 

您的组织名称是什么？

  [Unknown]： 

您所在的城市或区域名称是什么？

  [Unknown]： 

您所在的州或省份名称是什么？

  [Unknown]： 

该单位的两字母国家代码是什么

  [Unknown]：  CN

CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=CN 正确吗？

  [否]：  y

输入\<jetty\>的主密码

​        （如果和 keystore 密码相同，按回车）： 

再次输入新密码:

完成上述工作后,将在当前目录生成 keystore 证书文件,将keystore 考贝到 azkaban web服务器根目录中.如:cp keystore azkaban/webserver

### 配置时区

先配置好服务器节点上的时区

1. 先生成时区配置文件Asia/Shanghai, 用交互式命令tzselect即可

2. 拷贝改时间文件，覆盖系统本地的时区配置

   cp xxx /usr/share/zoneinfo/Aisa/Shanghai/etc/localtime

### 配置文件

1. azkaban web服务器配置

   进入azkaban web服务器安装目录 conf目录

   修改azkaban.properties文件

   ```ini
   #Azkaban Personalization Settings
   azkaban.name=Test                           #服务器UI名称,用于服务器上方显示的名字
   azkaban.label=My Local Azkaban                   #描述
   azkaban.color=#FF3601                              #UI颜色
   azkaban.default.servlet.path=/index         #
   web.resource.dir=web/                         #默认根web目录
   default.timezone.id=Asia/Shanghai              #默认时区,已改为亚洲/上海 默认为美国
    
   #Azkaban UserManager class
   user.manager.class=azkaban.user.XmlUserManager   #用户权限管理默认类
   user.manager.xml.file=conf/azkaban-users.xml              #用户配置,具体配置参加下文
    
   #Loader for projects
   executor.global.properties=conf/global.properties    # global配置文件所在位置
   azkaban.project.dir=projects                                                #
    
   database.type=mysql                              #数据库类型
   mysql.port=3306                                 #端口号
   mysql.host=hadoop03                            #数据库连接IP
   mysql.database=azkaban                         #数据库实例名
   mysql.user=root                             #数据库用户名
   mysql.password=root                          #数据库密码
   mysql.numconnections=100                        #最大连接数
    
   # Velocity dev mode
   velocity.dev.mode=false
   # Jetty服务器属性.
   jetty.maxThreads=25                            #最大线程数
   jetty.ssl.port=8443                                                                   #Jetty SSL端口
   jetty.port=8081                                                                         #Jetty端口
   jetty.keystore=keystore                         #SSL文件名
   jetty.password=123456                           #SSL文件密码
   jetty.keypassword=123456                       #Jetty主密码 与 keystore文件相同
   jetty.truststore=keystore                                                                #SSL文件名
   jetty.trustpassword=123456                      # SSL文件密码
    
   # 执行服务器属性
   executor.port=12321                            #执行服务器端口
    
   # 邮件设置
   mail.sender=xxxxxxxx@163.com                     #发送邮箱
   mail.host=smtp.163.com                           #发送邮箱smtp地址
   mail.user=xxxxxxxx                               #发送邮件时显示的名称
   mail.password=**********                          #邮箱密码
   job.failure.email=xxxxxxxx@163.com                #任务失败时发送邮件的地址
   job.success.email=xxxxxxxx@163.com               #任务成功时发送邮件的地址
   lockdown.create.projects=false                     #
   cache.directory=cache                             #缓存目录
   ```

2. azkaban 执行服务器配置

   进入执行服务器安装目录conf,修改azkaban.properties

   azkaban.properties 文件

   ```ini
   #Azkaban
   default.timezone.id=Asia/Shanghai             #时区
    
   # Azkaban JobTypes 插件配置
   azkaban.jobtype.plugin.dir=plugins/jobtypes        #jobtype 插件所在位置
    
   #Loader for projects
   executor.global.properties=conf/global.properties
   azkaban.project.dir=projects
    
   #数据库设置
   database.type=mysql              #数据库类型(目前只支持mysql)
   mysql.port=3306              #数据库端口号
   mysql.host=192.168.20.200      #数据库IP地址
   mysql.database=azkaban          #数据库实例名
   mysql.user=azkaban                #数据库用户名
   mysql.password=oracle                          #数据库密码
   mysql.numconnections=100                       #最大连接数
    
   # 执行服务器配置
   executor.maxThreads=50                         #最大线程数
   executor.port=12321                           #端口号(如修改,请与web服务中一致)
   executor.flow.threads=30                     #线程数
   ```

3. 用户配置

   进入azkaban web服务器conf目录,修改azkaban-users.xml， 增加管理员用户

   azkaban-users.xml 文件

   ```xml
   <azkaban-users>
           <user username="azkaban" password="azkaban" roles="admin" groups="azkaban" />
           <user username="metrics" password="metrics" roles="metrics"/>
           <user username="admin" password="admin" roles="admin,metrics" />
           <role name="admin" permissions="ADMIN" />
           <role name="metrics" permissions="METRICS"/>
   </azkaban-users>
   ```

### Azkaban启动

1. 启动web服务器

   在azkaban  web 根目录运行，服务器目录下执行启动命令

   bin/azkaban-web-start.sh

2. 启动执行服务器

   在执行服务器目录下执行启动命令, 在执行服务器根目录运行

   bin/azkaban-executor-start.sh

3. 浏览器访问web 服务

   在浏览器(建议使用谷歌浏览器)中输入https://服务器IP地址:8443 ,即可访问azkaban服务了.在登录中输入刚才新的户用名及密码,点击 login.

## Azkaban实战

Azkaba内置的任务类型支持command、java

### Command类型--单一job示例

1. 创建job描述文件

   vi command.job

   ```ini
   #command.job
   type=command                                                    
   command=echo 'hello'
   ```

2. 将job资源文件打包成zip文件

   zip command.job

3. 通过azkaban的web管理平台创建project并上传job压缩包

   首先创建project

   ![1581662118887](/img/1581662118887.png)

   上传zip包

   ![1581662141679](/img/1581662141679.png)

4. 启动执行该job

   ![1581662173660](/img/1581662173660.png)

### Command 类型--多job工作流flow

1. 创建有依赖关系的多job描述

   第一个job: foo.job

   ```ini
   # foo.job
   type=command
   command=echo foo
   ```

   第二个job: bar.job 依赖foo.job

   ```ini
   # bar.job
   type=command
   dependencies=foo
   command=echo bar
   ```

2. 将所有job资源文件打到一个zip包中

3. 在azkaban的web管理界面创建工程，并上传zip包
4. 启动工作流flow

### HDFS操作人物

1. 创建job描述文件

   ```ini
   # bar.job
   type=command
   dependencies=foo
   command=echo bar
   ```

2. 将job资源文件打包成zip文件
3. 通过azkaban的web管理平台创建project并上传job压缩包
4. 启动执行该job

### MAPREDUCE 任务

Mr任务依然可以使用command的job类型来执行

1. 创建1、job描述文件，及mr程序jar包（示例中直接使用hadoop自带的example jar）

   ```ini
   # mrwc.job
   type=command
   command=/home/hadoop/apps/hadoop-2.6.1/bin/hadoop  jar hadoop-mapreduce-examples-2.6.1.jar wordcount /wordcount/input /wordcount/azout
   ```

2. 将所有job资源文件打到一个zip包中
3. 在azkaban的web管理界面创建工程并上传zip包
4. 启动job

### HIVE脚本任务

1. 创建hive脚本 test.sql

   ```sql
   use default;
   drop table aztest;
   create table aztest(id int,name string) row format delimited fields terminated by ',';
   load data inpath '/aztest/hiveinput' into table aztest;
   create table azres as select * from aztest;
   insert overwrite directory '/aztest/hiveoutput' select count(1) from aztest; 
   ```

2. 创建job描述文件: hievf.job

   ```ini
   # hivef.job
   type=command
   command=/home/hadoop/apps/hive/bin/hive -f 'test.sql'
   ```

3. 将所有job资源文件打到一个zip包中
4. 在azkaban的web管理界面创建工程并上传zip包

5. 启动job

