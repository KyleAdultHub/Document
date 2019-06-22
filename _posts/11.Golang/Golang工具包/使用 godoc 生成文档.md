---
title: 使用 godoc 生成文档
date: "2019-4-29 13:45:00"
categories:
- Golang
- Golang工具包
tags:
- Golang
toc: true
typora-root-url: ..\..\..
---

### 代码注释规范

godoc 支持package、const、 var 和func 这些代码生成文档，而且只会对首字母大写自动生成，而小写的私有方法不会被生成到文档，下面介绍下各种注释的使用方式

#### Package

1. 在package name 代码上面，紧挨着代码进行注释
2. 注释内容中间不能有空行
3. 如果一个包中有多个包注释，就会把多个包的注释放在一起，并按照文件名的首字母顺序排序
4. 注释的格式为 package name [summery]     （summery可以是多行）
5. 注释最前面一句话会模块的summary会出现在package index中，第一句话以及之后的内容会出现在OverView中

**注释代码**

```go
/*
A web framework includes app server, logger, panicer, util and so on.
 */
 package document
```

**文档生成效果**

![1556538100747](/img/1556538100747.png)

#### 常量、变量和函数

1. 紧挨着定义代码，在常量、变量和函数上面进行注释
2. 注释格式为: FunctionName Summary      （Summary可以是多行）
3. 想圈起来说明参数可以加缩进， 进行预格式化
4. 注释最前面一句话会出现在package index中，第一句话以及之后的内容会出现在OverView中

5. 常量的summary会放到[ Constants ] 里， 变量的summary会放到[ Variables ] 里， 函数的summary会放到 [ func ] 中

**注释代码**

```
// Marshaler is the interface implemented by objects that
/*
can marshal themselves into valid JSON.
*/ 
type Marshaler interface {
	MarshalJSON() ([]byte, error)
}
```

**文档生成效果**

![1556538998023](/img/1556538998023.png)

#### BUG

1. godoc会先查找:[空格]BUG 然后显示在Package说明文档最下面

2. 如果代码中有bug，可以用BUG注释, 它会被识别为一个 bug，可以在文档中的「Bugs」中看到。

**注释代码**

```go
BUG(who): xxx
```

#### Deprecated

1. 通过Deprecated注释的内容将不会体现在`godoc`中，但是还是挺有用的，Goland可以识别它并作出提示。

#### 注释代码

```
// Deprecated: xxx
```

#### 返回值Output标签

1. 在函数体中如果定义了Output标签，会在文档页面上展示输出内容

2. 如果没有定义将不会展示( 非必须 )
3. 一般为测试代码使用，用来展示方法的输出结果

**注释代码**

```go
func ExamplePeel() {
    fmt.Println("Hello Banana")
    
    // Output:
    // Hello Banana
}
```

### 使用doc.go书写注释

1. 如果包注释超过3行，可以把注释都迁移到doc.go文件中（可以在当前目录新建一个doc.go文件）。

2. 多行注释自然需要支持一些复杂的格式，如果单行中首字母是大写，并且结尾没有标点符号是标题(标题字体会加粗变蓝，中文加粗变蓝需要加上一个大写字母)
3. 首字母是小写，或者结果又标点符号的是段落
4. 有缩进是预格式化， 在有预格式化的注释段中，不会有标题特征

### example_PackageName_test.go 的注释规则

1. 包示例代码注释非常重要，项目的使用方法就是通过每个包例子搭建起来的, 文件必须放在当前包下

2. 示例文件需要创建一个新文件，名称格式为 example_Packagename_test.go. 不加example前缀也是可以的，但是不加前缀通常是单元测试的文件命名规则， 包名的格式为 当前包名 + _test.

3. 包中函数名称的格式为ExampleFuncName[_tag]。不加函数名的话是包级别的示例。加函数名的话是函数级别的示例。
4. 函数的注释会展示在页面上
5. 函数结果加上 // Output: 注释，可以说明函数的返回值，并展示在文档上

#### 包级别的示例

**函数名称规则**

Example

**代码示例**

```
func Example() {
	logger.Info("hello, world.")
}
```

**文档展示结果**

![1556589808763](/img/1556589808763.png)

#### 函数级别示例

**函数名称规则**

ExampleFuncName

**代码示例**

```
func ExampleNewLogger() {
    w := os.Stdout
    flag := log.Llongfile
    l := logger.NewWriterLogger(w, flag, 3)
    l.Info("hello, world")
}
```

**文档展示结果**

![1556590412296](/img/1556590412296.png)

#### Output输出

**注释格式**

OutPut:

xxxxxxx

**代码示例**

```go
// 此函数将被展示在OverView区域, 并展示noOutput标签
func Example_noOutput() {
    fmt.Println("Hello OverView")
    // (Output: )非必须, 存在时将会展示输出结果
}

// 此函数将被展示在Function区域
// Peel必须是banana包实现的方法
func ExamplePeel() {
    fmt.Println("Hello Banana")
    
    // Output:
    // Hello Banana
}
```

### 本地运行godoc 文档

```shell
 godoc -http=:6060
```

