---
title: maven 使用
date: "2019-12-07 14:00:00"
categories:
- Java
- Java基础
tags:
- Java
toc: true
typora-root-url: ..\..\..
---

### 什么是Maven

构建java项目需要用到很多第三方的类库，需要引入大量的jar包。Jar包之间的关系错综复杂，一个Jar包往往又会引用其他Jar包，缺少任何一个Jar包都会导致项目编译失败。

以往开发项目时，程序员往往需要花较多的精力在引用Jar包搭建项目环境上，而这一项工作尤为艰难。

而Maven就是一款帮助程序员构建项目的工具，我们只需要告诉Maven需要哪些Jar 包，它会帮助我们下载所有的Jar，极大提升开发效率。

maven 下载地址: http://maven.apache.org/download.cgi

### Maven 目录结构

![1576225607971](/img/1576225607971.png)

### Maven基本命令

1. -v:   查询Maven版本

   本命令用于检查maven是否安装成功。

   Maven安装完成之后，在命令行输入mvn -v，若出现maven信息，则说明安装成功。

2. compile：编译

   将java源文件编译成class文件

3. test:      测试项目

   执行test目录下的测试用例

4. package:       打包

   将项目打成jar包

5. clean:      删除target文件夹

6. install:         安装

   将当前项目放到Maven的本地仓库中。供其他项目使用

### Maven 仓库

Maven仓库用来存放Maven管理的所有Jar包。分为：本地仓库 和 中央仓库。

- 本地仓库：Maven本地的Jar包仓库。
- 中央仓库： Maven官方提供的远程仓库。

当项目编译时，Maven首先从本地仓库中寻找项目所需的Jar包，若本地仓库没有，再到Maven的中央仓库下载所需Jar包。

### Maven 坐标

在Maven中，坐标是Jar包的唯一标识，Maven通过坐标在仓库中找到项目所需的Jar包。

如下代码中，groupId和artifactId构成了一个Jar包的坐标。

```xml
<dependency>
    <groupId>cn.missbe.web.search</groupId>
    <artifactId>resource-search</artifactId>
    <packaging>jar</packaging>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

- groupId:所需Jar包的项目名
- artifactId:所需Jar包的模块名
- version:所需Jar包的版本号

### 传递依赖和排除依赖

- 传递依赖：如果我们的项目引用了一个Jar包，而该Jar包又引用了其他Jar包，那么在默认情况下项目编译时，Maven会把直接引用和间接引用的Jar包都下载到本地。

- 排除依赖：如果我们只想下载直接引用的Jar包，那么需要在pom.xml中做如下配置：(将需要排除的Jar包的坐标写在中)

  ```xml
  <exclusions>
      <exclusion>
          <groupId>cn.missbe.web.search</groupId>
          <artifactId>resource-search</artifactId>
          <packaging>pom</packaging>
          <version>1.0-SNAPSHOT</version>
      </exclusion>
  </exclusions>
  ```

### 依赖范围 scope

在项目发布过程中，帮助决定哪些构件被包括进来。欲知详情请参考依赖机制。    
- compile ：默认范围，用于编译      
- provided：类似于编译，但支持你期待jdk或者容器提供，类似于classpath      
- runtime: 在执行时需要使用      
- test:    用于test任务时使用      
- system: 需要外在提供相应的元素。通过systemPath来取得      
- systemPath: 仅用于范围为system。提供相应的路径      
- optional:   当项目自身被依赖时，标注依赖是否传递。用于连续依赖时使用

### 依赖冲突

若项目中多个Jar同时引用了相同的Jar时，会产生依赖冲突，但Maven采用了两种避免冲突的策略，因此在Maven中是不存在依赖冲突的。

- 短路优先

  1. 本项目——>A.jar——>B.jar——>X.jar
  2. 本项目——>C.jar——>X.jar

  若本项目引用了A.jar，A.jar又引用了B.jar，B.jar又引用了X.jar，并且C.jar也引用了X.jar。

  在此时，Maven只会引用引用路径最短的Jar。

- 声明优先

  若引用路径长度相同时，再pom.xml中谁先被生命，就使用谁；

### 聚合

1. 什么是聚合

   将多个项目同时构建就称为聚合

2. 如何实现聚合

   再maven项目的pom中作如下配置

   ```xml
   <modules>
       <module>web-connection-pool</module>
       <module>web-java-crawler</module>
   </modules>
   ```

### 继承

1. 什么是继承

   再聚合多个项目时，如果这些被聚合的项目中需要引入相同的jar包，那么可以将这些jar写入父pom中，各个子项目继承该pom即可。

2. 如果实现继承

   - 父pom配置: 将需要继承的jar包的坐标放入标签即可

     ```xml
     <dependencyManagement>
         <dependencies>
               <dependency>
                 <groupId>cn.missbe.web.search</groupId>
                 <artifactId>resource-search</artifactId>
                 <packaging>pom</packaging>
                 <version>1.0-SNAPSHOT</version>
               </dependency> 
         </dependencies>
     </dependencyManagement>
     ```

   - 子pom配置:

     ```xml
     
     <parent>
             <groupId>父pom所在项目的groupId</groupId>
             <artifactId>父pom所在项目的artifactId</artifactId>
             <version>父pom所在项目的版本号</version>
     </parent>
      <parent>
             <artifactId>resource-search</artifactId>
             <groupId>cn.missbe.web.search</groupId>
             <version>1.0-SNAPSHOT</version>
     </parent>
     ```

### pom配置解析

#### 一、基本配置

**1.配置说明**

```xml
<!-- 模型版本。maven2.0必须是这样写，现在是maven2唯一支持的版本 -->  
<modelVersion>4.0.0</modelVersion>    

<!-- 公司或者组织的唯一标志，并且配置时生成的路径也是由此生成， 如com.winner.trade，maven会将该项目打成的jar包放本地路径：/com/winner/trade -->         
<groupId>com.winner.trade</groupId>

<!-- 本项目的唯一ID，一个groupId下面可能多个项目，就是靠artifactId来区分的 -->
<artifactId>...</artifactId>

<!-- 本项目目前所处的版本号 --> 
<version>1.0.0</version>

<!-- 打包的机制，如pom,jar, maven-plugin, ejb, war, ear, rar, par，默认为jar -->
<packaging>jar</packaging>

<!-- 定义本项目的依赖关系 --> 
<dependencies>...</dependencies>

<!-- 父项目的坐标。如果项目中没有规定某个元素的值，那么父项目中的对应值即为项目的默认值。坐标包括group ID，artifact ID和 version或者还有relativePath -->  
<parent>...</parent>

<!-- 定义本项目的依赖关系 --> 
<dependencyManagement>...</dependencyManagement>

<!-- 子模块代码，对modules下的模块进行聚合 --> 
<modules>...</modules>

<!-- 为pom定义一些常量，在pom中的其它地方可以直接引用 使用方式 如 ：${file.encoding} -->  
<properties>...</properties>
```

**2.dependencies 和 dependecyManagement 的区别**

1）dependencies即使在子项目中不写该依赖项，那么子项目仍然会从父项目中继承该依赖项（全部继承）

2）dependencyManagement里只是声明依赖，并不实现引入，因此子项目需要显示的声明需要用的依赖。如果不在子项目中声明依赖，是不会从父项目中继承下来的；只有在子项目中写了该依赖项，并且没有指定具体版本，才会从父项目中继承该项，并且version和scope都读取自父pom;另外如果子项目中指定了版本号，那么会使用子项目中指定的jar版本

#### 二、构建过程的设置

**1.配置示例**

```xml
<build>
    <finalName>platform-b2c</finalName>
    <resources>
        <resource>
            <directory>${basedir}/src/main/resources</directory>
            <filtering>true</filtering>
            <excludes>
                <exclude>**/*.xls</exclude>
            </excludes>
        </resource>
        <!-- xls文件不能filter处理，需区分出来 -->
        <resource>
            <directory>${basedir}/src/main/resources</directory>
            <filtering>false</filtering>
            <includes>
                <include>**/*.xls</include>
            </includes>
        </resource>
    </resources>
    <plugins>到时候详细看</plugins>
    <pluginManagement>到时候详细看</pluginManagement>
</build>
```

**2.配置说明**

```xml
<!-- 产生的构件的文件名，默认值是${artifactId}-${version}。 -->  
<finalName>myPorjectName</finalName>    

<!-- 构建产生的所有文件存放的目录,默认为${basedir}/target，即项目根目录下的target -->  
<directory>${basedir}/target</directory> 

<!--必须跟命令行上的参数相同例如jar:jar，或者与某个阶段（phase）相同例如install、compile等 -->  
<defaultGoal>install</defaultGoal>   

<!--当filtering开关打开时，使用到的过滤器属性文件列表。 -->  
<filters>  
    <filter>../filter.properties</filter>  
</filters>   

<!--项目相关的所有资源路径列表，例如和项目相关的配置文件、属性文件，这些资源被包含在最终的打包文件里。 --> 
<resources>  
    <resource>见3</resource>
</resources>      

<!--使用的插件列表 -->
<plugins>  
    <plugin>...</plugin>
</plugins>       

<!--pluginManagement主要是为了统一管理插件，确保所有子POM使用的插件版本保持一致，类似dependencies和dependencyManagement，只是一个声明，一般用在父pom文件中 -->
<pluginManagement>  
    <plugins>...</plugins>  
</pluginManagement>  
```

**3.resource 说明**

```xml
<resources>  
    <resource>  

        <!--描述了资源的目标路径。该路径相对target/classes目录（例如${project.build.outputDirectory}）。 -->  
        <!--举个例子，如果你想资源在特定的包里(org.apache.maven.messages)，你就必须该元素设置为org/apache/maven/messages。 -->  
        <!--然而，如果你只是想把资源放到源码目录结构里，就不需要该配置。 -->  
        <targetPath>resources</targetPath>  

        <!--是否使用参数值代替参数名。参数值取自properties元素或者文件里配置的属性，文件在filters元素里列出。 -->  
        <filtering>true</filtering>  

        <!--描述存放资源的目录，该路径相对POM路径 -->  
        <directory>src/main/resources</directory>  

        <!--包含的模式列表 -->  
        <includes>  
            <include>**/*.properties</include>  
            <include>**/*.xml</include>  
        </includes>  

        <!--排除的模式列表 如果<include>与<exclude>划定的范围存在冲突，以<exclude>为准 -->  
        <excludes>  
            <exclude>jdbc.properties</exclude>  
        </excludes>  

    </resource>  
</resources>  
```

**4.plugins说明**

```xml
<plugins>  
    <plugin>  
        <groupId></groupId>  
        <artifactId>maven-assembly-plugin</artifactId>  
        <version>2.5.5</version>  

        <!--在构建生命周期中执行一组目标的配置。每个目标可能有不同的配置。 -->  
        <executions>  
            <execution>  

                <!--执行目标的标识符，用于标识构建过程中的目标，或者匹配继承过程中需要合并的执行目标 -->  
                <id>assembly</id>  

                <!--绑定了目标的构建生命周期阶段，如果省略，目标会被绑定到源数据里配置的默认阶段 -->  
                <phase>package</phase>  

                <!--配置的执行目标 -->  
                <goals>  
                    <goal>single</goal>  
                </goals>  

                <!--配置是否被传播到子POM -->  
                <inherited>false</inherited>  

            </execution>  
        </executions>  

        <!--作为DOM对象的配置,配置项因插件而异 -->  
        <configuration>  
            <finalName>${finalName}</finalName>  
            <appendAssemblyId>false</appendAssemblyId>  
            <descriptor>assembly.xml</descriptor>  
        </configuration>  

        <!--是否从该插件下载Maven扩展（例如打包和类型处理器）， -->  
        <!--由于性能原因，只有在真需要下载时，该元素才被设置成true。 -->  
        <extensions>false</extensions>  

        <!--项目引入插件所需要的额外依赖 -->  
        <dependencies>  
            <dependency>...</dependency>  
        </dependencies>  

        <!--任何配置是否被传播到子项目 -->  
        <inherited>true</inherited>  

    </plugin>  
</plugins>  
```

#### 三、偏好设置profile

**1.配置示例**

```xml
<!--在列的项目构建profile，如果被激活，会修改构建处理 -->  
<profiles>  
  
    <!--根据环境参数或命令行参数激活某个构建处理 -->  
    <profile>  

        <!-- 一般是环境的id -->  
        <id />  
        <build />  
        <modules />  
        <repositories />  
        <pluginRepositories />  
        <dependencies />  
        <reporting />  
        <dependencyManagement />  
        <distributionManagement />  
        <!-- 环境的属性设置,一般可以给个属性值，比如dev,test,uat,prod， 从而激发某个环境的配置，比如用在build的resource中 -->  
        <properties />  

        <!--自动触发profile的条件逻辑。Activation是profile的开启钥匙。 -->  
        <activation>  
  
            <!--profile默认是否激活的标识 -->  
            <activeByDefault>false</activeByDefault>  
  
            <!--activation有一个内建的java版本检测，如果检测到jdk版本与期待的一样，profile被激活。 -->  
            <jdk>1.7</jdk>  
  
            <!--当匹配的操作系统属性被检测到，profile被激活。os元素可以定义一些操作系统相关的属性。 -->  
            <os>  
  
                <!--激活profile的操作系统的名字 -->  
                <name>Windows XP</name>  
  
                <!--激活profile的操作系统所属家族(如 'windows') -->  
                <family>Windows</family>  
  
                <!--激活profile的操作系统体系结构 -->  
                <arch>x86</arch>  
  
                <!--激活profile的操作系统版本 -->  
                <version>5.1.2600</version>  
  
            </os>  
  
            <!--如果Maven检测到某一个属性（其值可以在POM中通过${名称}引用），其拥有对应的名称和值，Profile就会被激活。 -->  
            <!-- 如果值字段是空的，那么存在属性名称字段就会激活profile，否则按区分大小写方式匹配属性值字段 -->  
            <property>  
  
                <!--激活profile的属性的名称 -->  
                <name>mavenVersion</name>  
  
                <!--激活profile的属性的值 -->  
                <value>2.0.3</value>  
  
            </property>  
  
            <!--提供一个文件名，通过检测该文件的存在或不存在来激活profile。missing检查文件是否存在，如果不存在则激活profile。 -->  
            <!--另一方面，exists则会检查文件是否存在，如果存在则激活profile。 -->  
            <file>  
  
                <!--如果指定的文件存在，则激活profile。 -->  
                <exists>/usr/local/hudson/hudson-home/jobs/maven-guide-zh-to-production/workspace/</exists>  
  
                <!--如果指定的文件不存在，则激活profile。 -->  
                <missing>/usr/local/hudson/hudson-home/jobs/maven-guide-zh-to-production/workspace/</missing>  
  
            </file>  
  
        </activation>  

    </profile>  

<profiles>    
```

### pom.xml 示例

pom.xml是Maven的核心，你的项目需要什么Jar包就在pom.xml里面配置。当编译项目时Maven读取该文件，并从仓库中下载相应的Jar包。

```xml

<project xmlns="http://maven.apache.org/POM/4.0.0"     
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"     
xsi:schemaLocation="http://maven.apache.org/POM/4.0.0http://maven.apache.org/maven-v4_0_0.xsd">     
    <!--父项目的坐标。如果项目中没有规定某个元素的值，
那么父项目中的对应值即为项目的默认值。 
坐标包括group ID，artifact ID和 version。-->    
	<parent>    
     	<!--被继承的父项目的构件标识符-->    
     	<artifactId/>    
     	<!--被继承的父项目的全球唯一标识符-->    
     	<groupId/>    
     	<!--被继承的父项目的版本-->    
     	<version/>      
	</parent>    
 <!--声明项目描述符遵循哪一个POM模型版本。模型本身的版本很少改变，虽然如此，
但它仍然是必不可少的，这是为了当Maven引入了新的特性或者其他模型变更的时候，
确保稳定性。-->       
	<modelVersion>4.0.0</modelVersion>     
    <!--项目的全球唯一标识符，通常使用全限定的包名区分该项目和其他项目。
并且构建时生成的路径也是由此生成， 如com.mycompany.app生成的相对路径为：
/com/mycompany/app-->     
    <groupId>cn.missbe.web</groupId>     
    <!-- 构件的标识符，它和group ID一起唯一标识一个构件。换句话说，
你不能有两个不同的项目拥有同样的artifact ID和groupID；在某个 
特定的group ID下，artifact ID也必须是唯一的。构件是项目产生的或使用的一个东西，
Maven为项目产生的构件包括：JARs，源码，二进制发布和WARs等。-->     
    <artifactId>search-resources</artifactId>     
    <!--项目产生的构件类型，例如jar、war、ear、pom。插件可以创建
他们自己的构件类型，所以前面列的不是全部构件类型-->     
    <packaging>war</packaging>     
    <!--项目当前版本，格式为:主版本.次版本.增量版本-限定版本号-->     
    <version>1.0-SNAPSHOT</version>     
    <!--项目的名称, Maven产生的文档用-->     
    <name>search-resources</name>     
    <!--项目主页的URL, Maven产生的文档用-->     
    <url>http://www.missbe.cn</url>     
    <!-- 项目的详细描述, Maven 产生的文档用。  当这个元素能够用HTML格式描述时
（例如，CDATA中的文本会被解析器忽略，就可以包含HTML标 签）， 
不鼓励使用纯文本描述。如果你需要修改产生的web站点的索引页面，
你应该修改你自己的索引页文件，而不是调整这里的文档。-->     
    <description>A maven project to study maven.</description>     
    <!--描述了这个项目构建环境中的前提条件。-->    
 <prerequisites>    
  <!--构建该项目或使用该插件所需要的Maven的最低版本-->    
    <maven/>    
 </prerequisites>      
    
  <!--构建项目需要的信息-->    
    <build>    
     <!--该元素设置了项目源码目录，当构建项目的时候，
构建系统会编译目录里的源码。该路径是相对于pom.xml的相对路径。-->    
  <sourceDirectory/>    
  <!--该元素设置了项目脚本源码目录，该目录和源码目录不同：
绝大多数情况下，该目录下的内容 会被拷贝到输出目录(因为脚本是被解释的，而不是被编译的)。-->    
  <scriptSourceDirectory/>    
  <!--该元素设置了项目单元测试使用的源码目录，当测试项目的时候，
构建系统会编译目录里的源码。该路径是相对于pom.xml的相对路径。-->    
  <testSourceDirectory/>    
  <!--被编译过的应用程序class文件存放的目录。-->    
  <outputDirectory/>    
  <!--被编译过的测试class文件存放的目录。-->    
  <testOutputDirectory/>    
  <!--使用来自该项目的一系列构建扩展-->    
  <extensions>    
   <!--描述使用到的构建扩展。-->    
   <extension>    
    <!--构建扩展的groupId-->    
    <groupId/>    
    <!--构建扩展的artifactId-->    
    <artifactId/>    
    <!--构建扩展的版本-->    
    <version/>    
   </extension>    
  </extensions>    
  
  <!--这个元素描述了项目相关的所有资源路径列表，例如和项目相关的属性文件，
这些资源被包含在最终的打包文件里。-->    
  <resources>    
   <!--这个元素描述了项目相关或测试相关的所有资源路径-->    
   <resource>    
    <!-- 描述了资源的目标路径。该路径相对target/classes目录（例如${project.build.outputDirectory}）。举个例 子，如果你想资源在特定的包里(org.apache.maven.messages)，你就必须该元素设置为org/apache/maven /messages。
然而，如果你只是想把资源放到源码目录结构里，就不需要该配置。-->    
    <targetPath/>    
    <!--是否使用参数值代替参数名。参数值取自properties元素或者文件里配置的属性，
文件在filters元素里列出。-->    
    <filtering/>    
    <!--描述存放资源的目录，该路径相对POM路径-->    
    <directory/>    
    <!--包含的模式列表，例如**/*.xml.-->    
    <includes/>    
    <!--排除的模式列表，例如**/*.xml-->    
    <excludes/>    
   </resource>    
  </resources>    
  <!--这个元素描述了单元测试相关的所有资源路径，例如和单元测试相关的属性文件。-->    
  <testResources>    
   <!--这个元素描述了测试相关的所有资源路径，参见build/resources/resource元素的说明-->    
   <testResource>    
    <targetPath/><filtering/><directory/><includes/><excludes/>    
   </testResource>    
  </testResources>    
  <!--构建产生的所有文件存放的目录-->    
  <directory/>    
  <!--产生的构件的文件名，默认值是${artifactId}-${version}。-->    
  <finalName/>    
  <!--当filtering开关打开时，使用到的过滤器属性文件列表-->    
  <filters/>    
  <!--子项目可以引用的默认插件信息。该插件配置项直到被引用时才会被解析或绑定到生命周期。
给定插件的任何本地配置都会覆盖这里的配置-->    
  <pluginManagement>    
   <!--使用的插件列表 。-->    
   <plugins>    
    <!--plugin元素包含描述插件所需要的信息。-->    
    <plugin>    
     <!--插件在仓库里的group ID-->    
     <groupId/>    
     <!--插件在仓库里的artifact ID-->    
     <artifactId/>    
     <!--被使用的插件的版本（或版本范围）-->    
     <version/>    
     <!--是否从该插件下载Maven扩展（例如打包和类型处理器），由于性能原因，
只有在真需要下载时，该元素才被设置成enabled。-->    
     <extensions/>    
     <!--在构建生命周期中执行一组目标的配置。每个目标可能有不同的配置。-->    
     <executions>    
      <!--execution元素包含了插件执行需要的信息-->    
      <execution>    
       <!--执行目标的标识符，用于标识构建过程中的目标，或者匹配继承过程中需要合并的执行目标-->    
       <id/>    
       <!--绑定了目标的构建生命周期阶段，如果省略，目标会被绑定到源数据里配置的默认阶段-->    
       <phase/>    
       <!--配置的执行目标-->    
       <goals/>    
       <!--配置是否被传播到子POM-->    
       <inherited/>    
       <!--作为DOM对象的配置-->    
       <configuration/>    
      </execution>    
     </executions>    
     <!--项目引入插件所需要的额外依赖-->    
     <dependencies>    
      <!--参见dependencies/dependency元素-->    
      <dependency>    
       ......    
      </dependency>    
     </dependencies>         
     <!--任何配置是否被传播到子项目-->    
     <inherited/>    
     <!--作为DOM对象的配置-->    
     <configuration/>    
    </plugin>    
   </plugins>    
  </pluginManagement>    
  <!--使用的插件列表-->    
  <plugins>    
   <!--参见build/pluginManagement/plugins/plugin元素-->    
   <plugin>    
    <groupId/><artifactId/><version/><extensions/>    
    <executions>    
     <execution>    
      <id/><phase/><goals/><inherited/><configuration/>    
     </execution>    
    </executions>    
    <dependencies>    
     <!--参见dependencies/dependency元素-->    
     <dependency>    
      ......    
     </dependency>    
    </dependencies>    
    <goals/><inherited/><configuration/>    
   </plugin>    
  </plugins>    
 </build>    
 <!--模块（有时称作子项目） 被构建成项目的一部分。
列出的每个模块元素是指向该模块的目录的相对路径-->    
 <modules/>    
    <!--发现依赖和扩展的远程仓库列表。-->     
    <repositories>     
     <!--包含需要连接到远程仓库的信息-->    
        <repository>    
         <!--如何处理远程仓库里发布版本的下载-->    
         <releases>    
          <!--true或者false表示该仓库是否为下载某种类型构件（发布版，快照版）开启。 -->    
    <enabled/>    
    <!--该元素指定更新发生的频率。Maven会比较本地POM和远程POM的时间戳。这里的选项是：always（一直），daily（默认，每日），interval：X（这里X是以分钟为单位的时间间隔），或者never（从不）。-->    
    <updatePolicy/>    
    <!--当Maven验证构件校验文件失败时该怎么做：ignore（忽略），fail（失败），或者warn（警告）。-->    
    <checksumPolicy/>    
   </releases>    
   <!-- 如何处理远程仓库里快照版本的下载。有了releases和snapshots这两组配置，
POM就可以在每个单独的仓库中，为每种类型的构件采取不同的 策略。
例如，可能有人会决定只为开发目的开启对快照版本下载的支持。
参见repositories/repository/releases元素 -->    
   <snapshots>    
    <enabled/><updatePolicy/><checksumPolicy/>    
   </snapshots>    
   <!--远程仓库唯一标识符。可以用来匹配在settings.xml文件里配置的远程仓库-->    
   <id>banseon-repository-proxy</id>     
   <!--远程仓库名称-->    
            <name>banseon-repository-proxy</name>     
            <!--远程仓库URL，按protocol://hostname/path形式-->    
            <url>http://192.168.1.169:9999/repository/</url>     
            <!-- 用于定位和排序构件的仓库布局类型-可以是default（默认）或者legacy（遗留）。Maven 2为其仓库提供了一个默认的布局；然 而，Maven 1.x有一种不同的布局。我们可以使用该元素指定布局是default（默认）还是legacy（遗留）。-->    
            <layout>default</layout>               
        </repository>     
    </repositories>    
    <!--发现插件的远程仓库列表，这些插件用于构建和报表-->    
    <pluginRepositories>    
     <!--包含需要连接到远程插件仓库的信息.参见repositories/repository元素-->    
  <pluginRepository>    
   ......    
  </pluginRepository>    
 </pluginRepositories>    
 
    <!--该元素描述了项目相关的所有依赖。 这些依赖组成了项目构建过程中的一个个环节。
它们自动从项目定义的仓库中下载。要获取更多信息，请看项目依赖机制。-->     
    <dependencies>     
        <dependency>    
   <!--依赖的group ID-->    
            <groupId>org.apache.maven</groupId>     
            <!--依赖的artifact ID-->    
            <artifactId>maven-artifact</artifactId>     
            <!--依赖的版本号。 在Maven 2里, 也可以配置成版本号的范围。-->    
            <version>3.8.1</version>     
            <!-- 依赖类型，默认类型是jar。它通常表示依赖的文件的扩展名，但也有例外
。一个类型可以被映射成另外一个扩展名或分类器。类型经常和使用的打包方式对应，
 尽管这也有例外。一些类型的例子：jar，war，ejb-client和test-jar。
如果设置extensions为 true，就可以在 plugin里定义新的类型。所以前面的类型的例子不完整。-->    
            <type>jar</type>    
            <!-- 依赖的分类器。分类器可以区分属于同一个POM，但不同构建方式的构件。
分类器名被附加到文件名的版本号后面。例如，如果你想要构建两个单独的构件成 JAR，
一个使用Java 1.4编译器，另一个使用Java 6编译器，你就可以使用分类器来生成两个单独的JAR构件。-->    
            <classifier></classifier>    
            <!--依赖范围。在项目发布过程中，帮助决定哪些构件被包括进来。欲知详情请参考依赖机制。    
                - compile ：默认范围，用于编译      
                - provided：类似于编译，但支持你期待jdk或者容器提供，类似于classpath      
                - runtime: 在执行时需要使用      
                - test:    用于test任务时使用      
                - system: 需要外在提供相应的元素。通过systemPath来取得      
                - systemPath: 仅用于范围为system。提供相应的路径      
                - optional:   当项目自身被依赖时，标注依赖是否传递。用于连续依赖时使用-->     
            <scope>test</scope>       
            <!--仅供system范围使用。注意，不鼓励使用这个元素，
并且在新的版本中该元素可能被覆盖掉。该元素为依赖规定了文件系统上的路径。
需要绝对路径而不是相对路径。推荐使用属性匹配绝对路径，例如${java.home}。-->    
            <systemPath></systemPath>     
            <!--当计算传递依赖时， 从依赖构件列表里，列出被排除的依赖构件集。
即告诉maven你只依赖指定的项目，不依赖项目的依赖。此元素主要用于解决版本冲突问题-->    
            <exclusions>    
             <exclusion>     
                    <artifactId>spring-core</artifactId>     
                    <groupId>org.springframework</groupId>     
                </exclusion>     
            </exclusions>       
            <!--可选依赖，如果你在项目B中把C依赖声明为可选，你就需要在依赖于B的项目（例如项目A）中显式的引用对C的依赖。可选依赖阻断依赖的传递性。-->     
            <optional>true</optional>    
        </dependency>    
    </dependencies>    
   
  
 <!-- 继承自该项目的所有子项目的默认依赖信息。这部分的依赖信息不会被立即解析,
而是当子项目声明一个依赖（必须描述group ID和 artifact ID信息），
如果group ID和artifact ID以外的一些信息没有描述，
则通过group ID和artifact ID 匹配到这里的依赖，并使用这里的依赖信息。-->    
 <dependencyManagement>    
  <dependencies>    
   <!--参见dependencies/dependency元素-->    
   <dependency>    
    ......    
   </dependency>    
  </dependencies>    
 </dependencyManagement>       
    <!--项目分发信息，在执行mvn deploy后表示要发布的位置。
有了这些信息就可以把网站部署到远程服务器或者把构件部署到远程仓库。-->     
    <distributionManagement>    
        <!--部署项目产生的构件到远程仓库需要的信息-->    
        <repository>    
         <!--是分配给快照一个唯一的版本号（由时间戳和构建流水号）？
还是每次都使用相同的版本号？参见repositories/repository元素-->    
   <uniqueVersion/>    
   <id>banseon-maven2</id>     
   <name>banseon maven2</name>     
            <url>file://${basedir}/target/deploy</url>     
            <layout/>    
  </repository>    
  <!--构件的快照部署到哪里？如果没有配置该元素，默认部署到repository元素配置的仓库，
参见distributionManagement/repository元素-->     
  <snapshotRepository>    
   <uniqueVersion/>    
   <id>banseon-maven2</id>    
            <name>Banseon-maven2 Snapshot Repository</name>    
            <url>scp://svn.baidu.com/banseon:/usr/local/maven-snapshot</url>     
   <layout/>    
  </snapshotRepository>    
  <!--部署项目的网站需要的信息-->     
        <site>    
         <!--部署位置的唯一标识符，用来匹配站点和settings.xml文件里的配置-->     
            <id>banseon-site</id>     
            <!--部署位置的名称-->    
            <name>business api website</name>     
            <!--部署位置的URL，按protocol://hostname/path形式-->    
            <url>     
                scp://svn.baidu.com/banseon:/var/www/localhost/banseon-web      
            </url>     
        </site>    
  <!--项目下载页面的URL。如果没有该元素，用户应该参考主页。
使用该元素的原因是：帮助定位那些不在仓库里的构件（由于license限制）。-->    
  <downloadUrl/>  
  <!-- 给出该构件在远程仓库的状态。不得在本地项目中设置该元素，
因为这是工具自动更新的。有效的值有：none（默认），
converted（仓库管理员从 Maven 1 POM转换过来），partner（直接从伙伴Maven 2仓库同步过来），deployed（从Maven 2实例部 署），verified（被核实时正确的和最终的）。-->    
  <status/>           
    </distributionManagement>    
    <!--以值替代名称，Properties可以在整个POM中使用，也可以作为触发条件（见settings.xml配置文件里activation元素的说明）。格式是<name>value</name>。-->    
    <properties/>    
</project>
```

