Scrapy-setting/log信息
配置scrapy的setting文件
为什么需要配置文件
配置文件存放一些公共的变量（比如数据库的地址，账号密码等）方便自己和别人的使用和修改
为了方便识别，一般配置文件中变量的命名都为大写，例如：SQL_HOST = '192.168.0.1'
在pider和pipeline中访问setting内容
在spider中访问setting的内容
​         class Myspider(scrapy.Spider):
​            name = 'myspider'
​            start_urls = ['http://*****';]
​            def parse(self, response):
​                # 可以直接通过访问实例属性的方法，访问setting文件中的内容
​                                # self.settings 返回的是一个字典，里面按照键值对存储所有的配置参数
​                key = self.settings.attributes.get('KEY'，None)               
在pipeline文件中访问setting的内容
​         class Mypipeline(object):
​             def open_spider(self, spider):
​                  # 可以通过传递过来的spider实例来访问setting中的内容
​                  BOT_NAME = spider.setting.get('KEY', None)   
配置文件的常用的配置
​        BOT_NAME = '***'   项目名称
​        SPIDER_MOUDLES = ['ys.spiders']     # 爬虫创建的位置
​        NEWSPIDER_MOUDLE = 'yg.spiders'    # 新建爬虫的位置
​        USER_AGENT ='***'      # 设置主句名称，用来告诉服务器请求的身份，注意，scrapy不能将USER_AGENT配置在        HEADERS中
​       ITEM_PIPLINES   # 管道
​        DOWNLOAD_MIDDLEWARES    # 下载中间件      
​       OBEY_ROBOTFILES = False    # 是否遵守robots协议
​        SPIDER_MIDDLEWARES    # 爬虫中间件
​        DEAFAULT_REQUEAT_HEADERS = { ***}   # 配置scrapy 默认的请求头，不能包含USER-AGRNT和cookies、refer
​        #  使用增量式，或分布式爬虫需要配置
​        DUPEFILTER_CLASS = "scrapy_redis.dupefilter.RFPDupeFilter"
​        SCHEDULER = "scrapy_redis.scheduler.Scheduler"
​        SCHEDULER_PERSIST = True
​        ITEM_PIPELINES = {
​            'scrapy_redis.pipelines.RedisPipeline': 400
​            }
​        REDIS_URL = 'redis://127.0.0.1: 6379'
​        
       RETRY_HTTP_CODES = []    # 设置会retry的状态码响应
        HTTPERROR_ALLOWED_CODES = [302,]    # 设置来指定spider能处理的response返回值
        LOG_LEVEL = "WARNING"     # 设置log提示的等级，log的从高到低的级别分别为：error、warning、info、debug
        RETRY_ENABLED = False     开启请求失败重新请求，默认是开启的
        RETRY_TIMES = 3        设置重新请求的次数
        DOWNLOAD_TIMEOUT = 6     设置请求最大等待时间
        COOKIES_ENABLE = True        #  是否启用cookie中间件，如果关闭是不会向服务器发送cookie的
        COOKIES_DEBUG = True        # 可以看带cookie在函数中传递的过程  
        DOWNLOAD_DELAY = 3   # 请求延时
        CONCURRENT_REQUESTS = 16   # 并发数量，默认值为16
        CONCURRENT_REQUESTS_PRE_DOMAIN = 16   # 每个域名最大并发
        CONCURRENT_REQUESTS_PR_IP = 16    # 每个ip最大的并发
         AUTOTHROTTLE_ENABLED = True    # 动态调整下载延时
        CONCURRENT_REQUESTS_PRE_DOMAIN = 1  # 设置允许对同一域名发起请求的并发数量限制
        JOBDIR = '路径'     # 配置记录爬虫状态目录的文件，使爬虫中途停止再启动的时候会接着上一次的状态继续爬取
log的配置与scrapy的debug信息
scrapy-log相关的配置
log的作用
为了让我们自己希望输出到终端的内容能容易看一些
配置log的显示等级
在setting中设置log显示的级别
​            LOG_LEVEL = "WARNING"        #  在setting文件中添加一行
log的从高到低的级别分别为：error、warning、info、debug
自定义log日志的方法
在想要显示log的位置添加log的输出
第一种打印方式：（不带log产生的位置）

第二种打印方式：（带log产生的具体文件）

在普通的py文件中配置logger日志的方法
作用
我们配置完了logger的输出文件后，我们可以在任何其他的程序后，可以导入logger模块，帮助我们来可以看到错误产生的位置和时间等信息
我们还可以将log存储的文件命名为每日的日期，这样方便我们分别存储我们每日的bug
示例：
​                    import logging
​                           # 配置logger的输出格式，和输出内容
​                    logging.basicConfig(
​                                    level=logging.INFO,
​                                    format='%(levelname)s : %(filename)s '
​                                           '[%(lineno)d] : %(message)s'
​                                           ' - %(asctime)s', datefmt='[%d-%b-%Y %H:%M:%S]'
​                                            filename='./logdebug.log')
​                   
                    if __name__ == '__main__':
                        logger = logging.getLogger(__name__)
                        log_message = 'error'
                        logger.warning(log_message）
输出的格式

详细示例

常见的scrapy的debug信息