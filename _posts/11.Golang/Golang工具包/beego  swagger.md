---
title: beego swagger
date: "2019-01-02 11:00:00"
categories:
- Golang
- Golang工具包
tags:
- Golang
- Micro
toc: true
typora-root-url: ..\..\..
---

## beego swager 自动化文档配置

### 配置区域

1. **全局配置**

   全局配置是对整个项目api的介绍和版本和名称相关的配置，配置位置在/routers/router.go，配置在router文件的最顶部

   示例:

   ```go
   // @APIVersion 			1.0.0
   // @Title 				项目的标题
   // @Description 		项目的描述.
   // @Contact 			xxx@xxx.com
   // @TermsOfServiceUrl   termsOfServiceUrl
   // @License				licence
   // @LicenseUrl          licenceUrl
   package  routers
   ```

   全局的注释如上所示，是显示给全局应用的设置信息，有如下这些设置

   - @APIVersion： 项目版本
   - @Title:   项目标题
   - @Description:  项目描述
   - @Contact: 联系方式
   - @License：支持证书
   - @LicenseUrl: 证书地址

2. **路由配置**

   目前自动化文档只支持如下的写法的解析，其他写法函数不会自动解析，即 namespace+Include 的写法，而且只支持二级解析，一级版本号，二级分别表示应用模块

   注意: 还需要添加 beego.SetStaticPath("/swagger", "swagger") 这条路由规则，这样才可以路由到swagger文档

   示例:

   ```go
   func init() {
   	ns :=
   		beego.NewNamespace("/v1",
   			beego.NSNamespace("/demo",
   				beego.NSInclude(
   					&controllers.DemoController{},
   				),
   			),
   		)
   	beego.AddNamespace(ns)
   	beego.SetStaticPath("/swagger", "swagger")
   }
   ```

3. **应用配置**

   ```go
   package controllers
   
   import (
   	"beego-swagger/models"
   	"fmt"
   	"github.com/astaxie/beego"
   	"strconv"
   	"time"
   )
   
   // Controller 的标题
   type DemoController struct {
   	beego.Controller
   }
   
   func (c *DemoController) URLMapping() {
   	c.Mapping("GetDemo", c.GetDemo)
   	c.Mapping("PostDemo", c.PostDemo)
   }
   
   // @Title 					getDemo api的标题
   // @Description 			getDemo api的介绍信息
   // Param 					api 参数信息, 包含五个参数: 1.参数名 2.参数类型(query、path、header) 3.参数数据类型 4.是否必须 5.注释
   // @Param 					key path string true 路径参数key,1-100之间的数字
   // @Param 					param query string false url参数param
   // Success 					成功相应消息, 接受三个参数: 1.status code 2.参数类型，必须用{}括起来 3.对象的字符串信息，如果是 {object} 类型，那么 bee 工具在生成 docs 的时候会扫描对应的对象，这里填写的是想对你项目的目录名和对象
   // @Success 				200 {object} models.SuccessGetResponse
   // Failure 					失败相应消息, 接受两个参数: 1.status code 2.错误信息
   // @Failure 				400 Invalid email supplied
   // @Failure 				500 get getDemo response error
   // router 					路由信息, 接受两个参数: 1.请求的路由地址 2.支持的请求方法, 放在[]中，可以写多个用,隔开
   // @router        			/getdemo/:key [get]
   func (c *DemoController) GetDemo() {
   	rsp := models.SuccessGetResponse{
   		Time:    time.Now().Format("2006-01-02 15:04:05"),
   	}
   	keyParam := c.Ctx.Input.Param(":key")
   	if keyParam == "" {
   		c.Ctx.Output.Status = 400
   	} else {
   		keyInt, err := strconv.Atoi(keyParam)
   		if err != nil ||  (1>keyInt || keyInt>100) {
   			c.Ctx.Output.Status = 400
   		} else {
   			rsp.Message = fmt.Sprintf("receire key: %s, param: %s", keyParam, c.GetString("param"))
   		}
   	}
   	c.Data["json"] = rsp
   	c.ServeJSON()
   }
   
   // @Title 		 	   postDemo api的标题
   // @Description        postDemo api的介绍信息
   // Param               api 参数信息, 包含五个参数: 1.参数名 2.参数类型(formData、query、path、body、header) 3.参数数据类型 4.是否必须 5.注释
   // @Param   		   form1 formData int true formData参数1,必须是偶数
   // @Param              form2 formData string false formData参数2
   // @Param              X-header header string true header参数
   // Success 			   成功相应消息, 接受三个参数: 1.status code 2.参数类型，必须用{}括起来 3.对象的字符串信息，如果是 {object} 类型，那么 bee 工具在生成 docs 的时候会扫描对应的对象，这里填写的是想对你项目的目录名和对象
   // @Success 		   200 {object} models.SuccessPostResponse
   // Failure             失败相应消息, 接受两个参数: 1.status code 2.错误信息
   // @Failure 		   400 no enough input
   // @Failure 		   500 get postDemo response error
   // router              路由信息, 接受两个参数: 1.请求的路由地址 2.支持的请求方法, 放在[]中，可以写多个用,隔开
   // @router 			   /postdemo [post]
   func (c *DemoController) PostDemo() {
   	rsp := models.SuccessPostResponse{
   		Time:    time.Now().Format("2006-01-02 15:04:05"),
   	}
   	form1Param, err := c.GetInt("form1")
   	form2Param := c.GetString("form2")
   	header := c.Ctx.Input.Header("X-header")
   	if form1Param%2 != 0 || err != nil {
   		c.Ctx.Output.Status = 400
   	} else {
   		rsp.Message = fmt.Sprintf("receire form1: %d, form2: %s, X-header: %s", form1Param, form2Param, header)
   	}
   	c.Data["json"] = rsp
   	c.ServeJSON()
   }
   ```

   **注释解释**:

   - DemoController上面的注释， 这个是该controller模块的标题;

   - @Title

     这个 API 所表达的含义，是一个文本，空格之后的内容全部解析为 title

   - @Description

     这个 API 详细的描述，是一个文本，空格之后的内容全部解析为 Description

   - @Param

     参数，表示需要传递到服务器端的参数，有五列参数，使用空格或者 tab 分割，五个分别表示的含义如下

     1. 参数名
     2. 参数类型，可以有的值是 formData、query、path、body、header，formData 表示是 post 请求的数据，query 表示带在 url 之后的参数，path 表示请求路径上得参数，例如上面例子里面的 key，body 表示是一个 raw 数据请求，header 表示带在 header 信息中得参数。
     3. 参数类型
     4. 是否必须
     5. 注释

   - @Success

     成功返回给客户端的信息，三个参数，第一个是 status code。第二个参数是返回的类型，必须使用 {} 包含，第三个是返回的对象或者字符串信息，如果是 {object} 类型，那么 bee 工具在生成 docs 的时候会扫描对应的对象，这里填写的是想对你项目的目录名和对象，例如 `models.ZDTProduct.ProductList` 就表示 `/models/ZDTProduct` 目录下的 `ProductList` 对象。

   - @Failure

   ​	失败返回的信息，包含两个参数，使用空格分隔，第一个表示 status code，第二个表示错误信息

   - @router

     路由信息，包含两个参数，使用空格分隔，第一个是请求的路由地址，支持正则和自定义路由，和之前的路由规则一样，第二个参数是支持的请求方法,放在 `[]` 之中，如果有多个方法，那么使用 `,` 分隔。

   注意: 每一行注释内的每隔参数之间用空格隔开，并且最好只用一个空格，不然可能会产生显示问题

## 文档自动生成

**文档生成和启动步骤**

1. 项目不管是不是用go-module作为包管理工具，项目必须在 **gopath/src** 目录

2. 开启文档开关

   在配置文件中设置：`EnableDocs = true`

3. 启动beego项目, 并启动swagger文档

   ```shell
   # -gendoc=true 表示每次自动化的 build 文档，-downdoc=true 就会自动的下载 swagger 文档查看器
   bee run -gendoc=true -downdoc=true
   ```

4. 查看文档

   地址: localhost:8080/swagger/

## 文档示例

![1577952020140](/img/1577952020140.png)