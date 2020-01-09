---
title: java JMS基础
date: "2019-12-23 20:00:00"
categories:
- Java
- Java基础
tags:
- Java
toc: true
typora-root-url: ..\..\..
---

### 什么是JMS

​	JMS即Java消息服务（Java Message Service）应用程序接口是一个Java平台中关于面向消息中间件（MOM）的API，用于在两个应用程序之间，或分布式系统中发送消息，进行异步通信。Java消息服务是一个与具体平台无关的API，绝大多数MOM提供商都对JMS提供支持。
​	JMS是一种与厂商无关的 API，用来访问消息收发系统消息。它类似于JDBC(Java Database Connectivity)：这里，JDBC 是可以用来访问许多不同关系数据库的 API，而 JMS 则提供同样与厂商无关的访问方法，以访问消息收发服务。许多厂商都支持 JMS，包括 IBM 的 MQSeries、BEA的 Weblogic JMS service和 Progress 的 SonicMQ，这只是几个例子。 JMS 使您能够通过消息收发服务（有时称为消息中介程序或路由器）从一个 JMS 客户机向另一个 JMS客户机发送消息。消息是 JMS 中的一种类型对象，由两部分组成：报头和消息主体。报头由路由信息以及有关该消息的元数据组成。消息主体则携带着应用程序的数据或有效负载。根据有效负载的类型来划分，可以将消息分为几种类型，它们分别携带：简单文本(TextMessage)、可序列化的对象 (ObjectMessage)、属性集合 (MapMessage)、字节流 (BytesMessage)、原始值流 (StreamMessage)，还有无有效负载的消息 (Message)。

### JMS规范
1. 专业技术规范
   JMS（Java Messaging Service）是Java平台上有关面向消息中间件(MOM)的技术规范，它便于消息系统中的Java应用程序进行消息交换,并且通过提供标准的产生、发送、接收消息的接口简化企业应用的开发，翻译为Java消息服务。

2. 体系架构
   JMS由以下元素组成。
   JMS提供者provider：连接面向消息中间件的，JMS接口的一个实现。提供者可以是Java平台的JMS实现，也可以是非Java平台的面向消息中间件的适配器。
   JMS客户：生产或消费基于消息的Java的应用程序或对象。
   JMS生产者：创建并发送消息的JMS客户。
   JMS消费者：接收消息的JMS客户。
   JMS消息：包括可以在JMS客户之间传递的数据的对象
   JMS队列：一个容纳那些被发送的等待阅读的消息的区域。与队列名字所暗示的意思不同，消息的接受顺序并不一定要与消息的发送顺序相同。一旦一个消息被阅读，该消息将被从队列中移走。
   JMS主题：一种支持发送消息给多个订阅者的机制。

3. Java消息服务应用程序结构支持两种模型
   1、点对点或队列模型
   在点对点或队列模型下，一个生产者向一个特定的队列发布消息，一个消费者从该队列中读取消息。这里，生产者知道消费者的队列，并直接将消息发送到消费者的队列。

   ![1577106078784](/img/1577106078784.png)

   这种模式被概括为：
   只有一个消费者将获得消息
   生产者不需要在接收者消费该消息期间处于运行状态，接收者也同样不需要在消息发送时处于运行状态。
   每一个成功处理的消息都由接收者签收

   2、发布者/订阅者模型
   发布者/订阅者模型支持向一个特定的消息主题发布消息。0或多个订阅者可能对接收来自特定消息主题的消息感兴趣。在这种模型下，发布者和订阅者彼此不知道对方。这种模式好比是匿名公告板。

   ![1577106094072](/img/1577106094072.png)

   这种模式被概括为：
   多个消费者可以获得消息
   在发布者和订阅者之间存在时间依赖性。发布者需要建立一个订阅（subscription），以便客户能够订阅。订阅者必须保持持续的活动状态以接收消息，除非订阅者建立了持久的订阅。在那种情况下，在订阅者未连接时发布的消息将在订阅者重新连接时重新发布。

## acitveMQ部署

#### 1.下载ActiveMQ

去官方网站下载：http://activemq.apache.org/

#### 2.运行ActiveMQ

解压缩apache-activemq-5.5.1-bin.zip，

修改配置文件activeMQ.xml，将0.0.0.0修改为localhost

```xml
<transportConnectors>
       <transportConnector name="openwire" uri="tcp://localhost:61616"/>
       <transportConnector name="ssl"     uri="ssl://localhost:61617"/>
       <transportConnector name="stomp"   uri="stomp://localhost:61613"/>
      <transportConnector uri="http://localhost:8081"/>
       <transportConnector uri="udp://localhost:61618"/>
```

然后双击apache-activemq-5.5.1\bin\activemq.bat运行ActiveMQ程序。

启动ActiveMQ以后，登陆：http://localhost:8161/admin/，创建一个Queue，命名为FirstQueue。

## activeMQ使用代码示例

### 消费者使用方法

**consumerTool**

```java
import javax.jms.Connection;      
import javax.jms.Destination;      
import javax.jms.ExceptionListener;
import javax.jms.JMSException;      
import javax.jms.MessageConsumer;      
import javax.jms.Session;      
import javax.jms.MessageListener;      
import javax.jms.Message;      
import javax.jms.TextMessage;      
     
import org.apache.activemq.ActiveMQConnection;      
import org.apache.activemq.ActiveMQConnectionFactory;      
     
public class ConsumerTool implements MessageListener,ExceptionListener {      
    private String user = ActiveMQConnection.DEFAULT_USER;      
    private String password = ActiveMQConnection.DEFAULT_PASSWORD;      
    private String url =ActiveMQConnection.DEFAULT_BROKER_URL;      
    private String subject = "mytopic";      
    private Destination destination = null;      
    private Connection connection = null;      
    private Session session = null;      
    private MessageConsumer consumer = null;  
    public static Boolean isconnection=false;
    // 初始化      
    private void initialize() throws JMSException, Exception {      
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(      
                user, password, url);      
        connection = connectionFactory.createConnection();      
        session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);      
        destination = session.createTopic(subject);      
        consumer = session.createConsumer(destination);     
    }      
     
    // 消费消息      
    public void consumeMessage() throws JMSException, Exception {      
        initialize();      
        connection.start();
        consumer.setMessageListener(this);    
        connection.setExceptionListener(this);
        isconnection=true;
        System.out.println("Consumer:->Begin listening...");      
        // 开始监听  
        // Message message = consumer.receive();      
    }
    // 关闭连接      
    public void close() throws JMSException {      
        System.out.println("Consumer:->Closing connection");      
        if (consumer != null)      
            consumer.close();      
        if (session != null)      
            session.close();      
        if (connection != null)      
            connection.close();      
    }
    // 消息处理函数      
    public void onMessage(Message message) {      
        try {      
            if (message instanceof TextMessage) {      
                TextMessage txtMsg = (TextMessage) message;      
                String msg = txtMsg.getText();      
                System.out.println("Consumer:->Received: " + msg);      
            } else {      
                System.out.println("Consumer:->Received: " + message);      
            }      
        } catch (JMSException e) {      
            // TODO Auto-generated catch block      
            e.printStackTrace();      
        }      
    }

	public void onException(JMSException arg0) {
		isconnection=false;
	}      
}      
```

**consumerRun**

```java
import javax.jms.JMSException;

public class ConsumerTest implements Runnable {
	static Thread t1 = null;

	/**
	 * @param args
	 * @throws InterruptedException
	 * @throws InterruptedException
	 * @throws JMSException
	 * @throws InterruptedException
	 */
	public static void main(String[] args) throws InterruptedException {

		t1 = new Thread(new ConsumerTest());
		t1.setDaemon(false);
		t1.start();
		/**
		 * 如果发生异常，则重启consumer
		 */
		/*while (true) {
			System.out.println(t1.isAlive());
			if (!t1.isAlive()) {
				t1 = new Thread(new ConsumerTest());
				t1.start();
				System.out.println("重新启动");
			}
			Thread.sleep(5000);
		}*/
		// 延时500毫秒之后停止接受消息
		// Thread.sleep(500);
		// consumer.close();
	}

	public void run() {
		try {
			ConsumerTool consumer = new ConsumerTool();
			consumer.consumeMessage();
			while (ConsumerTool.isconnection) {	
			}
		} catch (Exception e) {
		}

	}
}
```

### 发布者使用

**producerTool**

```java
import javax.jms.Connection;      
import javax.jms.DeliveryMode;      
import javax.jms.Destination;      
import javax.jms.JMSException;      
import javax.jms.MessageProducer;      
import javax.jms.Session;      
import javax.jms.TextMessage;      
     
import org.apache.activemq.ActiveMQConnection;      
import org.apache.activemq.ActiveMQConnectionFactory;      
     
public class ProducerTool {        
    private String user = ActiveMQConnection.DEFAULT_USER;         
    private String password = ActiveMQConnection.DEFAULT_PASSWORD;       
    private String url = ActiveMQConnection.DEFAULT_BROKER_URL;       
    private String subject = "mytopic";      
    private Destination destination = null;      
    private Connection connection = null;      
    private Session session = null;      
    private MessageProducer producer = null;
    // 初始化      
    private void initialize() throws JMSException, Exception {      
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(      
                user, password, url);      
        connection = connectionFactory.createConnection();      
        session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);      
        destination = session.createTopic(subject);      
        producer = session.createProducer(destination);      
        producer.setDeliveryMode(DeliveryMode.NON_PERSISTENT);      
    }
    // 发送消息      
    public void produceMessage(String message) throws JMSException, Exception {      
        initialize();      
        TextMessage msg = session.createTextMessage(message);      
        connection.start();      
        System.out.println("Producer:->Sending message: " + message);      
        producer.send(msg);      
        System.out.println("Producer:->Message sent complete!");      
    }
    // 关闭连接      
    public void close() throws JMSException {      
        System.out.println("Producer:->Closing connection");      
        if (producer != null)      
            producer.close();      
        if (session != null)      
            session.close();      
        if (connection != null)      
            connection.close();      
    }      
}      
```

**producerRun**

```java
import java.util.Random;

import javax.jms.JMSException;      

public class ProducerTest {      
     
    /**    
     * @param args    
     */     
    public static void main(String[] args) throws JMSException, Exception {      
        ProducerTool producer = new ProducerTool(); 
        Random random = new Random();
        for(int i=0;i<20;i++){
        	
        	Thread.sleep(random.nextInt(10)*1000);
        	
        	producer.produceMessage("Hello, world!--"+i);      
        	producer.close();
        }
        
    }      
}      
```

## JMS 实现

要 使用Java消息服务，你必须要有一个JMS提供者，管理会话和队列。既有开源的提供者也有专有的提供者。

开源的提供者包括：

Apache ActiveMQ 

JBoss 社区所研发的 HornetQ

Joram

Coridan的MantaRay

The OpenJMS Group的OpenJMS

专有的提供者包括：

BEA的BEA WebLogic Server JMS

TIBCO Software的EMS

GigaSpaces Technologies的GigaSpaces

Softwired 2006的iBus

IONA Technologies的IONA JMS

SeeBeyond的IQManager（2005年8月被Sun Microsystems并购）

webMethods的JMS+ -

my-channels的Nirvana

Sonic Software的SonicMQ

SwiftMQ的SwiftMQ

IBM的WebSphere MQ

**kafka不满足有序的队列，所以只可作为缓存中间件**