---
title: hadoop 编译
date: "2019-12-29 14:00:00"
categories:
- 大数据
- hadoop
tags:
- 大数据
- hadoop
toc: true
typora-root-url: ..\..\..
---

## 编译hadoop步骤

### 下载依赖

1. 下载编译需要的软件包
   - apache-ant-1.9.4-bin.tar.gz
   - findbugs-3.0.0.tar.gz
   - protobuf-2.5.0.tar.gz
   - apache-maven-3.0.5-bin.tar.gz

2. 下载并解压缩hadoop源码包

   ```shell
    tar -zxvf hadoop-2.4.0-src.tar.gz
   ```

3. 安装各依赖软件

   **安装maven**

   ```shell
   tar -zxvf apache-maven-3.0.5-bin.tar.gz -C /opt/
   vi /etc/profile
   # 增加maven的bin目录到path
   source /etc/profile
   ```

   **安装ant**

   ```shell
   tar -zxvf apache-ant-1.9.4-bin.tar.gz -C /opt/
   vi /etc/profile
   # 增加ant 的bin目录到path
   source /etc/profile
   ```

   **安装findbugs**

   ```shell
   tar -zxvf findbugs-3.0.0.tar.gz -C /opt/
   vi /etc/profile
   # 增加findbugs 的bin目录到path
   source /etc/profile
   ```

   **安装protubuf**

   ```shell
   tar -zxvf protobuf-2.5.0.tar.gz
   cd protobuf-2.5.0
   ./configure
   make
   make check
   make install 
   ```

4. 安装linux依赖

   ```shell
   yum install -y cmake openssl-devel ncurses-devel
   ```

5. 编译hadoop

   ```shell
   cd hadoop-2.4.0-src
   mvn clean install -DskipTests
   mvn package -Pdist,native -DskipTests -Dtar  # 等待编译完成
   ```

6. 编译成功的文件在： /hadoop-2.4.0-src/hadoop-dist/target/