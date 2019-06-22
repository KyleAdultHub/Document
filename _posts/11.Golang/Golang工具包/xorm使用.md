---
title: Go-xorm 使用
date: "2019-5-20 10:45:00"
categories:
- Golang
- Golang工具包
tags:
- Golang
- Orm
toc: true
typora-root-url: ..\..\..
---

# Xorm 使用

#### 创建引擎

- 创建引擎

  driverName, dataSourceName和database/sql接口相同

第一种方法

```go
import (
    _ "github.com/go-sql-driver/mysql"
    "github.com/go-xorm/xorm"
)

var engine *xorm.Engine

func main() {
    var err error
    engine, err = xorm.NewEngine("mysql", "root:123@/test?charset=utf8")
}
```

第二种方法

```go
import (
    _ "github.com/mattn/go-sqlite3"
    "github.com/go-xorm/xorm"
)

var engine *xorm.Engine

func main() {
    var err error
    engine, err = xorm.NewEngine("sqlite3", "./test.db")
}
```

创建Engine组(操作集群)

```go
dataSourceNameSlice := []string{masterDataSourceName, slave1DataSourceName, slave2DataSourceName}
engineGroup, err := xorm.NewEngineGroup(driverName, dataSourceNameSlice)
masterEngine, err := xorm.NewEngine(driverName, masterDataSourceName)
slave1Engine, err := xorm.NewEngine(driverName, slave1DataSourceName)
slave2Engine, err := xorm.NewEngine(driverName, slave2DataSourceName)
engineGroup, err := xorm.NewEngineGroup(masterEngine, []*Engine{slave1Engine, slave2Engine})
```

所有使用 `engine` 都可以简单的用 `engineGroup` 来替换。

#### 同步结构体到数据库表

- 定义一个和表同步的结构体，并且自动同步结构体到数据库

```go
type User struct {
    Id int64
    Name string
    Salt string
    Age int
    Passwd string `xorm:"varchar(200)"`
    Created time.Time `xorm:"created"`
    Updated time.Time `xorm:"updated"`
}

err := engine.Sync2(new(User))
```

#### CURD操作
##### Insert 操作

- `Insert` 插入一条或者多条记录

```go
affected, err := engine.Insert(&user)
// INSERT INTO struct () values ()

affected, err := engine.Insert(&user1, &user2)
// INSERT INTO struct1 () values ()
// INSERT INTO struct2 () values ()

affected, err := engine.Insert(&users)
// INSERT INTO struct () values (),(),()

affected, err := engine.Insert(&user1, &users)
// INSERT INTO struct1 () values ()
// INSERT INTO struct2 () values (),(),()
```

##### Delete操作

- `Delete` 删除记录，需要注意，删除必须至少有一个条件，否则会报错。要清空数据库可以用EmptyTable

```go
affected, err := engine.Where(...).Delete(&user)
// DELETE FROM user Where ...

affected, err := engine.ID(2).Delete(&user)
// DELETE FROM user Where id = ?
```
##### update 更新操作

- `Update` 更新数据，除非使用Cols,AllCols函数指明，默认只更新非空和非0的字段

```go
affected, err := engine.ID(1).Update(&user)
// UPDATE user SET ... Where id = ?

affected, err := engine.Update(&user, &User{Name:name})
// UPDATE user SET ... Where name = ?

var ids = []int64{1, 2, 3}
affected, err := engine.In(ids).Update(&user)
// UPDATE user SET ... Where id IN (?, ?, ?)

// force update indicated columns by Cols
affected, err := engine.ID(1).Cols("age").Update(&User{Name:name, Age: 12})
// UPDATE user SET age = ?, updated=? Where id = ?

// force NOT update indicated columns by Omit
affected, err := engine.ID(1).Omit("name").Update(&User{Name:name, Age: 12})
// UPDATE user SET age = ?, updated=? Where id = ?

affected, err := engine.ID(1).AllCols().Update(&user)
// UPDATE user SET name=?,age=?,salt=?,passwd=?,updated=? Where id = ?
```

##### query操作

- `Query` 最原始的也支持SQL语句查询，返回的结果类型为 []map[string\]byte。`QueryString` 返回 []map\[string\]string, `QueryInterface` 返回 `[]map[string]interface{}`.

```go
results, err := engine.Query("select * from user")
results, err := engine.Where("a = 1").Query()

results, err := engine.QueryString("select * from user")
results, err := engine.Where("a = 1").QueryString()

results, err := engine.QueryInterface("select * from user")
results, err := engine.Where("a = 1").QueryInterface()
```

##### Get 操作

- `Get` 查询单条记录

```
has, err := engine.Get(&user)
// SELECT * FROM user LIMIT 1

has, err := engine.Where("name = ?", name).Desc("id").Get(&user)
// SELECT * FROM user WHERE name = ? ORDER BY id DESC LIMIT 1

var name string
has, err := engine.Table(&user).Where("id = ?", id).Cols("name").Get(&name)
// SELECT name FROM user WHERE id = ?

var id int64
has, err := engine.Table(&user).Where("name = ?", name).Cols("id").Get(&id)
has, err := engine.SQL("select id from user").Get(&id)
// SELECT id FROM user WHERE name = ?

var valuesMap = make(map[string]string)
has, err := engine.Table(&user).Where("id = ?", id).Get(&valuesMap)
// SELECT * FROM user WHERE id = ?

var valuesSlice = make([]interface{}, len(cols))
has, err := engine.Table(&user).Where("id = ?", id).Cols(cols...).Get(&valuesSlice)
// SELECT col1, col2, col3 FROM user WHERE id = ?
```

##### Find操作(查询多条记录)

- `Find` 查询多条记录，当然可以使用Join和extends来组合使用

```go
var users []User
err := engine.Where("name = ?", name).And("age > 10").Limit(10, 0).Find(&users)
// SELECT * FROM user WHERE name = ? AND age > 10 limit 10 offset 0

type Detail struct {
    Id int64
    UserId int64 `xorm:"index"`
}

type UserDetail struct {
    User `xorm:"extends"`
    Detail `xorm:"extends"`
}

var users []UserDetail
err := engine.Table("user").Select("user.*, detail.*")
    Join("INNER", "detail", "detail.user_id = user.id").
    Where("user.name = ?", name).Limit(10, 0).
    Find(&users)
// SELECT user.*, detail.* FROM user INNER JOIN detail WHERE user.name = ? limit 10 offset 0
```

##### 记录Exist 判断

- `Exist` 检测记录是否存在

```
has, err := testEngine.Exist(new(RecordExist))
// SELECT * FROM record_exist LIMIT 1

has, err = testEngine.Exist(&RecordExist{
		Name: "test1",
	})
// SELECT * FROM record_exist WHERE name = ? LIMIT 1

has, err = testEngine.Where("name = ?", "test1").Exist(&RecordExist{})
// SELECT * FROM record_exist WHERE name = ? LIMIT 1

has, err = testEngine.SQL("select * from record_exist where name = ?", "test1").Exist()
// select * from record_exist where name = ?

has, err = testEngine.Table("record_exist").Exist()
// SELECT * FROM record_exist LIMIT 1

has, err = testEngine.Table("record_exist").Where("name = ?", "test1").Exist()
// SELECT * FROM record_exist WHERE name = ? LIMIT 1
```

#### 执行原生sql语句

- `Exec` 执行一个SQL语句

```go
affected, err := engine.Exec("update user set age = ? where name = ?", age, name)
```

#### 数据库遍历

- `Iterate` 和 `Rows` 根据条件遍历数据库，可以有两种方式: Iterate and Rows

```go
err := engine.Iterate(&User{Name:name}, func(idx int, bean interface{}) error {
    user := bean.(*User)
    return nil
})
// SELECT * FROM user

err := engine.BufferSize(100).Iterate(&User{Name:name}, func(idx int, bean interface{}) error {
    user := bean.(*User)
    return nil
})
// SELECT * FROM user Limit 0, 100
// SELECT * FROM user Limit 101, 100

rows, err := engine.Rows(&User{Name:name})
// SELECT * FROM user
defer rows.Close()
bean := new(Struct)
for rows.Next() {
    err = rows.Scan(bean)
}
```

#### 统计操作

- `Count` 获取记录条数

```go
counts, err := engine.Count(&user)
// SELECT count(*) AS total FROM user
```

- `Sum` 求和函数

```go
agesFloat64, err := engine.Sum(&user, "age")
// SELECT sum(age) AS total FROM user

agesInt64, err := engine.SumInt(&user, "age")
// SELECT sum(age) AS total FROM user

sumFloat64Slice, err := engine.Sums(&user, "age", "score")
// SELECT sum(age), sum(score) FROM user

sumInt64Slice, err := engine.SumsInt(&user, "age", "score")
// SELECT sum(age), sum(score) FROM user
```

#### 条件编辑器

- 条件编辑器

```go
err := engine.Where(builder.NotIn("a", 1, 2).And(builder.In("b", "c", "d", "e"))).Find(&users)
// SELECT id, name ... FROM user WHERE a NOT IN (?, ?) AND b IN (?, ?, ?)
```

#### 事务操作

- 在一个Go程中多次操作数据库，但没有事务

```go
session := engine.NewSession()
defer session.Close()

user1 := Userinfo{Username: "xiaoxiao", Departname: "dev", Alias: "lunny", Created: time.Now()}
if _, err := session.Insert(&user1); err != nil {
    return err
}

user2 := Userinfo{Username: "yyy"}
if _, err := session.Where("id = ?", 2).Update(&user2); err != nil {
    return err
}

if _, err := session.Exec("delete from userinfo where username = ?", user2.Username); err != nil {
    return err
}

return nil
```

- 在一个Go程中有事务

```go
session := engine.NewSession()
defer session.Close()

// add Begin() before any action
if err := session.Begin(); err != nil {
    // if returned then will rollback automatically
    return err
}

user1 := Userinfo{Username: "xiaoxiao", Departname: "dev", Alias: "lunny", Created: time.Now()}
if _, err := session.Insert(&user1); err != nil {
    return err
}

user2 := Userinfo{Username: "yyy"}
if _, err := session.Where("id = ?", 2).Update(&user2); err != nil {
    return err
}

if _, err := session.Exec("delete from userinfo where username = ?", user2.Username); err != nil {
    return err
}

// add Commit() after all actions
return session.Commit()
```

- 事物的简写方法

```go
res, err := engine.Transaction(func(session *xorm.Session) (interface{}, error) {
    user1 := Userinfo{Username: "xiaoxiao", Departname: "dev", Alias: "lunny", Created: time.Now()}
    if _, err := session.Insert(&user1); err != nil {
        return nil, err
    }

    user2 := Userinfo{Username: "yyy"}
    if _, err := session.Where("id = ?", 2).Update(&user2); err != nil {
        return nil, err
    }

    if _, err := session.Exec("delete from userinfo where username = ?", user2.Username); err != nil {
        return nil, err
    }
    return nil, nil
})
```

#### 上下文缓存

- 上下文缓存，如果启用，那么针对单个对象的查询将会被缓存到系统中，可以被下一个查询使用。

```go
	sess := engine.NewSession()
	defer sess.Close()

	var context = xorm.NewMemoryContextCache()

	var c2 ContextGetStruct
	has, err := sess.ID(1).ContextCache(context).Get(&c2)
	assert.NoError(t, err)
	assert.True(t, has)
	assert.EqualValues(t, 1, c2.Id)
	assert.EqualValues(t, "1", c2.Name)
	sql, args := sess.LastSQL()
	assert.True(t, len(sql) > 0)
	assert.True(t, len(args) > 0)

	var c3 ContextGetStruct
	has, err = sess.ID(1).ContextCache(context).Get(&c3)
	assert.NoError(t, err)
	assert.True(t, has)
	assert.EqualValues(t, 1, c3.Id)
	assert.EqualValues(t, "1", c3.Name)
	sql, args = sess.LastSQL()
	assert.True(t, len(sql) == 0)
	assert.True(t, len(args) == 0)
```