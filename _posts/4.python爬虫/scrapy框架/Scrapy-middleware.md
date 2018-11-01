Scrapy-middleware
中间件使用方法
使用方法
在middlewares文件中编写一个Downloader Middlewares的类，定义中间件
在setting中开启Downloader Middlewares，数值越小权限越大
request会先经过权限大的中间件，response会先经过权限小的中间件
​                SPIDER_MIDDLEWARES = {
​                    # 开启middleware，写清楚middleware所在的路径
​                    'login.middlewares.LoginSpiderMiddleware': 543,
​                     }
常用自定义中间件
Downloader Middlewares（下载中间件）的方法
process_request(self, request, spider)
作用
当每个请求通过下载中间件时，该方法被调用,必须返回如下值的其中一个
返回内容
返回None
Scrapy将继续处理该请求，执行其他的中间件的相应方法，直到合适的下载器处理函数（下载处理程序）被调用，该请求被执行（其响应被下载）
返回Request对象
如果其返回 Request 对象，Scrapy则停止调用 process_request方法并重新调度返回的request。当新返回的request被执行后， 相应地中间件链将会根据下载的response被调用。
返回Response对象
Scrapy将不会调用任何其他的 process_request或 process_exception方法，或相应地下载函数； 将返回该response,已安装的中间件的 process_response() 方法则会在每个response返回时被调用。
抛出异常（包括抛出一个IgnoreRequest异常）
停止调用 process_request，下载中间件的 process_exception方法会被调用。如果没有任何一个方法处理该异常， 则request的errback(Request.errback)方法会被调用。如果没有代码处理抛出的异常， 则该异常被忽略且不记录(不同于其他异常那样)。
process_response（self, request, response, spider）  
作用
当响应通过每个下载中间件的时候，调用方法，必须返回如下之一
返回内容
返回Response对象（可以与传入的响应相同，也可以是全新的对象）
该响应将被在链中的其他中间件的process_response方法处理。
返回Request对象
中间件链停止，返回的请求将被重新调度下载。处理类似于process_request()返回请求所做的那样。
产生异常（包括抛出一个IgnoreRequest异常）
停止调用 process_response，下载中间件链的 process_exception方法会被调用，如果没有任何一个方法处理该异常，则调用请求的errback（Request.errback）。如果没有代码处理抛出的异常，则该异常被忽略且不记录（不同于其他异常那样）。
process_exception(self, request，exception, spider)   
作用：
当下载处理器（下载处理程序）、下载中间件的方法抛出异常（包括IgnoreRequest异常）时，Scrapy调用process_exception,返回如下几个参数之一
返回内容：
返回None：Scrapy将会继续处理该异常，调用接着已安装的其他中间件的process_exception方法，直到所有中间件都被调用完毕，则调用默认的异常处理。
返回Request对象，则返回的请求将被重新调用下载。这将停止中间件的process_exception方法执行，就如返回一个响应的那样。
常设置的中间件
给请求添加请求头的内容
​      from .settings import USER_AGENTS 
​       class RandomUserAgent(object):    # 配置下载中间件
​           def process_request(self, request, spider):
​                ug_list = USER_AGENTS
​                user_agent = random.choice(ug_list)
​                # 在requests请求前添加自定义的USER_AGENT
​                request.headers['User-Agent'] = user_agent
给请求添加代理ip
​        class RandomProxy(object):
​               def process_request(self, request, spider):
​                    proxy = random.choice(PROXIES)

                    if proxy['user_passwd'] is None:
                            # 没有代理账户验证的代理使用方式
                            request.meta['proxy'] = "http://" + proxy['ip_port']
                    else:
                            # 对账户密码进行base64编码转换
                            base64_userpasswd = base64.b64encode(proxy['user_passwd'])
                           # 对应到代理服务器的信令格式里
                           request.headers['Proxy-Authorization'] = 'Basic ' + base64_userpasswd
                    request.meta['proxy'] = "https://" + proxy['ip_port']
将跳转（302）等状态码不是200的响应，重新发起请求（只适用于scrapy，不适用与scrapy-redis）
​    class Forbidden302Middleware(object):
​        def process_response(self, request, response, spider):
​            if response.status != 200:
​                print('捕获到一个相应状态码不是200的请求:', request.url, ': ', response.status_code)
​                return request
​            return response
给请求添加cookie的中间件
​    import random
​    class CookiesMiddleware(object):
​        def process_request(self,request,spider):
​            cookie = random.choice(cookie_pool)  # cookie_poll是通过其他模块定期爬取获得的COOKIE集合
​            request.cookies = cooki
在下载中间件中通过selenium实现模拟登录

所有内置下载中间件
关闭内置中间件的方法
示例：'scrapy.downloadermiddlewares.retry.RetryMiddleware': None
​    DOWNLOADER_MIDDLEWARES = {
​        # Engine side
​        'scrapy.downloadermiddlewares.robotstxt.RobotsTxtMiddleware': 100,
​        'scrapy.downloadermiddlewares.httpauth.HttpAuthMiddleware': 300,
​        'scrapy.downloadermiddlewares.downloadtimeout.DownloadTimeoutMiddleware': 350,
​        'scrapy.downloadermiddlewares.defaultheaders.DefaultHeadersMiddleware': 400,
​        'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': 500,
​        'scrapy.downloadermiddlewares.retry.RetryMiddleware': 550,
​        'scrapy.downloadermiddlewares.ajaxcrawl.AjaxCrawlMiddleware': 560,
​        'scrapy.downloadermiddlewares.redirect.MetaRefreshMiddleware': 580,
​        # 对压缩(gzip, deflate)数据的支持
​        'scrapy.downloadermiddlewares.httpcompression.HttpCompressionMiddleware': 590,
​        'scrapy.downloadermiddlewares.redirect.RedirectMiddleware': 600,
​        'scrapy.downloadermiddlewares.cookies.CookiesMiddleware': 700,
​        'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 750,
​        'scrapy.downloadermiddlewares.stats.DownloaderStats': 850,
​        'scrapy.downloadermiddlewares.httpcache.HttpCacheMiddleware': 900,
​        
常用的伪造User-Agent
user_agent_list = [       
​    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.1 (KHTML, like Gecko) Chrome/22.0.1207.1 Safari/537.1",        
​    "Mozilla/5.0 (X11; CrOS i686 2268.111.0) AppleWebKit/536.11 (KHTML, like Gecko) Chrome/20.0.1132.57 Safari/536.11",        
​    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/536.6 (KHTML, like Gecko) Chrome/20.0.1092.0 Safari/536.6",        
​    "Mozilla/5.0 (Windows NT 6.2) AppleWebKit/536.6 (KHTML, like Gecko) Chrome/20.0.1090.0 Safari/536.6",       
​    "Mozilla/5.0 (Windows NT 6.2; WOW64) AppleWebKit/537.1 (KHTML, like Gecko) Chrome/19.77.34.5 Safari/537.1",       
​    "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/536.5 (KHTML, like Gecko) Chrome/19.0.1084.9 Safari/536.5",        
​    "Mozilla/5.0 (Windows NT 6.0) AppleWebKit/536.5 (KHTML, like Gecko) Chrome/19.0.1084.36 Safari/536.5",        
​    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1063.0 Safari/536.3",       
​    "Mozilla/5.0 (Windows NT 5.1) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1063.0 Safari/536.3",        
​    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_8_0) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1063.0 Safari/536.3",        
​    "Mozilla/5.0 (Windows NT 6.2) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1062.0 Safari/536.3",        
​    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1062.0 Safari/536.3",        
​    "Mozilla/5.0 (Windows NT 6.2) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1061.1 Safari/536.3",        
​    "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1061.1 Safari/536.3",        
​    "Mozilla/5.0 (Windows NT 6.1) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1061.1 Safari/536.3",       
​    "Mozilla/5.0 (Windows NT 6.2) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1061.0 Safari/536.3",        
​    "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/535.24 (KHTML, like Gecko) Chrome/19.0.1055.1 Safari/535.24",       
​    "Mozilla/5.0 (Windows NT 6.2; WOW64) AppleWebKit/535.24 (KHTML, like Gecko) Chrome/19.0.1055.1 Safari/535.24",       
​    "Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US) AppleWebKit/531.21.8 (KHTML, like Gecko) Version/4.0.4 Safari/531.21.10",        
​    "Mozilla/5.0 (Windows; U; Windows NT 5.2; en-US) AppleWebKit/533.17.8 (KHTML, like Gecko) Version/5.0.1 Safari/533.17.8",      
​    "Mozilla/5.0 (Windows; U; Windows NT 6.1; en-US) AppleWebKit/533.19.4 (KHTML, like Gecko) Version/5.0.2 Safari/533.18.5",        
​    "Mozilla/5.0 (Windows; U; Windows NT 6.1; en-GB; rv:1.9.1.17) Gecko/20110123 (like Firefox/3.x) SeaMonkey/2.0.12",       
​    "Mozilla/5.0 (Windows NT 5.2; rv:10.0.1) Gecko/20100101 Firefox/10.0.1 SeaMonkey/2.7.1",       
​    "Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10_5_8; en-US) AppleWebKit/532.8 (KHTML, like Gecko) Chrome/4.0.302.2 Safari/532.8",       
​    "Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10_6_4; en-US) AppleWebKit/534.3 (KHTML, like Gecko) Chrome/6.0.464.0 Safari/534.3",      
​    "Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10_6_5; en-US) AppleWebKit/534.13 (KHTML, like Gecko) Chrome/9.0.597.15 Safari/534.13",      
​    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_7_2) AppleWebKit/535.1 (KHTML, like Gecko) Chrome/14.0.835.186 Safari/535.1",      
​    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_6_8) AppleWebKit/535.2 (KHTML, like Gecko) Chrome/15.0.874.54 Safari/535.2",       
​    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_6_8) AppleWebKit/535.7 (KHTML, like Gecko) Chrome/16.0.912.36 Safari/535.7",       
​    "Mozilla/5.0 (Macintosh; U; Mac OS X Mach-O; en-US; rv:2.0a) Gecko/20040614 Firefox/3.0.0 ",       
​    "Mozilla/5.0 (Macintosh; U; PPC Mac OS X 10.5; en-US; rv:1.9.0.3) Gecko/2008092414 Firefox/3.0.3",      
​    "Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10.5; en-US; rv:1.9.1) Gecko/20090624 Firefox/3.5",       
​    "Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10.6; en-US; rv:1.9.2.14) Gecko/20110218 AlexaToolbar/alxf-2.0 Firefox/3.6.14",        
​    "Mozilla/5.0 (Macintosh; U; PPC Mac OS X 10.5; en-US; rv:1.9.2.15) Gecko/20110303 Firefox/3.6.15",     
​    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.6; rv:2.0.1) Gecko/20100101 Firefox/4.0.1"
​    ]
代理ip代码示例
proxy_ip.pyrandomproxymiddleware.py
获取cookie_pool(cookie_list)的模块
get_cookie_list.py