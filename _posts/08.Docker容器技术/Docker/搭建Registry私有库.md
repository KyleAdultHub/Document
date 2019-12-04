---
title: 搭建Registry 私有库
date: "2018-09-17 19:23:11"
categories:
- Docker容器技术
- Docker
tags:
- Docker
toc: true
---

### **1.关于Registry仓库**

官方的[Docker hub](https://hub.docker.com/)是一个用于管理公共镜像的好地方，我们可以在上面找到我们想要的镜像，也可以把我们自己的镜像推送上去。但是，有时候，我们的使用场景需要我们拥有一个私有的镜像仓库用于管理我们自己的镜像。这个可以通过开源软件Registry来达成目的。

 Registry在github上有两份代码：[老代码库](https://github.com/docker/docker-registry)和[新代码库](https://github.com/docker/distribution)。老代码是采用python编写的，存在pull和push的性能问题，出到0.9.1版本之后就标志为deprecated，不再继续开发。从2.0版本开始就到在新代码库进行开发，新代码库是采用go语言编写，修改了镜像id的生成算法、registry上镜像的保存结构，大大优化了pull和push镜像的效率。

 官方在Docker hub上提供了registry的镜像（[详情](https://hub.docker.com/_/registry/)），我们可以直接使用该registry镜像来构建一个容器，搭建我们自己的私有仓库服务。Tag为latest的registry镜像是0.9.1版本的，我们直接采用2.1.1版本。

<!-- more -->
### 2.Registry的部署

**运行下面命令获取registry镜像**

```shell
$ sudo docker pull registry:2.1.1
```

**然后启动一个容器**

```shell
$ sudo docker run -d -v /opt/data/registry/:/var/lib/registry -p 5000:5000 --restart=always --name registry registry:2.1.1
```

**验证服务是否启动成功**

说明我们已经启动了registry服务，打开浏览器输入http://127.0.0.1:5000/v2

### 3.验证

**向仓库中push镜像**

现在我们通过将镜像push到registry来验证一下。我的机器上有个hello-world的镜像，我们要通过docker tag将该镜像标志为要推送到私有仓库，

```shell
$ sudo docker tag hello-world 127.0.0.1:5000/hello-world
```

接下来，我们运行docker push将hello-world镜像push到我们的私有仓库中，

```shell
$ sudo docker push 127.0.0.1:5000/hello-world
```

```
The push refers to a repository [127.0.0.1:5000/hello-world] (len: 1)

975b84d108f1: Image successfully pushed

3f12c794407e: Image successfully pushed

latest: digest: sha256:1c7adb1ac65df0bebb40cd4a84533f787148b102684b74cb27a1982967008e4b size: 2744
```

现在我们可以查看我们本地/opt/registry目录下已经有了刚推送上来的hello-world。我们也在浏览器中输入http://127.0.0.1:5000/v2/_catalog，如下图所示，

 

**从镜像库中拉取镜像**

现在我们可以先将我们本地的127.0.0.1:5000/hello-world和hello-world先删除掉，

```shell
$ sudo docker rmi hello-world

$ sudo docker rmi 127.0.0.1:5000/hello-world
```

然后使用docker pull从我们的私有仓库中获取hello-world镜像，

```shell
 $ sudo docker pull 127.0.0.1:5000/hello-world
```

```
Using default tag: latest

latest: Pulling from hello-world

b901d36b6f2f: Pull complete

0a6ba66e537a: Pull complete

Digest: sha256:1c7adb1ac65df0bebb40cd4a84533f787148b102684b74cb27a1982967008e4b

Status: Downloaded newer image for 127.0.0.1:5000/hello-world:latest

lienhua34@lienhua34-Compaq-Presario-CQ35-Notebook-PC ~ $ sudo docker images

REPOSITORY TAG IMAGE ID CREATED VIRTUAL SIZE

registry 2.1.1 b91f745cd233 5 days ago 220.1 MB

ubuntu 14.04 a5a467fddcb8 6 days ago 187.9 MB

127.0.0.1:5000/hello-world latest 0a6ba66e537a 2 weeks ago 960 B
```

### 4.查询镜像库

**查询镜像库中的镜像**

```shell
http://127.0.0.1:5000/v2/_catalog
```

### 5.错误排查

**错误描述**

在push 到docker registry时，可能会报错：

```
The push refers to a repository [127.0.0.1:5000/registry]

Get https://127.0.0.1:5000/v1/_ping: http: server gave HTTP response to HTTPS client
```

这个问题可能是由于客户端采用https，docker registry未采用https服务所致。一种处理方式是把客户对地址“192.168.1.100:5000”请求改为http。

目前很多文章都是通过修改docker的配置文件“etc/systemconfig/docker"，重启docker来解决这个问题。但发现docker1.12.3版本并无此文件，根据网上创建此文件，并填入相应内容，重启docker无效果，仍然报此错误。

 **解决办法**

在”/etc/docker/“目录下，创建”daemon.json“文件。在文件中写入：

{ "insecure-registries":["192.168.1.100:5000"] }

保存退出后，重启docker。

