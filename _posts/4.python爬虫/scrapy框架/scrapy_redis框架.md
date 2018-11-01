scrapy_redis框架
scrapy_redis的介绍
scrapy_redis的官方介绍
基于redis的scrapy组件
使用scrapy_redis要安装scrapy_redis组件（pip install scrapy_redis）
scrapy_redis使用场景
增量式抓取、分布式抓取、持久化抓取
scrapy_redis实现的功能及与原生scrapy的区别
request去重
scrapy：可以实现去重，但是scrapy去重不能实现持久化，即当下次如果再次运行程序，会再次存储之前存储过得内容
scrapy_redis：会将请求过的指纹集合保存起来，用来判断重复我请求，即便重新运行程序，已经请求过的请求，也不会进入待请求对列中
爬虫持久化
scrapy：每次运行爬虫都是一次全新的开始
scrapy_redis：会将待请求的序列化后的request对象保存起来，这样每次运行程序都会从待请求列队中读取request对象进行请求，即使终止程序，再次运行，依然会从待请求对列中获取请求对象
分布式爬虫
scrapy：不能实现分布式
scrapy_redis：可以通过redis存储爬取的状态（已爬取，和待爬取），让多台机器上的爬虫程序，共用一个redis服务器，这样数据是共享的，实现了分布式的爬虫
分布式和集群的区别（备注）：
分布式：一个业务拆分成多个子业务，部署在不同的服务器上
集群，同一个业务，部署在多个服务器上
scrapy_redis的爬虫工作流程

Scrapy_redis和scrapy框架使用的区别
克隆scrapy_redis的示例代码
克隆 github上的scrapy-redis源码文件
git clone https://github.com/rolando/scrapy-redis.git
打开example-project的scrapy_redis示例源代码
mv scrapy-redis/example-project    项目目录
example-project项目目录结构如下

redis_scrapy代码上和普通scrapy爬虫的区别
一、spider爬虫文件
第一种Scrapy爬虫（如示例dmoz.py）
区别：
和普通的爬虫文件没有区别
文件主要内容：

第二种RedisSpider爬虫（如示例的myspider_redis.py）
区别
爬虫类继承自RedisSpider
要指定爬虫类的redis_key类属性，指定reids中存储start_url的位置
新增了爬虫类的方法
def make_request_from_data(self, data):   # data 为接收到额redis_key传递过来的url，我们可以自行拼接成想要的url进行请求
search_key = data
url = self.URL_MAIN % (search_key, 1)
result_item = SearchInfoResultItem()
result_item["task_value"] = search_key
return Request(url, meta={"result_item": result_item}, dont_filter=True)
作用
防止分布式爬虫在不同的程序中，都会去请求该start_url，这样会造成请求的重复，响应也会重复的交给解析函数去处理，从redis中取出的start_url第一次被爬虫取出之后便会删除掉，不会造成重复的请求
在持久化和去重的功能上实现了RedisSpider爬虫才实现了分布式
代码示例：

第三种RedisCrawSpider爬虫（如示例的myspider_redis.py）
区别
爬虫类继承自RedisCrawlSpider
比RedisSpider爬虫就多了一个rules过滤功能
作用
在持久化和去重的功能上实现了RedisSpider爬虫才实现了分布式
另外可以实现自动的对响应中的连接进行匹配，进行请求
代码示例

二、setting文件
区别：
设置了三个新的属性
DUPEFILTER_CLASS = "scrapy_redis.dupefilter.RFPDupeFilter"
指定爬虫给request对象去重的方法
SCHEDULER = "scrapy_redis.scheduler.Scheduler"
指定爬虫处理请求列队的方法
SCHEDULER_PERSIST = True
列队中的内容是否持久保存，为False的时候，在关闭程序的时候，清空redis
添加了一个处理item的pipeline
 'scrapy_redis.pipelines.RedisPipeline': 400
用来将items自动保存到reids中，我们可以取消，不用reids来保存
设置了redis的地址
REDIS_URL = 'redis://192.168.207.130: 6379'
不写使用哪个数据库的话，默认使用0数据库
文件主要内容：

三、运行爬虫后，reids数据库的中多出的键（运行程序后）
redis数据库多个三个键
爬虫名：dupefilter   用来存储爬取过的request指纹集合，数据类型为set类型
爬虫名：requests  用来存储已经序列化的待请求对象，数据类型为zset类型
爬虫名：items    用来存储爬取出来的数据，数据类型为list类型
每个键存储的内容：

Scrapy_redis功能实现的方法(Scrapy_redis的源码)
scrapy_redis之redispipeline（实现item数据自动存储redis数据库）
文件位置
/python3.5/site-packages/scrapy_redis/pipelines/RedisPipeline
源码主要功能
实现将item存储在redis中
源码片段

scrapy_redis之RFPDupeFilter（主要实现去重）
文件位置
/python3.5/site-packages/scrapy_redis、dupefilter/RFPDpeFilter
源码主要功能
判断请求是否存在内存中，如果不存在
对请求进行一系列操作后，将其转化为指纹
将指纹存储在内存中
主要实现去重功能
源码片段

加密指纹生成过程
  import hashlib
  fp = hashlib.sha1()
  fp.update(url)
  fp.update(request.method)
  fp.update(request.body or b"")
  return fp.hexdiget()  #sha1结果的16进制的字符串
scrapy_redis之Scheduler（主要实现持久化）
文件位置
/python3.5/site-packages/scrapy_redis/scheduler/Scheduler
代码主要功能
判断取消过滤是否为true
判断url地址是否是第一次看到
如果过滤，且是第一次看到的请求，那么将会进入待请求的列队
主要实现持久化功能
源码片段

scrapy_redis的总结
scrapy_redis与scrapy的区别
本质区别
scrapy_redis_spider相比于之前的scrapy_spider多了持久化和持久化去重的功能
主要是因为scrapy_redis将指纹和请求进行了在redis中的存储,正是redis实现了持久化
代码区别
仅仅是在setting中增加了五行代码
拓展：
scrapy_redis实现了请求的增量式爬虫，还可以实现内容增量式爬虫
可以将dont_filter设置不进行过滤
会将每条数据根据发帖人，发帖时间，帖子更新时间等使用md5算法生成指纹
在进行存储的时候 进行判断是否已经存在，以及是否更新
如果不存在那么便插入
如果存在但是更新时间已经更新，那么便更新
工作场景常用的数据存储模式
优点
可以利用redis的读取快速的特点，进行数据的快速读取
可以实现读取和存储分离，防止锁表的产生
存储数据用mysql或者mongodb
先将爬取到的数据去重后存储到mysql或mongodb中
也可以先存储到数据库中，在对数据库进行数据的去重
redis应用其速度快的特点，用来展示数据
定期从mysql或者mongodb中将去重后的数据读取到redis中
读取数据从redis中读取，其速度远远优于mysql和mongodb