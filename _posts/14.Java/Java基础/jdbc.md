---
title: jdbc 使用
date: "2019-12-13 14:00:00"
categories:
- Java
- Java基础
tags:
- Java
toc: true
typora-root-url: ..\..\..
---

### JDBC 的介绍

JDBC:java database connectivity

JDBC定义一套标准接口，即访问数据库的通用API，不同的数据库厂商根据各自数据库的特点去实现这些接口，实现接口、类：驱动：由数据库厂商实现

-JDBC是java应用程序和数据库之间的通信桥梁，是java应用程序访问数据库的通道

-JDBC标准主要由一组接口组成，其好处是统一了各种数据库访问方式

-JDBC接口的实现类称为数据库驱动，由各个数据库厂商提供，使用JDBC必须导入这个驱动！一定知道驱动是什么！

### JDBC 进行CURD

#### JDBC使用步骤

1、导入JDBC驱动

再maven配置中导入依赖

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.32</version>
</dependency>
```

1、注册JDBC驱动  

​    参数：“驱动程序类名”  

```java
Class.forname("驱动程序类名")
```

3、获得Connection对象 

   需要三个参数：url，username，password  - 连接到数据库

```java
 connection = DriverManager.getConnection(url, user, password);
```

4、创建Statement（语句）对象  

```java
state = conn.getStatemnent()    方法创建对象

state.execute(ddl)    执行任何SQL，常用于执行DDL、DCL

state.executeUpdate(dml）  执行DML语句，如：insert，update，delete

state.executeQuery(dql)   执行DQL语句，如：select
```

5处理执行结果：

```java
state.execute(ddl)    如果没有异常则成功 boolean

state.executeUpdate(dml）   返回数字，表示更新“行”数量，抛出异常则失败   int

state.executeQuery(dql)   返回ResultSet（结果）对象，代表2维查询结果，  ResultSet

使用for遍历处理，如果查询失败抛出异常
```

6、关闭数据连接！！

```java
conn.close();
```
#### JDBC使用示例

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;

/**
 * @author Evan
 */
public class JDBCTest {
    public static void main(String[] args) throws Exception {
        Connection connection = null;
        PreparedStatement prepareStatement = null;
        ResultSet rs = null;

        try {
            // 加载驱动
            Class.forName("com.mysql.jdbc.Driver");
            // 获取连接
            String url = "jdbc:mysql://127.0.0.1:3306/ssmdemo";
            String user = "root";
            String password = "123456";
            connection = DriverManager.getConnection(url, user, password);
            // 获取statement，preparedStatement
            String sql = "select * from tb_user where id=?";
            prepareStatement = connection.prepareStatement(sql);
            // 设置参数
            prepareStatement.setLong(1, 1l);
            // 执行查询
            rs = prepareStatement.executeQuery();
            // 处理结果集
            while (rs.next()) {
                System.out.println(rs.getString("userName"));
                System.out.println(rs.getString("name"));
                System.out.println(rs.getInt("age"));
                System.out.println(rs.getDate("birthday"));
            }
        } finally {
            // 关闭连接，释放资源
            if (rs != null) {
                rs.close();
            }
            if (prepareStatement != null) {
                prepareStatement.close();
            }
            if (connection != null) {
                connection.close();
            }
        }
    }
}
```

### 解决sql注入问题

#### 解决办法

Statement主要用于执行静态SQL语句，即内容固定不变的SQL语句； 这样容易引入sql注入的攻击问题；

PreparedStatement对象用于执行带参数的预编译执行计划；可以重复使用执行计划，提高DB效率，可以重用执行计划，而且可以执行多次；可以避免注入攻击。

所以使用PreparedStatement 来代替Statement 来解决sql注入的问题

#### 使用示例

```java
package demo;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import org.junit.Test;

public class TestLogin {

    @Test
    public void testLogin() {
        try {
            login("zhangsan' or 'zhangsan", "666");
        } catch (Exception ex) {
            ex.printStackTrace();
        }
    }

    public void login(String username, String password) throws Exception {
        Class.forName("com.mysql.jdbc.Driver");
        String url = "jdbc:mysql://localhost:3306/mybase";
        String usern = "root";
        String pwd = "xuyiqing";
        Connection conn = DriverManager.getConnection(url, usern, pwd);
        String sql = "select * from users where username=? and upassword=?";
        PreparedStatement pstmt = conn.prepareStatement(sql);
        pstmt.setString(1, username);
        pstmt.setString(2, password);
        ResultSet rs = pstmt.executeQuery();
        if (rs.next()) {
            System.out.println("登录成功");
            System.out.println(sql);
        } else {
            System.out.println("账号或密码错误！");
        }
        if (rs != null) {
            rs.close();
        }
        if (pstmt != null) {
            pstmt.close();
        }
        if (conn != null) {
            conn.close();
        }
    }
}
```

### 事务操作

#### 怎样进行事务操作

数据库提供了事务控制功能，支持ACID特性

JDBC提供了API，方便地调用数据库的事务功能，其方法有：

- Connection.getAutoCommit()：获得当前事务的提交方式，默认是true

- Connection.setAutoCommit()：设置事务的提交属性，参数是true：自动提交；false：不自动提交，取消自动提交，后续手动提交

- Connection.Commit()：提交事务

- Connection.rollback()：回滚事务

#### 使用示例

```java
public static void dbTest() {
    Connection conn = null;
    try {
        conn = DBUtils.getConnection();
        conn.setAutoCommit(false);
        //业务处理 执行计划使用步骤
        String sql = "insert into test_wcx values( ?,?,?)";
        // 1、将带参数的SQL发送到数据库创建执行计划
        PreparedStatement ps = conn.prepareStatement(sql);
        //2、替换执行计划中的参数
        ps.setInt(1, 2);
        ps.setString(2, "weicx");
        ps.setString(3, "test");
        //3、执行执行计划，得到结果
        int result = ps.executeUpdate();
        System.out.println(result);
        conn.commit();
    } catch (Exception e) {
 
        e.printStackTrace();
 
        DBUtils.rollback(conn);
 
    } finally {
 
        DBUtils.close(conn);
 
    }
 
}
```

### 数据库连接池

#### 为什么使用连接池

数据库连接池是管理并发访问数据库连接的理想解决方案

解决并发问题;数据库的并发有限,限制连接数，避免数据库崩溃;重用数据库连接

#### 解决办法

连接池是创建和管理连接的缓冲池技术，将连接准备好被任何需要他们的应用使用

DriverManager管理数据库连接适合单线程情况，而在多线程并发情况下，为了能够重用数据库连接，同时控制并发连接总数，保护数据库避免连接过载，一定要使用数据库连接池

#### DBCP的使用

数据库连接池的开源实现非常多，DBCP是其中之一。

使用DBCP步骤：

1. 导入连接池依赖，修改maven 的pom文件

   ```xml
   <dependency>
       <groupId>commons-dbcp</groupId>
       <artifactId>commons-dbcp</artifactId>
       <version>1.4</version>
   </dependency>
   <dependency>
   	<groupId>commons-pool</groupId>
   	<artifactId>commons-pool</artifactId>
   	<version>1.5.4</version>
   </dependency>
   
   ```

2. 创建连接池对象

3. 设置数据库必须的连接参数, 可选的连接参数

4. 从连接池中获得活动的数据库连接

5. 执行sql操作

6. 使用以后关闭数据库连接，这个关闭不再是真的关闭连接，而是将使用过的连接归还给连接池

#### 使用示例

**普通方式连接**

```java
package com.java.dbcp;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import org.apache.commons.dbcp.BasicDataSource;
import com.java.jdbc.util.JDBCUtil;

public class DBCPDemo {
	
	public void testDBCP01() {
		
		Connection connection = null;
		PreparedStatement ps = null;
		
		try {
			//1、构建数据源对象
			BasicDataSource dataSource = new BasicDataSource();
			
			dataSource.setDriverClassName("com.mysql.jdbc.Driver");
			dataSource.setUrl("jdbc:mysql://localhost/php?useSSL=false");
			dataSource.setUsername("root");
			dataSource.setPassword("123456");
			
			//2、得到连接对象
			connection = dataSource.getConnection();
			
			String sql = "insert into stus values(6,?,?)";
			
			ps = connection.prepareStatement(sql);
			
			ps.setString(1, "ling");
			ps.setString(2, "666666");
			
			ps.executeUpdate();
			
		} catch (SQLException e) {
			e.printStackTrace();
		}finally {
			JDBCUtil.release(connection, ps);
		}
		
	}
	public static void main(String[] args) {
		DBCPDemo dbcp = new DBCPDemo();
		dbcp.testDBCP01();
	}
}
```

**通过配置文件的方式连接**

```java
package com.java.dbcp;

import java.io.FileInputStream;
import java.io.InputStream;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.util.Properties;
import javax.sql.DataSource;
import org.apache.commons.dbcp.BasicDataSourceFactory;
import com.java.jdbc.util.JDBCUtil;

public class DBCPDemo02 {
	
	public void dbcpdemo() {
		
		Connection connection = null;
		PreparedStatement ps = null;
		
		try {
			
			BasicDataSourceFactory factory = new BasicDataSourceFactory();
			Properties properties = new Properties();
			InputStream iStream = new FileInputStream("src/dbcpconfig.properties");
			properties.load(iStream);
			DataSource dataSource = factory.createDataSource(properties);
			
			//2、得到连接对象
			connection = dataSource.getConnection();
			
			String sql = "insert into stus values(7,?,?)";
			
			ps = connection.prepareStatement(sql);
			
			ps.setString(1, "wu");
			ps.setString(2, "123456789");
			
			ps.executeUpdate();
			
		} catch (Exception e) {
			e.printStackTrace();
		}finally {
			JDBCUtil.release(connection, ps);
		}
		
	}
	public static void main(String[] args) {
		DBCPDemo02 demo = new DBCPDemo02();
		demo.dbcpdemo();
	}
}
```

**配置文件示例**

```xml
#连接设置
driverClassName=com.mysql.jdbc.Driver
url=jdbc:mysql://localhost/php?useSSL=false
username=root
password=abc123456

#<!-- 初始化连接 -->
initialSize=10

#最大连接数量
maxActive=50

#<!-- 最大空闲连接 -->
maxIdle=20

#<!-- 最小空闲连接 -->
minIdle=5

#<!-- 超时等待时间以毫秒为单位 6000毫秒/1000等于60秒 -->
maxWait=60000


#JDBC驱动建立连接时附带的连接属性属性的格式必须为这样：[属性名=property;] 
#注意："user" 与 "password" 两个属性会被明确地传递，因此这里不需要包含他们。
connectionProperties=useUnicode=true;characterEncoding=gbk

#指定由连接池所创建的连接的自动提交（auto-commit）状态。
defaultAutoCommit=true

#driver default 指定由连接池所创建的连接的事务级别（TransactionIsolation）。
#可用值为下列之一：（详情可见javadoc。）NONE,READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE
defaultTransactionIsolation=READ_UNCOMMITTED
```

