---
title: Django-项目结构
date: "2016-11-04 22:00:00"
categories:
- Django
- Django五大模块
tags:
- Django
toc: true
typora-root-url: ..\..\..
---


Django项目相关命令操作

- 创建项目专用目录

  - mkdir /data/server
  - cd /data/server/

- 创建DJango项目

  - django-admin startproject 项目名称
  - django-admin startproject itcast

- 项目操作相关命令

  - 命令格式：python manage.py 命令
    - 常用命令（一定要再项目目录中执行）
      - startapp       创建项目应用
      - makemigrations       生成迁移文件

      - migrate       执行迁移

      - shell       shell命令窗口

        - shell和python的区别：
          - 启动python命令解释器两种方式：python manage.py shell 和 python
          - 但是manage.py shell特点是：在启动解释器时候，使用当前Django项目的配置文件，可以直接使用当前项目的所有环境变量等信息。而python命令没有这种功能（当然也可以手动配置虚拟化环境所在的根目录下的.bash_profile文件，添加DJANGO_SETTING_MODULE这个变量）

      - runserver       启动django服务

        - 启动项目的四种方式
          - 1、python manage.py runserver         直接启动 ，浏览器访问地址：127.0.0.1:8000，只能在虚拟机中的浏览器中查看
          - 2、python  manage.py  runserver  端口号       指定端口启动   浏览器访问地址：127.0.0.1:指定端口号，只能在虚拟机中的浏览器中查看

          - 3、python manage.py runserver 服务器地址:监听端口号     指定ip和端口启动      只有启动方式三，才能在虚拟机的外侧可以访问得到，但是如果指定的ip地址是0.0.0.0的话，本地和外部网络都可以访问的到，用于部署服务器。

            - ip地址只能填写本机的ip地址

          - 4、python manage.py runserver >> /dev/null 2>&1 &        脚本启动

      - test  (app名称)     执行应用中的test文件

Django项目简单案例介绍

- 案例分析

  - ![1546102034788](/img/1546102034788.png)

  - 项目需求分析

    - 1、既然是一个Django项目，那么必须部署一个完整的Django项目开发环境，
    - 2、该项目有具体的应用，

      - 所以必须有项目和应用相关的操作
      - 项目名称：book_project
      - 应用名称：book_app

    - 3、三个展示网页的调用，必须有足够的数据逻辑处理(网页跳转必须根据产品设计，自己开发模板文件格式）

      - 首页：index.html
      - 图书页：book.html
      - 人物页：hero.html

  - 项目方案

    - 1、环境部署部分
      - 1.1 python虚拟环境部署
      - 1.2 django环境部署
      - 1.3 开发环境部署

    - 2、项目和应用部分配置

      - 2.1 项目的基本操作
      - 2.2 应用的基本操作
      - 2.3模板目录的配置

    - 3、访问流程部分

      - ​       数据库的配置和数据的获取后面的M实例再进行完善
      - 3.2 web网页模板开发
      - 3.3 数据获取与逻辑处理
      - 3.4 路由配置

    - 4、整体调试

  - 项目方案的实施（VT的设计）

    - 1、环境部署---省略

    - 2、项目应用部分配置

      - 2.1项目基本操作
        - 创建项目
          - cd  /data/server
          - django-admin startproject book_project

        - 创建出来项目初始的结构：

          - ​			![1546102052390](/img/1546102052390.png)				
            - manage.py是项目管理文件，通过它管理项目。
            - itcast与项目同名的目录，存放项目的所有基础文件。
            - \_init_.py是一个空文件，作用是这个目录test1可以被当作包使用。
            - settings.py是项目的整体配置文件。
            - urls.py是项目的URL配置文件。
            - wsgi.py是项目与WSGI兼容的Web服务器入口，详细内容会在布署中讲到。

      - 2.2应用基本操作

        - 创建应用
          - python manage.py startapp book_app
            - 应用起名为book_app

        - 配置应用

          - ![1546102068937](/img/1546102068937.png)
            - 在django项目的配置文件settings.py中有应用注册的一项配置：INSTALLED_APPS,想要创建的应用生效，需要在其中添加应用的名称；

      - 2.3模板目录的创建与配置

        - 创建Templates
          - cd /data/server/book_project/
          - mkdir templates

        - 配置Templates

          - ![1546102094307](/img/1546102094307.png)
            - 在django项目的模板配置信息在settings.py文件中的TEMPLATES部分中。配置一条：'DIRS': [os.path.join(BASE_DIR, 'templates')],
              - BASE_DIR为项目目录，temlates指的是目录中的temlates的文件夹
              - 配置后当访问模板文件的时候会直接在project/temlates目录下进行寻找

    - 3、访问流程部分

      - 3.2首页模板开发
        - 在templates中创建和应用同名的目录
          - cd /data/server/book_project/templates
          - mkdir book_app

            - 避免后期每个应用的前端代码过多，产生混乱

        - 创建首页模板并编辑

          - index.html

      - 3.3创建网页处理逻辑

        - vim /data/server/book_project/book_app/views.py
        - def index(request):

          - \# 接收到请求后，返回的内容就是 book_app/index.html
          - return render(request,'book_app/index.html'）

            - 定义收到请求后返回的页面

      - 3.4路由设计

        - urlpatterns = [
        - url(r'^admin/', include(admin.site.urls)),

        - \# 如果请求的 url 是 127.0.0.1:8000 ，那么就交给 views 文件的 index 函数去处理

        - url(r'^$', index),

        - ]

          - 说明：
            - 设置urls.py文件的urlpatterns配置，增加匹配url的规则磨合匹配后的请求跳转路径，url(r'^admin/', include(admin.site.urls))是匹配规则,admin就是一个关键字，经过url的配置，那么就将该请求交给admin.site.urls这个文件处理

          - 注意：

            - 这里的admin后面必须加/，当url的端口后面不为空的时候，浏览器一般会默认在文件的最后边加上/，如果匹配的时候后边有$，如果没有/将不能匹配搭配内容；

    - 4、测试 --  最基本的思路应完成

      - 后面需要再完善图书和任务的模板，通过视图函数将模板进行渲染

    - 访问流程图（没有M的功能）

    - ![1546102113461](/img/1546102113461.png)

- 设计Model模型类（完成MVT的设计）

- 目标

  - 所以我们需要在模板中完成一个"车票"的功能，让从数据库获取的数据在模板中进行填充（完成上面没有实现的数据获取的功能）。

- 设计过程的方案

  - 1、数据库的配置
  - 2、model文件中数据模型的设计
  - 3、数据的获取方法
  - 4、模板文件填充数据
  - 5、逻辑升级

- 方案的实施

  - 1.数据库配置

    - 安装数据库
      - apt-get install mysql-server -y

    - 配置数据库服务器端的字符编码（数据库传输出来的编码类型）

      - vim /etc/mysql/mysql.conf.d/mysqld.cnf
        - 编辑上面文件，在[mysqld]的下面添加一条配置character-set-server=utf8即可，这样数据库输出的汉字不会是乱码
    - 重启数据库
      - /etc/init.d/mysql restart
    - 验证编码设置是否成功
      - show variables like '%char%ser%';  在数据库中操作
      - variables 数据库的配置属性设置

    - 安装DJANGO和mysql通讯的工具
      - pip install pymysql

    - 编辑Django项目的配置文件settings.py文件

      - ![1546102128242](/img/1546102128242.png)
      - select User,Host from user;   可以在数据库中提前获取主机名和用户名；
      - 在py_django/lib/python3.5/site-packages/django/db/backends/   里面存储了所有的django支持的数据库，django支持7中关系型的数据库

    - 编辑Django项目的__init__.py文件，在里面增加两行配置

      - import pymysql
      - pymysql.install_as_MySQLdb()

        - 如果不加上这两行配置会在进入 python manage.py shell  解释环境的时候报错
        - 将pymasql转义成MySqldb，因为django默认是启动MySqldb与mysql通讯得，而py3又不支持MySqldb所以只能设置一下了

    - 最后创建数据库

      - create database itcastdb charset=utf8;

  - 2.Model模型的设计

  - - model模型分析
      - Model模型的功能
        - 1、设计数据库表
        - 2、进行数据库的操作(生成迁移、执行迁移、向数据库表中插入数据)

    - 数据库表设计（在应用下的model文件里面设计）
      - from django.db import models

      - class BookInfo(models.Model):
      - btitle = models.CharField(max_length=20)#图书名称
      - bpub_date = models.DateField()#发布日期

      - bread = models.IntegerField(default=0)#阅读量

      - bcomment = models.IntegerField(default=0)#评论量

      - isDelete = models.BooleanField(default=False)#逻辑删除

      - 创建的类对应数据库的表，类属性对应数据库表相应的字段
      - 生成的数据库表的名字为:   应用名_类名   （全部会换成小写）
      - 创建出来的表会默认增加一个逐渐字段

  - 生成迁移文件

    - python manage.py makemigrations
      - 这个命令的主要意义就是生成了一个迁移文件，这个生成的文件在哪里？在 book_app/migrations 这个目录里面，文件的名称是 0001_initial.py
  - 执行迁移文件

    - python manage.py migrate
      - 执行完迁移文件，才相当于创建完成了数据库表格
      - 记住，任何修改models文件都需要重新生成迁移文件并执行一次迁移

      - 如果只对字段名称和一些约束进行修改，可能会迁移失败，因为生成的迁移文件和数据库中已经完成迁移的内容重复，这个时候不能再次执行迁移

        - 解决办法：
          - 1.删除掉数据库中的迁移文件相关的表
          - 2.修改生成新的迁移文件的名称

    - 备注：如果执行迁移成功，但是没有成功修改或者创建表单（获取迁移文件对应的sql语句）

      - python manage.py sqlmigrate  应用名  迁移文件名（不包含后缀）
        - 查看迁移文件对应的sql语句在数据库中手动更改

  - 向数据库中插入数据

  - 3.数据获取

    - 修改model返回的数据（由对象改为字符串）
      - class BookInfo(models.Model):
        - btitle = models.CharField(max_length=20)#图书名称
        - bpub_date = models.DateField()#发布日期

        - bread = models.IntegerField(default=0)#阅读量

        - bcomment = models.IntegerField(default=0)#评论量

        - isDelete = models.BooleanField(default=False)#逻辑删除

        - def __str__(self):             新增__str__方法

          - return self.btitle
            - 注意：
            - 在这里里面我们只是重写了一个私有类方法，__str__，他的作用就是将返回的结果转换成字符串，只是将通过过滤器获取的对象结果的显示为定义返回的内容，实质上过滤器获取到的还是对象；

    - 确定获取数据的方法
      - from book_app.models import *     导入models的类对象
      - BookInfo.objects.all()     获取所有图书的对象
        - 返回结果是一个列表，包含所有图书的对象        
        - objects 是Django的 models模型类中定义的一个默认的对象管理器
        - all(),是一个方法，作用就是获取类对象的数据库表中的所有数据

    - 通过重写__str__方法可以修改all()等过滤器方法获得的内容
  - 4.图书模板升级

    - ![1546102148977](/img/1546102148977.png)
    - 网页的title我们使用 Tem_title来代替，格式就是 {{Tem_title}}

    - 获取的所有图书信息，我们使用Tem_booklist来代替，那么我们来遍历这个变量获取具体的图书信息：

    - 网页需要逻辑view传递过来两个参数，模板中的标签会代替view传递过来的参数

    - 通过遍历数据标签，将每个其中每个值都输出来；

    - {{变量名标签}}的作用：

      - 从view视图中获取的数据，通过标签变量名的方式，传递到模板文件中

  - 5.图书逻辑升级

    - ![1546102170866](/img/1546102170866.png)
    - render(请求,'文件',内容)，render可以通过第三个参数，把数据传递给模板文件
    - 将需要遍历的列表 ，和模板的title两个参数放到列表中在rendert函数第三个参数中一起返回
    - 注意：因为Django的默认工作目录是在project目录，而models是在当前目录，所以导入models文件的时候，需要在models前面加上一个. ，代表当前目录，代表book_app .models

  - 整个访问流程梳理

  - ![1546102184966](/img/1546102184966.png)

- 完善人物信息的设计（M部分的完善）

- - 需求：

    - 人物页面中的数据从数据库中获取，人物信息和书进行关联，通过点击书名，可以获取对应包括的人物

  - 设计过程分析：

    - 1、数据模型文件的配置

    - 2、指定数据的获取需要数据

    - 3、模板文件填充数据

    - 4、逻辑升级

    - 5、路由系统跳转的配置

    - 1.数据模型文件的配置

      - 人物的数据库表设计
        - ![1546102197986](/img/1546102197986.png)
      - 需要注意：
        - 1.进行外键的设置，要将一个字段设置为书名的外键，可以根据书对应的id来取出所有对应书id的英雄对象；
        - 2.要重写__str__方法

      - 生成迁移文件

      - 执行迁移

      - 向数据库中添加数据

    - 2.确定数据的获取方法 和需要的数据

      - 需求：需要获取书对应的所有人物
      - 完成方法：

        - 定义一个指定id的图书对象
        - 使用关联格式方法获取指定图书对象所关联的所有人物信息

      - 示例：

        - ![1546102212472](/img/1546102212472.png)

    - 3.人物模板升级

      - 需求：
        - 页面的title要更换
        - 图书的人物要更换，同时要注明该人物是属于哪本图书

        - 增加一个超链接，直接返回上一级，方便我们好再次查看效果

        - 示例：![1546102228980](/img/1546102228980.png

    - 4.人物逻辑升级

      - 需求：
        - 获取图书数据
          - book = BookInfo.objects.get(id=1)
            - id 通过url传递给view

        - 获取人物数据

          - herolist = book.heroinfo_set.all()
          - 通过数据关联来获取任务数据、

        - 指定模板，添加数据
          - context = {'Tem_title':'英雄信息', 'Tem_book':book, 'Tem_herolist':herolist}
          - 将上下文变量存储在字典中，传递给模板文件；

        - 示例：

        - ![1546102249389](/img/1546102249389.png)

- 5.路由系统配置

  - url(r'^renwu(\d+)/$', renwu),
    - 可以将(\d+)中的内容传递给view，逻辑函数可以根据传递过来的数值，来判断是要获取哪本书的对象，就能根据获取书的对象，来获取人物的对象； 

开发流程总结

- 1、Django框架的Models数据模型类文件开发

  - 数据库环境配置
  - 设计model模型
  - 向数据库中添加或者更新数据
  - 定义获取数据的方法，和获取的数据

- 2、Django框架的Templates模板文件开发

  - 定义模板的展示框架
  - 定义需要替换数据位置的标签和所需变量
  - 设置模板中的超链接等

- 3、Django框架的Views数据逻辑处理文件开发

  - 定义需要从url获取的参数
  - 存储查询集，存储在字典中，用来传递给模板文件使用
  - 构建response对象，将参数传递给模板文件，通过方法返回Response报文返回给客户端

- 4、Django的路由配置调整

  - 在正则中定义需要传递给view的数据
  - 定义路由功能，指定需要执行的view函数