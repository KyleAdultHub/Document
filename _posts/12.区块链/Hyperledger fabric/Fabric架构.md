---
title: Fabric架构
date: "2019-06-21 14:00:00"
categories:
- 区块链
- Hyperledger fabric
tags:
- 区块链
toc: true
typora-root-url: ..\..\..
---

### Fabric的交易流程

#### Fabric 节点的主要分类

![1561105355873](/img/1561105355873.png)

#### 交易七步曲

1.  **Propose**

   客户端提交交易提案，及调用了链码的请求

   ![1561105411590](/img/1561105411590.png)

2. **Execute**

   endorse节点运行仿真结果，产生读写集，并签名返回给客户端

   ![1561105485250](/img/1561105485250.png)

3. **Proposal**

   客户端接收到了endorse节点的运行结果，根据背书策略，如果背书通过，产生proposal给order节点打包区块

   ![1561105546228](/img/1561105546228.png)

4. **Order Trasantion**

   Order节点将交易打包进区块， 

   ![1561105622229](/img/1561105622229.png)

5. **Deliver**

   排序节点将打包好的区块分发给commiter节点

   ![1561105913857](/img/1561105913857.png)

6. **Validate**

   Commiter节点收到区块后，对每一个交易进行验证，如果验证通过(会进行背书策略，当前key的状态等验证)，就运行读写集，并将区块上链

   ![1561105704107](/img/1561105704107.png)

7. **Notify**

   客户端收到区块上链的Event，确认了交易成功

   ![1561105980867](/img/1561105980867.png)

#### Fabric 怎么解决双花的

在第一步双花的情况会运行成功，但是当第一次花费成功的情况下，后面节点再验证交易的读写集的时候，会因为key的状态变化而验证失败，所以双花只会在第一次的交易中成功；

### Fabric 的channel

#### system channel 

在网络启动的时候系统自动创建的channel

#### 创建channel的过程

在创建channel的时候， 会想系统channel 发送一个创建channel的请求，  产生创建channel的交易

不同channel的数据是隔离的，即一个channel会维护同一套账本

![1561107312693](/img/1561107312693.png)

#### 在channel上部署智能合约

同一个channel中的智能合约可以互相调用

不同channel的智能合约指定查询但是不能更改账本的内容

![1561107378873](/img/1561107378873.png)

### Fabric 应用开发

#### 开发流程

1. 部署Fabric网络
2. 开发并部署智能合约
3. 通过fabric-ca  和fabric-sdk  进行application层的业务逻辑开发

![1561108008604](/img/1561108008604.png)

#### 开发人员主要关心的步骤

通过fabric进行开发，我们主要需要开发的有两部分：

即上图橘黄色的部分: 1. chaincode开发； 2. Application开发

