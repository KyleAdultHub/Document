---
title: java 代理、反射
date: "2019-12-24 10:00:00"
categories:
- Java
- Java基础
tags:
- Java
toc: true
typora-root-url: ..\..\..
---

## 反射

### 什么是反射

　在Java里面一个类有两种状态--编译和运行状态，通常我们需要获取这个类的信息都是在编译阶段获得的，也就是直接点出来或者new出来，可是如果需要在类运行的阶段获得Java的类的信息的话，

就需要用到Java的反射。

### 反射的作用

反射被广泛地用于那些需要在运行时检测或修改程序行为的程序中。这是一个相对高级的特性，只有那些语言基础非常扎实的开发者才应该使用它。如果能把这句警示时刻放在心里，那么反射机制就会成为一项强大的技术，可以让应用程序做一些几乎不可能做到的事情。

### 反射的使用

```java
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;

import org.junit.Before;
import org.junit.Test;

public class MyReflect {
	public String className = null;
	@SuppressWarnings("rawtypes")
	public Class personClass = null;
	/**
	 * 反射Person类
	 * @throws Exception 
	 */
	@Before
	public void init() throws Exception {
		className = "cn.java.reflect.Person";
		personClass = Class.forName(className);
	}
	/**
	 *获取某个class文件对象
	 */
	@Test
	public void getClassName() throws Exception {
		System.out.println(personClass);
	}
	/**
	 *获取某个class文件对象的另一种方式
	 */
	@Test
	public void getClassName2() throws Exception {
		System.out.println(Person.class);
	}
	/**
	 *创建一个class文件表示的真实对象，底层会调用空参数的构造方法
	 */
	@Test
	public void getNewInstance() throws Exception {
		System.out.println(personClass.newInstance());
	}
	/**
	 *获取非私有的构造函数
	 */
	@SuppressWarnings({ "rawtypes", "unchecked" })
	@Test
	public void getPublicConstructor() throws Exception {
		Constructor  constructor  = personClass.getConstructor(Long.class,String.class);
		Person person = (Person)constructor.newInstance(100L,"zhangsan");
		System.out.println(person.getId());
		System.out.println(person.getName());
	}
	/**
	 *获得私有的构造函数
	 */
	@SuppressWarnings({ "rawtypes", "unchecked" })
	@Test
	public void getPrivateConstructor() throws Exception {
		Constructor con = personClass.getDeclaredConstructor(String.class);
		con.setAccessible(true);//强制取消Java的权限检测
		Person person2 = (Person)con.newInstance("zhangsan");
		System.out.println(person2.getName());
	}
	/**
	 *获取非私有的成员变量
	 */
	@SuppressWarnings({ "rawtypes", "unchecked" })
	@Test
	public void getNotPrivateField() throws Exception {
		Constructor  constructor  = personClass.getConstructor(Long.class,String.class);
		Object obj = constructor.newInstance(100L,"zhangsan");
		
		Field field = personClass.getField("name");
		field.set(obj, "lisi");
		System.out.println(field.get(obj));
	}
    
    /**
	 *获取私有的成员变量
	 */
	@SuppressWarnings({ "rawtypes", "unchecked" })
	@Test
	public void getPrivateField() throws Exception {
		Constructor  constructor  = personClass.getConstructor(Long.class);
		Object obj = constructor.newInstance(100L);
		
		Field field2 = personClass.getDeclaredField("id");
		field2.setAccessible(true);//强制取消Java的权限检测
		field2.set(obj,10000L);
		System.out.println(field2.get(obj));
	}
	/**
	 *获取非私有的成员函数
	 */
	@SuppressWarnings({ "unchecked" })
	@Test
	public void getNotPrivateMethod() throws Exception {
		System.out.println(personClass.getMethod("toString"));
		
		Object obj = personClass.newInstance();//获取空参的构造函数
		Object object = personClass.getMethod("toString").invoke(obj);
		System.out.println(object);
	}
	/** 
	 *获取私有的成员函数
	 */
	@SuppressWarnings("unchecked")
	@Test
	public void getPrivateMethod() throws Exception {
		Object obj = personClass.newInstance();//获取空参的构造函数
		Method method = personClass.getDeclaredMethod("getSomeThing");
		method.setAccessible(true);
		Object value = method.invoke(obj);
		System.out.println(value);

	}
	/**
	 *
	 */
	@Test
	public void otherMethod() throws Exception {
        //当前加载这个class文件的那个类加载器对象
		System.out.println(personClass.getClassLoader());
		//获取某个类实现的所有接口
		Class[] interfaces = personClass.getInterfaces();
		for (Class class1 : interfaces) {
			System.out.println(class1);
		}
		//反射当前这个类的直接父类
		System.out.println(personClass.getGenericSuperclass());
		/**
		 * getResourceAsStream这个方法可以获取到一个输入流，这个输入流会关联到name所表示的那个文件上。
		 */
		//path 不以’/'开头时默认是从此类所在的包下取资源，以’/'开头则是从ClassPath根下获取。其只是通过path构造一个绝对路径，最终还是由ClassLoader获取资源。
		System.out.println(personClass.getResourceAsStream("/log4j.properties"));
		//默认则是从ClassPath根下获取，path不能以’/'开头，最终是由ClassLoader获取资源。
		System.out.println(personClass.getResourceAsStream("/log4j.properties"));
		
		//判断当前的Class对象表示是否是数组
		System.out.println(personClass.isArray());
		System.out.println(new String[3].getClass().isArray());
		
		//判断当前的Class对象表示是否是枚举类
		System.out.println(personClass.isEnum());
		System.out.println(Class.forName("cn.java.reflect.City").isEnum());
		
		//判断当前的Class对象表示是否是接口
		System.out.println(personClass.isInterface());
		System.out.println(Class.forName("cn.java.reflect.TestInterface").isInterface());
		
		
	}
}
```

## 动态代理

### java 动态代理的设计模式

Java动态代理的优势是实现无侵入式的代码扩展，也就是方法的增强；让你可以在不用修改源码的情况下，增强一些方法；在方法的前后你可以做你任何想做的事情（甚至不去执行这个方法就可以）。

动态代理为其它对象提供一种代理以控制对这个对象的访问控制；在某些情况下，客户不想或者不能直接引用另一个对象，这时候代理对象可以在客户端和目标对象之间起到中介的作用。

### 静态代理

- 静态代理类：由程序员创建或者由第三方工具生成，再进行编译；在程序运行之前，代理类的.class文件已经存在了。
- 静态代理类通常只代理一个类。
- 静态代理事先知道要代理的是什么。

### 动态代理

- 动态代理类：在程序运行时，通过反射机制动态生成。
- 动态代理类通常代理接口下的所有类。
- 动态代理事先不知道要代理的是什么，只有在运行的时候才能确定。
- 动态代理的调用处理程序必须实现InvocationHandler接口，及使用Proxy类中的newProxyInstance方法动态的创建代理类。
- Java动态代理只能代理接口，要代理类需要使用第三方的CLIGB等类库。

### 动态代理实现流程

1. 书写代理类和代理方法，在代理方法中实现代理Proxy.newProxyInstance
2. 代理中需要的参数分别为：被代理的类的类加载器soneObjectclass.getClassLoader()，被代理类的所有实现接口 new Class[] { Interface.class }，句柄方法new InvocationHandler()
3. 在句柄方法中复写invoke方法，invoke方法的输入有3个参数Object proxy（代理类对象）, Method method（被代理类的方法）,Object[] args（被代理类方法的传入参数），在这个方法中，我们可以定制化的开发新的业务。
4. 获取代理类，强转成被代理的接口
5. 最后，我们可以像没被代理一样，调用接口的认可方法，方法被调用后，方法名和参数列表将被传入代理类的invoke方法中，进行新业务的逻辑流程。

### 动态代理代码示例

**接口类**

```java
public interface IBoss {//接口
	int yifu(String size);
}
```

**业务代码**

```java
public class Boss implements IBoss{
	public int yifu(String size){
		System.err.println("天猫小强旗舰店，老板给客户发快递----衣服型号："+size);
		//这件衣服的价钱，从数据库读取
		return 50;
	}
}
```

#### 原调用方法

**调用代码**

```java
public class SaleAction {
		@Test
	public void saleByBossSelf() throws Exception {
		IBoss boss = new Boss();
		System.out.println("老板自营！");
		int money = boss.yifu("xxl");
		System.out.println("衣服成交价：" + money);
	}
}
```

#### 使用代理类进行调用

**代理类**

```java
public static IBoss getProxyBoss(final int discountCoupon) throws Exception {
	Object proxedObj = Proxy.newProxyInstance(Boss.class.getClassLoader(),
			new Class[] { IBoss.class }, new InvocationHandler() {
				public Object invoke(Object proxy, Method method,
						Object[] args) throws Throwable {
						Integer returnValue = (Integer) method.invoke(new Boss(),
								args);// 调用原始对象以后返回的值
						return returnValue - discountCoupon;
				}
			});
	return (IBoss)proxedObj;
}
}
```

**通过代理调用代码**

```java
public class ProxySaleAction {
		@Test
	public void saleByProxy() throws Exception {
		IBoss boss = ProxyBoss.getProxyBoss(20);// 将代理的方法实例化成接口
		System.out.println("代理经营！");
	    int money = boss.yifu("xxl");// 调用接口的方法，实际上调用方式没有变
		System.out.println("衣服成交价：" + money);
	}
}
```

