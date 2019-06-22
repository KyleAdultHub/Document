---
title: go-micro 框架使用
date: "2019-5-22 15:41:00"
categories:
- Golang
- Golang工具包
tags:
- Golang
- Micro
toc: true
typora-root-url: ..\..\..
---

### Micro介绍

#### 什么是Micro

Micro是一个着眼于分布式系统开发的微服务生态系统。

Micro由开源的库与工具组成，旨在辅助微服务开发。

#### Micro开源库

- **go-micro** - 基于Go语言的可插拔RPC微服务开发框架；包含服务发现、RPC客户/服务端、广播/订阅机制等等。
- **go-plugins** - go-micro的插件, 其包含etcd、kubernetes、nats、rabbitmq、grpc等等。
- **micro** - 微服务工具集包含传统的入口点（entry point）；API 网关、CLI、Slack Bot、代理及Web UI。
- 文档位置: https://micro.mu/docs/

> 下面主要介绍使用go-micro  对service 的编写注册和启动流程，关于micro的工具箱的使用，可以参考上面提供的文档地址

### 使用GO-Micro编写服务端

#### 环境准备

```shell
brew install protobuf     # 下载protoc工具
go get github.com/micro/go-micro       # 安装go-micro
go get github.com/golang/protobuf/{proto,protoc-gen-go}    
go get github.com/micro/{protoc-gen-micro,micro}
```

#### go-micro  Service接口

先看一下go-micro的service interface，是构建micro服务所需的主要组件。它把所有Go-Micror的基础包打包成单一组件接口。

接下来对于服务的开发都将主要围绕service 接口来进行

```go
type Service interface {
    Init(...Option)
    Options() Options
    Client() client.Client
    Server() server.Server
    Run() error
    String() string
}
```

#### 编写.proto文件，定义Service 的Api

我们使用protobuf文件来定义服务的API接口。使用protobuf可以非常方便去严格定义API，提供服务端与客户端双边具体一致的类型。

下面是定义的示例

`greeter.proto`

```go
syntax = "proto3";

service Greeter {
    rpc Hello(HelloRequest) returns (HelloResponse) {}
}

message HelloRequest {
    string name = 1;
}

message HelloResponse {
    string greeting = 2;
}
```

我们定义了一个服务叫做Greeter的处理器，它有一个接收HelloRequest并返回HelloResponse的Hello方法。

> .proto文件的详细定义方法可以参考: https://www.jianshu.com/p/ea656dc9b037

#### 生成API接口 

我们需要下面这个工具来生成protobuf代码文件，它们负责生成定义的go代码实现。

```shell
protoc --proto_path=$GOPATH/src:. --micro_out=. --go_out=. greeter.proto
```

生成的类现在可以引入**handler**中，在服务或客户端来创建请求了。

**下面是部分生成的代码**

handler的开发，将会直接饮用生成的Request  和Response 等对象。

```go
type HelloRequest struct {
    Name string `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
}

type HelloResponse struct {
    Greeting string `protobuf:"bytes,2,opt,name=greeting" json:"greeting,omitempty"`
}

// Greeter service 客户端的API

type GreeterClient interface {
    Hello(ctx context.Context, in *HelloRequest, opts ...client.CallOption) (*HelloResponse, error)
}

type greeterClient struct {
    c           client.Client
    serviceName string
}

func NewGreeterClient(serviceName string, c client.Client) GreeterClient {
    if c == nil {
        c = client.NewClient()
    }
    if len(serviceName) == 0 {
        serviceName = "greeter"
    }
    return &greeterClient{
        c:           c,
        serviceName: serviceName,
    }
}

func (c *greeterClient) Hello(ctx context.Context, in *HelloRequest, opts ...client.CallOption) (*HelloResponse, error) {
    req := c.c.NewRequest(c.serviceName, "Greeter.Hello", in)
    out := new(HelloResponse)
    err := c.c.Call(ctx, req, out, opts...)
    if err != nil {
        return nil, err
    }
    return out, nil
}

// Greeter service 服务端

type GreeterHandler interface {
    Hello(context.Context, *HelloRequest, *HelloResponse) error
}

func RegisterGreeterHandler(s server.Server, hdlr GreeterHandler) {
    s.Handle(s.NewHandler(&Greeter{hdlr}))
}
```

#### 实现handler处理器 

服务端需要注册**handlers**，这样才能提供服务并接收请求。处理器相当于是一个拥有公共方法的公共类，它需要符合签名`func(ctx context.Context, req interface{}, rsp interface{}) error`

通过上面的内容，我们看到，Greeter interface的签名的看上去就是这样：

```go
type GreeterHandler interface {
        Hello(context.Context, *HelloRequest, *HelloResponse) error
}
```

Greeter处理器实现：

```go
import proto "github.com/micro/examples/service/proto"

type Greeter struct{}

func (g *Greeter) Hello(ctx context.Context, req *proto.HelloRequest, rsp *proto.HelloResponse) error {
    rsp.Greeting = "Hello " + req.Name
    return nil
}
```

#### 服务启动

1. **创建Service服务**

   可以使用`micro.NewService`创建服务

   ```go
   import "github.com/micro/go-micro"
   service := micro.NewService() 
   ```

   初始化时，也可以传入相关选项

   ```go
   service := micro.NewService(
           micro.Name("greeter"),
           micro.Version("latest"),
   )
   ```

   所有的可选参数参考：[配置项](https://godoc.org/github.com/micro/go-micro#Option)

   Go Micro也提供通过命令行参数`micro.Flags`传递配置参数：

   ```go
   import (
           "github.com/micro/cli"
           "github.com/micro/go-micro"
   )
   
   service := micro.NewService(
           micro.Flags(
                   cli.StringFlag{
                           Name:  "environment",
                           Usage: "The environment",
                   },
           )
   )
   ```

2. **初始化服务**

   初始化服务时候，可以解析命令行标识参数，增加标识参数可以使用`micro.Action`选项：

   ```go
   service.Init(
           micro.Action(func(c *cli.Context) {
                   env := c.StringFlag("environment")
                   if len(env) > 0 {
                           fmt.Println("Environment set to", env)
                   }
           }),
   )
   ```

   Go Micro提供预置的标识，`service.Init`执行时就会设置并解析这些参数。所有的标识[参考](https://godoc.org/github.com/micro/go-micro/cmd#pkg-variables).

3. **注册服务**

   处理器会与服务一起被注册，就像http处理器一样。

   ```go
   service := micro.NewService(
       micro.Name("greeter"),
   )
   
   proto.RegisterGreeterHandler(service.Server(), new(Greeter))
   ```

4. **运行服务 Service**

   服务可以调用`server.Run`运行起来。这一步会让服务绑到配置中的地址（默认遵循RFC1918，分配随机的端口）接收请求。

   另外，这一步会在服务启动时向注册中心`注册`，并在服务接收到关闭信号时`卸载`。

   ```go
   if err := service.Run(); err != nil {
       log.Fatal(err)
   }
   ```

   之后可以通过 go run greeter.go

   > 注: 如果需要更改register为 consul， 更改环境变量 export MICRO_REGISTRY=consul

#### 完成的服务 

greeter.go

```go
package main

import (
        "log"

        "github.com/micro/go-micro"
        proto "github.com/micro/examples/service/proto"

        "golang.org/x/net/context"
)

type Greeter struct{}

func (g *Greeter) Hello(ctx context.Context, req *proto.HelloRequest, rsp *proto.HelloResponse) error {
        rsp.Greeting = "Hello " + req.Name
        return nil
}

func main() {
        service := micro.NewService(
                micro.Name("greeter"),
                micro.Version("latest"),
        )

        service.Init()

      
    proto.RegisterGreeterHandler(service.Server(), new(Greeter))

        if err := service.Run(); err != nil {
                log.Fatal(err)
        }
}
```

需要注意的是，要保证服务发现机制运行起来，这样服务才能注册，其它服务或客户端才能发现它。

#### 对服务端进行测试

**使用命令**

```shell
micro call SrvName funcName  args 
```

**使用示例**

```shell
micro call go.micro.srv.demo Demo.Call "{\"name\": \"John\"}"
```

### 使用GO-Micro编写客户端 

Client包用于查询服务，当创建服务时，也包含了一个客户端，这个客户端匹配服务所使用的初始化包。

查询上面的服务很简单：

```go
// 创建greate客户端，这需要传入服务名与服务的客户端方法构建的客户端对象
greeter := proto.NewGreeterClient("greeter", service.Client())

// 在Greeter handler上请求调用Hello方法
rsp, err := greeter.Hello(context.TODO(), &proto.HelloRequest{
    Name: "John",
})
if err != nil {
    fmt.Println(err)
    return
}

fmt.Println(rsp.Greeter)
```

`proto.NewGreeterClient` 需要服务名与客户端实例来请求服务。