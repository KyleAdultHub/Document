Scrapy-pipeline
pipeline的配置
pipeline类的作用
接收引擎传递过来的数据，对数据进行处理
配置pipeline类（在pipelines文件中）
​    class Myspiderpipline(object):
​        def process_item(self, item, spider):   # 实现数据处理的方法，方法名不能修改
​            with open('temp.txt', 'a') as f:
​                json.dump( item, f, ensure_ascii=False, indent=2)
​                        # 将item打印，并将item传递给后面的pipeline，实现数据在管道pipeline之间的传递
​                    return item      
注意：
process_item方法中的spider参数，表示传递参数过来的爬虫对象
spider： 传递数据的爬虫对象，spider.name可以返回爬虫的名称
item：传递过来的数据
setting文件中的设置开启pipeline
​    ITEM_PIPELINE = {
​        'myspider.pipelines.MyspiderPipeline': 300,
​    }
注意：
'muspider.pipelines.MyspiderPipeline'是pipeline文件的路径
300  是pipeline的权重，pipeline的权重越小，其优先级越高，可以理解为距离引擎的远近
如果权重相同的两个pipeline，scrapy会自动的将键值进行排序，根据排序结果定义访问顺序
pipeline的方法
​      form pymongo import MongoClient   
​      class Myspiderpipline(object):
​            # open_spider在爬虫开启请求start_url前的时候只执行一次，一般用于导入数据库
​             def open_spider(self,  spider):  
​                    # 实例化一个mongoclient
​                   con = MongoClient(spider.settings.get('HOST'), spider.settings.get('PORT'))
​                   db = con[spider.settings.get('DB')]
​                   self.collection = db[spider.settings.get('COLLECTON')]
​         
            # close_spider在爬虫关闭的时候只执行 一次，一般用于需要关闭的数据库关闭
            def close_spider(self,  spider):
                    print('关闭pipeline')
                    return item      
            # from_crawler连接到setting.py的配置文件
            def from_crawler(cls, crawler):
                    return  cls(mongo_uri=crawler.settings.get('MGONGO_URI'))
备注：
一个项目会有多个spider，不同的pipeline处理不同的item的内容
一个spider的内容可能要做不同的操作，比如存入不同的数据库中
pipeline向mongoDB中插入数据
在pipeline类的外部定义使用的mongodb集合
​     from pymongo import MongoClient
​     client = MongoClient(host='127.0.0.1', port=27017)    # 实例化mongodb客户端
​     collection = client['myspider']['yangguang']
在pipeline类open_spider中导入mongodb
​    form pymongo import MongoClient   
​    class Myspiderpipline(object):
​        # open_spider在爬虫开启的时候只执行一次
​        def open_spider(self,  spider):  
​            # 实例化一个mongoclient
​            con = MongoClient(spider.settings.get('HOST'), spider.settings.get('PORT'))
​            db = con[spider.settings.get('DB')]
​            self.collection = db[spider.settings.get('COLLECTON')]
向集合中插入数据
​            def process_item(self, item, spider):
​                item['collection_text'] = ****
​                collection.insert(dict(item))   # 向mongodb中插入数据
​                return item
向Mongodb中插入数据
要在setting中配置
MONGO_URI = '127.0.0.1'
MONGO_DATABASE = '数据库名'

pipeline向Mysql中存入数据
数据同步的方式写入Mysql

数据异步的方式写入Mysql

数据自动导出json文件
自定义形式导出

使用scrapy自带exportor导出

将爬取去到的图片地址对应的图片进行存储
setting文件中设置图片存储位置

pipeline中对图片进行下载

ArticleImagePipeline类的方法
get_media_requests(self, item, info)
该方法实现将item字段中的url字段取出来，然后直接生成Request对象，请求对象会加入到对列中，等待执行下载
file_path(self, request, response=None, info=None)
这个方法用来返回保存的文件名
item_completed(self, results, item, info)
当单个item完成下载后的处理方法
results参数就是该item对应的下载结果，它是一个列表形式，列表的每一个元素就是一个元组，其中包含了下载成功或者失败的信息
附件：
Desktop.zip