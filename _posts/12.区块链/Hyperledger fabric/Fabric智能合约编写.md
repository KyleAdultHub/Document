---
title: Fabric 智能合约编写
date: "2019-06-11 20:22:00"
categories:
- 区块链
- Hyperledger fabric
tags:
- 区块链
- Fabric
toc: true
typora-root-url: ..\..\..
---



### Chaincode 介绍

#### chaincode作用

Fabric的Chaincode是一段运行在容器中的程序。Chaincode是客户端程序和Fabric之间的桥梁。

通过Chaincode客户端程序可以发起交易，查询交易。

#### chaincode 运行环境

Chaincode是运行在Dokcer容器中，因此相对来说安全。

#### 支持语言

目前支持 java,node，go,go是最稳定的。其他还在完善。

#### chaincode 运行和使用步骤

1. 创建代码目录

   mkdir -p  {path}

2. 存放编写好的chaincode源代码

3. 部署chaincode

   ```shell
   peer chaincode install -n {chaincodeId} -v {version} -p {path}
   ```

4. 实例化chaincode

   ```shell
   peer chaincode instantiate -o {order_address} -C {channel} -n {chaincodeId} -v {version} -c '{"Args": ["init", "a", "100", "b", "200"]}' -P "OR('Org1MSP.member', 'Org2MSP.member')"
   ```

5. 调用chaincode

   ```shell
   peer chaincode invoke -o {order_address} -C {chainnel} -n {chaincodeId} -v {version} -c '{"Args":["invoke", "1", "a", "b"]}'
   ```

### Golang 版Chaincode 编写规范

#### Chaincode 代码结构

##### 包名

一个chaincode通常是一个 Goalng源文件，包名必须是main ——   package main

##### shim包

“github.com/hyperledger/fabric/core/shaincode/shim”

pb  "github.com/hyperledger/fabric/protos/peer"

shim提供了Fabric系统的上下文环境，包含了Chaincode和Fabic交互的接口。

在Chaincode中，执行赋值，查询，等功能都是需要通过shim.

##### 定义结构体

```go
type chainCodeExample struct {}
```

chaincode必须定义一个结构体，结构体的名称可以是任意符合Golang命名规范的字符串，并且必须实现Init 和 Invoke 接口方法；

##### 实现Init方法

```go
func (t *chainCodeExample) Init(stub shim.ChaincodeStubInterface) pb.Response {
    Func, args := stub.GetFunvtionAndParameters()
}
```

Init方法是系统初始化方法，当执行命令 peer chaincode instantiate 实例化chaincode的时候调用该方法；

##### 实现Invoke方法

```go
func (t *chainCodeExample) Invoke (stub shim.ChaincodeStubInterface) pb.Response {
    Func, args := stub.GetFunvtionAndParameters()
}
```

Invoke方法主要是写入数据等对链进行查询和插入修改等操作的主要方法；

#### shim包核心方法

##### Success

```go
func (t *chainCodeExample) Invoke (stub shim.ChaincodeStubInterface) pb.Response {
    Func, args := stub.GetFunvtionAndParameters()
    return shim.Success([]byte("success invoke"))
}
```

Success 方法负责将正确的消息返回给调用chaincode 的客户端

##### Error

```go
func (t *chainCodeExample) Invoke (stub shim.ChaincodeStubInterface) pb.Response {
    Func, args := stub.GetFunvtionAndParameters()
    return shim.Success([]byte("success invoke"))
}
```

Error方法将错误的信息返回给调用Chaincode的客户端。

##### LogLevel

```go
func (t *chainCodeExample) Invoke (stub shim.ChaincodeStubInterface) pb.Response {
    logLevel, _ := shim.LogLevel("DEBUG")
    shim.SetLoggingLevel(logLevel)
    return shim.Success([]byte("success invoke and not opter!"))
}
```

LogLevel 方法负责修改Chaincode 中运行日志级别

#### Chaincode StubInterface 接口方法

shim包提供了一个接口 ChaincodeStubInte， 在Invoke方法和Init 方法中该接口作为参数传入方法中；

##### **ChaincodeStubInte 主要方法**

- GetFunctionAndParameters                   获取调用链码的方法名和参数列表
- PutState                                                      存储数据到账本中
- DelState                                                       删除账本中的数据
- GetState                                                       从账本中获取指定数据
- CreateCompositeKey                                 创建符合键
- GetStateByPartialCompositeKey            通过符合键取值
- SplitCompositeKey                                     拆分复合键
- GetStateByRange                                        查询指定key 指定范围的历史记录
- GetHistoryForKey                                        获取指定key的历史记录
- GetTxID                                                         获取交易编号
- GetTxTimestamp                                         获取交易的时间
- GetCreator                                                    获取交易的创建者
- InvokeChaincode                                         调用其他的链码

##### 1、 PutState方法

将调用者的数据存储到Fabric链上

```go
func (t *chainCodeExample) Invoke (stub shim.ChaincodeStubInterface) pb.Response {
    stub.PutState("a", []byte("test"))
    return shim.Success([]byte("success invoke"))
}
```

##### 2、 GetState方法

将调用者的数据存储到Fabric链上

```go
func (t *chainCodeExample) Invoke (stub shim.ChaincodeStubInterface) pb.Response {
    var keyValue []byte
    var err error
    keyValue, err = stub.GetState("getKey")
    if err != nil {
        retutn shim.Error("find error")
    }
    reutrn shim.Success(keyValue)
}
```

##### 3、GetStateByRange方法

根据Key的范围来查询相关数据

```go
func (t *chainCodeExample) Invoke (stub shim.ChaincodeStubInterface) pb.Response {
    startKey := "startKey"
    enKey := "endKey"
    keysIter, err := stub.GetStateByRange(startKey, endKey)
    if err != nil {
        return shim.Error(fmt.Sprintf("GetStateByRange find err: %s", err))
    }
    defer keysIter.Close()
    var keyValues map[string]string
    for keysIter.HasNext() {
        response, iterError := keysIter.Next()
        if iterError != nil {
            return shim.Error(fmt.Sprintf("find and error %s", iterErr))
        }
        keyValues[response.Key] = response.Value
    }
    for key, value := range keyValues {
        fmt.Printf("key %d contains %s\n", key, value)
    }
    jsonKeys, err := json.Marshal(keys)
    if err != nil {
        return shim.Error(fmt.Sprintf("find error on marshaling json: %s", err))
    }
    return shim.Success(jsonKeys)
}
```

#####  4、GetHistoryForKey方法

查询某个键的历史记录

```go
func (t *chainCodeExample) Invoke (stub shim.ChaincodeStubInterface) pb.Response {
	keyName := "key"
    keysIter, err := stub.GetHistoryForKey(keyName)
    if err != nil {
        return shim.Error(fmt.Sprintf("GetHistoryForKey failed err: %s", err))
    }
    defer keysIter.Close()
    var keys []string
    for keysIter.HasNext() {
        response, iterError := keysIter.Next()
        if iterError != nil {
            return shim.Error(fmt.Sprintf("iter occur error %s", iterErr))
        }
        txid := response.TxId
        txvalue := response.Value
        txstatus := response.IsDelete
        txtimesamp := response.Timestamp
        tm := time.Unix(txtimesamp.Seconds, 0)
        datestr := tm.Format("2006-01-02 03:04:05 PM")
        fmt.Printf("Tx info - txid: %s value: %s if delete: %t datetime: %s \n", txid, string(txvalue), txstatus, datestr)
        keys = append(keys, txid)
    }
    jsonKeys, err := json.Marshal(keys)
    if err != nil {
        return shim.Error(fmt.Sprintf("find error on marshaling json: %s", err))
    }
    return shim.Success(jsonKeys)
}
```

##### 5、DelState 方法

```go
func (t *chainCodeExample) Invoke (stub shim.ChaincodeStubInterface) pb.Response {
    delKey := "testKey"
    err := stub.DelState(delKey)
    if err != nil {
        return shim.Error("delete error !")
    }
    reutrn shim.Success([]byte("delete success !"))
}
```

##### 6、CreateCompositeKey 方法

负责创建组合键

```go
func (t *chainCodeExample) Invoke (stub shim.ChaincodeStubInterface) pb.Response {
	params := []string("a", "b", "c")
	cparams := "test_params"
	keyName := "testKey"
	cKey, _ := stub.CreateCompositeKey(keyName, params)
	err := stub.PutState(ckey, []byte(c_params))
	if err != nil {
        fmt.Println("find errors %s", err)
	}
	return shim.Success([]byte(cKey))
}
```

##### 7、GetStateByPartialCompositeKey  和 SplitCompositeKey 方法

用来查询复合键的值

```go
func (t *chainCodeExample) Invoke (stub shim.ChaincodeStubInterface) pb.Response {
	params := []string("a", "b")
	keyName := "testKey"
	response, err := stub.GetStateByPartialCompositeKey(keyName, params)
	if err != nil {
        error_str := fmt.Sprintf("find error: %s", err)
        return shim.Error(error_str)
	}
	defer response.Close()
	var i int
	var tlist []string
	for i = 0; response.HasNext; i++ {
        responseRange, err := response.Next()
        if err != nil {
            err_str := fmt.Springf("find error: %s", err)
            fmt.Println(err_str)
            return shim.Error(error_str)
        }
        value1, compositeKeyParts, _ := stub.SplitCompositeKey(responseRange.Key)
        value2 := compositeKeyParts[0]
        value3 := compositeKeyParts[1]
        fmt.Printf("find value v1:%s v2:%s v3:%s\n", value1, value2, value3)
	}
	return shim.Success("success")
}
```

##### 8、GetTxTimestamp方法

负责好偶去当前客户端发送交易的时间戳

```go
func (t *chainCodeExample) Invoke (stub shim.ChaincodeStubInterface) pb.Response {
    txTime, err := stub.GetTxTimestamp()
    if err != nil {
        fmt.Printf("Error getting transaction timestamp: %s", err)
        return shim.Error(fmt.Springf("Error getting transaction timestamp: %s', err))
    }
    tm := time.Unix(txTime.Seconds, 0)
    fmt.Printf("Transaction Time: %v\n", time.Format("2006-01-02 03:04:05 PM"))
    return shim.Success([]byte(fmt.Springf("time is : %s", tm.Format("2006-01-02 03:04:05 PM"))))
}
```

##### 9、InvokeChaincode方法

用来在chaincode中调用其他的chaincode链码

```go
func (t *chainCodeExample) Invoke (stub shim.ChaincodeStubInterface) pb.Response {
    chaincodeId := "chaincode2"
    channelId := "channel2"
    params1 := []string{"query", "a"}
    queryArgs := make([][]byte, len(params1))
    for i, arg := range params1 {
        queryArgs[i] = []byte(arg)
    }
    response := stub.InvokeChaincode(chaincodeId, params1, channelId)
    if response.Status != shim.OD {
        errStr := fmt.Sprintf("Failed to query chaincode, got error: %s", response.Payload)
        fmt.Printf(errStr)
        return shim.Error(errStr)
    }
    result := string(response.Payload)
    fmt.Printf("invoke chaincode %s", result)
    return shim.Success([]byte("success invoke chaincode and not opter!"))
}
```

### Chaincode 操作命令

```shell
Available Commands:
	install
	instantiate
	invoke
	list
	package
	query
	signpackage
	upgrade
Flags:
		--cafile
    -o, --orderer
        --tls
        --transient
Global Flags:
		--logging-level
		--test.coverprofile
	-v, --version
```

### Chaincode背书规则指定

#### 背书介绍

Fabaric 中对数据参与方对数据的确认是真实通过Chaincode来进行的。

**什么是背书呢？**

背书就是仪表交易被确认的过程。大概意思就是交易你必须背会一本书才能操作。

背书策略被用来指示对相关的参与方如何对交易进行确认。当一个节点接收到一个交易请求的时候，会调用vscc系统（系统Chaincode，专门负责处理背书相关的操作）与交易的Chaincode共同来验证交易的合法性。在vscc和交易的 Chaincode共同对交易的确认中，通常会做一下的校验。

1. 所有的背书是否有效
2. 参与背书的数量是否满足要求
3. 所有背书参与方是否满足要求

#### 指定背书规则的方法

背书策略的设置是通过Chaincode部署时instantiate命令中的-p参数来设置的。

```shell
peer chaincode instantiate -o {order_address} -C {channel} -n {chaincodeId} -v {version} -c '{"Args": ["init", "a", "100", "b", "200"]}' -P "OR('Org1MSP.member', 'Org2MSP.member')"
```

这个参数包说明是当前Chaincode发起的交易，需要组织编号为 Org1MSP的组织编号为Org2MSP的组织中的任何一个用户共同参与交易的确认并且同意。这样交易才能生效并且记录到 区块链中。

通过上述背书策略的示例我们可以知道背书策略是通过一定的关键字和系统的属性组成的。

#### 背书编写示例

1. 逻辑与关系

   ```shell
   AND('Org1MSP.member', 'Org2MSP.member', 'Org3MSP.member')
   ```

2. 逻辑或关系

   ```shell
   OR('Org1MSP.member', 'Org2MSP.member', 'Org3MSP.member')
   ```

3. 逻辑与或关系并存

   ```shell
   OR('Org1MSP.member', AND('Org2MSP.member', 'Org3MSP.member'))
   ```
