---
title: swagger从源码生成spec
date: "2019-4-29 13:45:00"
categories:
- Golang
- Golang工具包
tags:
- Golang
toc: true
typora-root-url: ..\..\..
---

### 从源码生成spec文档

#### spec 生成命令

```
Usage:
  swagger [OPTIONS] generate spec [spec-OPTIONS]

generate a swagger spec document from a go application

Application Options:
  -q, --quiet               silence logs
  -o, --output=LOG-FILE     redirect logs to file

Help Options:
  -h, --help                Show this help message

[spec command options]
      -b, --base-path=      the base path to use (default: .)
      -t, --tags=           build tags
      -m, --scan-models     includes models that were annotated with 'swagger:model'
          --compact         when present, doesn't prettify the json
      -o, --output=         the file to write to
      -i, --input=          the file to use as input
```

#### spec生成方法

##### 生成包的spec

```shell
swagger generate spec -o ./swagger.json
```

如果不提供一个mian文件给swagger， swagger 将遍历包中的左右文件以及文件的依赖并生成spec

如果想要提供给swagger一个main文件，可以在main文件中添加如下注释:

```shell
//go:generate swagger generate spec
```

它使用go工具加载器加载应用程序，然后扫描代码库使用的所有软件包。这意味着对于可被发现的东西，它需要通过主包触发的代码路径来访问。

##### 合并yml文件定义的spec

```shell
swagger generate spec -i ./swagger.yml -o ./swagger.json
```

##### 生成yaml格式的spec文件

```
swagger generate spec -o ./swagger.yml
```

#### Spec生成规则

##### swagger:meta

配置 包spec文件的一些原数据，

语法

```
swagger:meta
```

可配置的属性如下

| Annotation              | Format                                                       |
| ----------------------- | ------------------------------------------------------------ |
| **Terms Of Service**    | allows for either a url or a free text definition describing the terms of services for the API |
| **Consumes**            | a list of default (global) mime type values, one per line, for the content the API receives. [List of supported mime types](https://goswagger.io/use/spec/meta.html#supported-mime-types) |
| **Produces**            | a list of default (global) mime type values, one per line, for the content the API sends. [List of supported mime types](https://goswagger.io/use/spec/meta.html#supported-mime-types) |
| **Schemes**             | a list of default schemes the API accept (possible values: http, https, ws, wss) https is preferred as default when configured |
| **Version**             | the current version of the API                               |
| **Host**                | the host from where the spec is served                       |
| **Base path**           | the default base path for this API                           |
| **Contact**             | the name of for the person to contact concerning the API eg. John Doe <john@blogs.com> [http://john.blogs.com](http://john.blogs.com/) |
| **License**             | the name of the license followed by the URL of the license eg. MIT [http://opensource.org/license/MIT](https://opensource.org/license/MIT) |
| **Security**            | a dictionary of key: []string{scopes}                        |
| **SecurityDefinitions** | list of supported authorization types <https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#securityDefinitionsObject> |
| **Extensions**          | list of extensions to Swagger Schema. The field name MUST begin with x-, for example, x-internal-id. The value can be null, a primitive, an array or an object. |

示例:

```go
// Package classification Petstore API.
//
// the purpose of this application is to provide an application
// that is using plain go code to define an API
//
// This should demonstrate all the possible comment annotations
// that are available to turn go code into a fully compliant swagger 2.0 spec
//
// Terms Of Service:
//
// there are no TOS at this moment, use at your own risk we take no responsibility
//
//     Schemes: http, https
//     Host: localhost
//     BasePath: /v2
//     Version: 0.0.1
//     License: MIT http://opensource.org/licenses/MIT
//     Contact: John Doe<john.doe@example.com> http://john.doe.com
//
//     Consumes:
//     - application/json
//     - application/xml
//
//     Produces:
//     - application/json
//     - application/xml
//
//     Security:
//     - api_key:
//
//     SecurityDefinitions:
//     api_key:
//          type: apiKey
//          name: KEY
//          in: header
//     oauth2:
//         type: oauth2
//         authorizationUrl: /oauth2/auth
//         tokenUrl: /oauth2/token
//         in: header
//         scopes:
//           bar: foo
//         flow: accessCode
//
//     Extensions:
//     x-meta-value: value
//     x-meta-array:
//       - value1
//       - value2
//     x-meta-array-obj:
//       - name: obj
//         value: field
//
// swagger:meta
package classification
```

##### swagger:route

路径配置的方法， 此操作获取一个唯一ID，用于后面该方法的名称

语法

```
swagger:route [method] [path pattern] [?tag1 tag2 tag3] [operation id]
```

属性

| Annotation    | Format                                                       |
| ------------- | ------------------------------------------------------------ |
| **Consumes**  | a list of operation specific mime type values, one per line, for the content the API receives |
| **Produces**  | a list of operation specific mime type values, one per line, for the content the API sends |
| **Schemes**   | a list of operation specific schemes the API accept (possible values: http, https, ws, wss) https is preferred as default when configured |
| **Security**  | a dictionary of key: []string{scopes}                        |
| **Responses** | a dictionary of status code to named response                |

示例

```
// ServeAPI serves the API for this record store
func ServeAPI(host, basePath string, schemes []string) error {

    // swagger:route GET /pets pets users listPets
    //
    // Lists pets filtered by some parameters.
    //
    // This will show all available pets by default.
    // You can get the pets that are out of stock
    //
    //     Consumes:
    //     - application/json
    //     - application/x-protobuf
    //
    //     Produces:
    //     - application/json
    //     - application/x-protobuf
    //
    //     Schemes: http, https, ws, wss
    //
    //     Security:
    //       api_key:
    //       oauth: read, write
    //
    //     Responses:
    //       default: genericError
    //       200: someResponse
    //       422: validationError
    mountItem("GET", basePath+"/pets", nil)
}
```

##### swagger:parameters

参数注释结构连接到一个或多个操作。 生成的swagger spec中的参数可以由多个结构组成。

语法

```go
swagger:parameters [operationid1 operationid2]
```

参数

| Annotation            | Format                                                       |
| --------------------- | ------------------------------------------------------------ |
| **In**                | where to find the parameter                                  |
| **Collection Format** | when a slice the formatter for the collection when serialized on the request |
| **Maximum**           | specifies the maximum a number or integer value can have     |
| **Minimum**           | specifies the minimum a number or integer value can have     |
| **Multiple of**       | specifies a value a number or integer value must be a multiple of |
| **Minimum length**    | the minimum length for a string value                        |
| **Maximum length**    | the maximum length for a string value                        |
| **Pattern**           | a regular expression a string value needs to match           |
| **Minimum items**     | the minimum number of items a slice needs to have            |
| **Maximum items**     | the maximum number of items a slice can have                 |
| **Unique**            | when set to true the slice can only contain unique items     |
| **Required**          | when set to true this value needs to be present in the request |
| **Example**           | an example value, parsed as the field's type (objects and slices are parsed as JSON) |

对于slice属性，还需要定义一些项。这可能是一个嵌套集合，用于指示嵌套级别，值是一个基于0的索引

| Annotation                 | Format                                                       |
| -------------------------- | ------------------------------------------------------------ |
| **Items.n.Maximum**        | specifies the maximum a number or integer value can have at the level *n* |
| **Items.n.Minimum**        | specifies the minimum a number or integer value can have at the level *n* |
| **Items.n.Multiple of**    | specifies a value a number or integer value must be a multiple of |
| **Items.n.Minimum length** | the minimum length for a string value at the level *n*       |
| **Items.n.Maximum length** | the maximum length for a string value at the level *n*       |
| **Items.n.Pattern**        | a regular expression a string value needs to match at the level *n* |
| **Items.n.Minimum items**  | the minimum number of items a slice needs to have at the level *n* |
| **Items.n.Maximum items**  | the maximum number of items a slice can have at the level *n* |
| **Items.n.Unique**         | when set to true the slice can only contain unique items at the level *n* |

示例：

```
// swagger:parameters listBars addBars
type BarSliceParam struct {
    // a BarSlice has bars which are strings
    //
    // min items: 3
    // max items: 10
    // unique: true
    // items.minItems: 4
    // items.maxItems: 9
    // items.items.minItems: 5
    // items.items.maxItems: 8
    // items.items.items.minLength: 3
    // items.items.items.maxLength: 10
    // items.items.items.pattern: \w+
    // collection format: pipe
    // in: query
    // example: [[["bar_000"]]]
    BarSlice [][][]string `json:"bar_slice"`
}
```

##### swagger:operation

**操作**注释将路径链接到方法

语法

```go
swagger:operation [method] [path pattern] [?tag1 tag2 tag3] [operation id]
```

参数

| Field Name   | Type                                                         | Description                                                  |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| tags         | [`string`]                                                   | A list of tags for API documentation control. Tags can be used for logical grouping of operations by resources or any other qualifier. |
| summary      | `string`                                                     | A short summary of what the operation does. For maximum readability in the swagger-ui, this field SHOULD be less than 120 characters. |
| description  | `string`                                                     | A verbose explanation of the operation behavior. [GFM syntax](https://guides.github.com/features/mastering-markdown/#GitHub-flavored-markdown) can be used for rich text representation. |
| externalDocs | [External Documentation Object](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#externalDocumentationObject) | Additional external documentation for this operation.        |
| operationId  | `string`                                                     | Unique string used to identify the operation. The id MUST be unique among all operations described in the API. Tools and libraries MAY use the operationId to uniquely identify an operation, therefore, it is recommended to follow common programming naming conventions. |
| consumes     | [`string`]                                                   | A list of MIME types the operation can consume. This overrides the [`consumes`](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#swaggerConsumes)definition at the Swagger Object. An empty value MAY be used to clear the global definition. Value MUST be as described under [Mime Types](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#mimeTypes). |
| produces     | [`string`]                                                   | A list of MIME types the operation can produce. This overrides the [`produces`](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#swaggerProduces)definition at the Swagger Object. An empty value MAY be used to clear the global definition. Value MUST be as described under [Mime Types](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#mimeTypes). |
| parameters   | [[Parameter Object](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#parameterObject) \| [Reference Object](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#referenceObject)] | A list of parameters that are applicable for this operation. If a parameter is already defined at the [Path Item](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#pathItemParameters), the new definition will override it, but can never remove it. The list MUST NOT include duplicated parameters. A unique parameter is defined by a combination of a [name](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#parameterName) and [location](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#parameterIn). The list can use the [Reference Object](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#referenceObject) to link to parameters that are defined at the [Swagger Object's parameters](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#swaggerParameters). There can be one "body" parameter at most. |
| responses    | [Responses Object](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#responsesObject) | **Required.** The list of possible responses as they are returned from executing this operation. |
| schemes      | [`string`]                                                   | The transfer protocol for the operation. Values MUST be from the list: `"http"`, `"https"`, `"ws"`, `"wss"`. The value overrides the Swagger Object [`schemes`](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#swaggerSchemes)definition. |
| deprecated   | `boolean`                                                    | Declares this operation to be deprecated. Usage of the declared operation should be refrained. Default value is `false`. |
| security     | [[Security Requirement Object](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#securityRequirementObject)] | A declaration of which security schemes are applied for this operation. The list of values describes alternative security schemes that can be used (that is, there is a logical OR between the security requirements). This definition overrides any declared top-level [`security`](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#swaggerSecurity). To remove a top-level security declaration, an empty array can be used. |

示例

```yaml
// ServeAPI serves the API for this record store
func ServeAPI(host, basePath string, schemes []string) (err error) {
    // swagger:operation GET /pets getPet
    //
    // Returns all pets from the system that the user has access to
    //
    // Could be any pet
    //
    // ---
    // produces:
    // - application/json
    // - application/xml
    // - text/xml
    // - text/html
    // parameters:
    // - name: tags
    //   in: query
    //   description: tags to filter by
    //   required: false
    //   type: array
    //   items:
    //     type: string
    //   collectionFormat: csv
    // - name: limit
    //   in: query
    //   description: maximum number of results to return
    //   required: false
    //   type: integer
    //   format: int32
    // responses:
    //   '200':
    //     description: pet response
    //     schema:
    //       type: array
    //       items:
    //         "$ref": "#/definitions/pet"
    //   default:
    //     description: unexpected error
    //     schema:
    //       "$ref": "#/definitions/errorModel"
    mountItem("GET", basePath+"/pets", nil)

    return
}
```

##### swagger:response

读取用swagger:response修饰的结构， 并使用该信息填充相应的标题和模式

swagger：route可以指定状态代码的响应名称，然后匹配的响应将用于swagger定义中的该操作。

语法

```go
swagger:response [?response name]
```

属性

| Annotation            | Description                                                  |
| --------------------- | ------------------------------------------------------------ |
| **In**                | where to find the field                                      |
| **Collection Format** | when a slice the formatter for the collection when serialized on the request |
| **Maximum**           | specifies the maximum a number or integer value can have     |
| **Minimum**           | specifies the minimum a number or integer value can have     |
| **Multiple of**       | specifies a value a number or integer value must be a multiple of |
| **Minimum length**    | the minimum length for a string value                        |
| **Maximum length**    | the maximum length for a string value                        |
| **Pattern**           | a regular expression a string value needs to match           |
| **Minimum items**     | the minimum number of items a slice needs to have            |
| **Maximum items**     | the maximum number of items a slice can have                 |
| **Unique**            | when set to true the slice can only contain unique items     |
| **Example**           | an example value, parsed as the field's type (objects and slices are parsed as JSON) |

对于切片属性，还有要定义的项目。这可能是嵌套集合，用于指示嵌套级别，该值是基于0的索引，因此items.minLength与items.0.minLength相同

| Annotation                 | Format                                                       |
| -------------------------- | ------------------------------------------------------------ |
| **Items.n.Maximum**        | specifies the maximum a number or integer value can have at the level *n* |
| **Items.n.Minimum**        | specifies the minimum a number or integer value can have at the level *n* |
| **Items.n.Multiple of**    | specifies a value a number or integer value must be a multiple of |
| **Items.n.Minimum length** | the minimum length for a string value at the level *n*       |
| **Items.n.Maximum length** | the maximum length for a string value at the level *n*       |
| **Items.n.Pattern**        | a regular expression a string value needs to match at the level *n* |
| **Items.n.Minimum items**  | the minimum number of items a slice needs to have at the level *n* |
| **Items.n.Maximum items**  | the maximum number of items a slice can have at the level *n* |
| **Items.n.Unique**         | when set to true the slice can only contain unique items at the level *n* |

示例

```
// A ValidationError is an error that is used when the required input fails validation.
// swagger:response validationError
type ValidationError struct {
    // The error message
    // in: body
    Body struct {
        // The validation message
        //
        // Required: true
        // Example: Expected type int
        Message string
        // An optional field name to which this validation applies
        FieldName string
    }
}
```

##### swagger:momdel

定义结构数据

语法

```go
swagger:model [?model name]
```

属性

| Annotation         | Description                                                  |
| ------------------ | ------------------------------------------------------------ |
| **Maximum**        | specifies the maximum a number or integer value can have     |
| **Minimum**        | specifies the minimum a number or integer value can have     |
| **Multiple of**    | specifies a value a number or integer value must be a multiple of |
| **Minimum length** | the minimum length for a string value                        |
| **Maximum length** | the maximum length for a string value                        |
| **Pattern**        | a regular expression a string value needs to match           |
| **Minimum items**  | the minimum number of items a slice needs to have            |
| **Maximum items**  | the maximum number of items a slice can have                 |
| **Unique**         | when set to true the slice can only contain unique items     |
| **Required**       | when set to true this value needs to be set on the schema    |
| **Read Only**      | when set to true this value will be marked as read-only and is not required in request bodies |
| **Example**        | an example value, parsed as the field's type (objects and slices are parsed as JSON) |

对于切片属性，还有要定义的项目。这可能是嵌套集合，用于指示嵌套级别，该值是基于0的索引，因此items.minLength与items.0.minLength相同

| Annotation                 | Format                                                       |
| -------------------------- | ------------------------------------------------------------ |
| **Items.n.Maximum**        | specifies the maximum a number or integer value can have at the level *n* |
| **Items.n.Minimum**        | specifies the minimum a number or integer value can have at the level *n* |
| **Items.n.Multiple of**    | specifies a value a number or integer value must be a multiple of |
| **Items.n.Minimum length** | the minimum length for a string value at the level *n*       |
| **Items.n.Maximum length** | the maximum length for a string value at the level *n*       |
| **Items.n.Pattern**        | a regular expression a string value needs to match at the level *n* |
| **Items.n.Minimum items**  | the minimum number of items a slice needs to have at the level *n* |
| **Items.n.Maximum items**  | the maximum number of items a slice can have at the level *n* |
| **Items.n.Unique**         | when set to true the slice can only contain unique items at the level *n* |

示例

```
// User represents the user for this application
//
// A user is the security principal for this application.
// It's also used as one of main axes for reporting.
//
// A user can have friends with whom they can share what they like.
//
// swagger:model
type User struct {
    // the id for this user
    //
    // required: true
    // min: 1
    ID int64 `json:"id"`

    // the name for this user
    // required: true
    // min length: 3
    Name string `json:"name"`

    // the email address for this user
    //
    // required: true
    // example: user@provider.net
    Email strfmt.Email `json:"login"`

    // the friends for this user
    Friends []User `json:"friends"`
}
```

##### swagger:allOf

将嵌入类型标记为allOf的成员

语法

```
swagger:allOf
```

示例

```go
// A SimpleOne is a model with a few simple fields
type SimpleOne struct {
    ID   int64  `json:"id"`
    Name string `json:"name"`
    Age  int32  `json:"age"`
}

// A Something struct is used by other structs
type Something struct {
    DID int64  `json:"did"`
    Cat string `json:"cat"`
}

// Notable is a model in a transitive package.
// it's used for embedding in another model
//
// swagger:model withNotes
type Notable struct {
    Notes string `json:"notes"`

    Extra string `json:"extra"`
}

// An AllOfModel is composed out of embedded structs but it should build
// an allOf property
type AllOfModel struct {
    // swagger:allOf
    SimpleOne
    // swagger:allOf
    mods.Notable

    Something // not annotated with anything, so should be included

    CreatedAt strfmt.DateTime `json:"createdAt"`
}
```

##### swagger:strfmt

**strfmt**标注名称的类型为字符串格式。该名称是必需的，将用作此特定字符串格式的格式名称。

语法

```go
swagger:strfmt [name]
```

字符串格式包含

- uuid, uuid3, uuid4, uuid5
- email
- uri (absolute)
- hostname
- ipv4
- ipv6
- credit card
- isbn, isbn10, isbn13
- social security number
- hexcolor
- rgbcolor
- date
- date-time
- duration
- password
- custom string formats

示例

```go
func init() {
  eml := Email("")
  Default.Add("email", &eml, govalidator.IsEmail)
}

// Email represents the email string format as specified by the json schema spec
//
// swagger:strfmt email
type Email string

// MarshalText turns this instance into text
func (e Email) MarshalText() ([]byte, error) {
    return []byte(string(e)), nil
}

// UnmarshalText hydrates this instance from text
func (e *Email) UnmarshalText(data []byte) error { // validation is performed later on
    *e = Email(string(data))
    return nil
}

func (b *Email) Scan(raw interface{}) error {
    switch v := raw.(type) {
    case []byte:
        *b = Email(string(v))
    case string:
        *b = Email(v)
    default:
        return fmt.Errorf("cannot sql.Scan() strfmt.Email from: %#v", v)
    }

    return nil
}

func (b Email) Value() (driver.Value, error) {
    return driver.Value(string(b)), nil
}
```

##### swagger:discriminated

将嵌入类型标记为allOf的成员并设置x-class值。在接口定义上，对允许*swagger：name的*方法有另一个注释

语法

```go
swagger:allOf org.example.something.TypeName
```

示例

```go
// TeslaCar is a tesla car
//
// swagger:model
type TeslaCar interface {
    // The model of tesla car
    //
    // discriminator: true
    // swagger:name model
    Model() string

    // AutoPilot returns true when it supports autopilot
    // swagger:name autoPilot
    AutoPilot() bool
}

// The ModelS version of the tesla car
//
// swagger:model modelS
type ModelS struct {
    // swagger:allOf com.tesla.models.ModelS
    TeslaCar
    // The edition of this Model S
    Edition string `json:"edition"`
}

// The ModelX version of the tesla car
//
// swagger:model modelX
type ModelX struct {
    // swagger:allOf com.tesla.models.ModelX
    TeslaCar
    // The number of doors on this Model X
    Doors int32 `json:"doors"`
}
```

##### swagger:ignore

将结构标记为从Swagger规范输出中显式忽略

语法

```
swagger:ignore
```

### 在线swagger editor 文档编辑器

#### 启动方式

1. 镜像拉取

   docker pull swaggerapi/swagger-editor

2. 镜像运行

   docker run --rm -p 80:8080 swaggerapi/swagger-editor

