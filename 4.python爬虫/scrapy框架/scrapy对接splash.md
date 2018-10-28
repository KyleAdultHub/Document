scrapy对接splash
Scrapy 对接 Splash
环境准备
首先在这之前请确保已经正确安装好了Splash并正常运行，同时安装好了ScrapySplash库
Scrapy-Splash文档
https://github.com/scrapy-plugins/scrapy-splash
Scrapy-splash的配置
新建项目和spider
scrapy startproject scrapysplashtest     新建项目
scrapy genspider taobao www.taobao.com     新建spider
修改setting.py文件, 添加splash配置
SPLASH_URL = 'http://localhost:8050'         添加splash服务的地址
DUPEFILTER_CLASS = 'scrapy_splash.SplashAwareDupeFilter'       配置去重类
HTTPCACHE_STORAGE = 'scrapy_splash.SplashAwareFSCacheStorage'     还需要配置一个Cache存储HTTPCACHE_STORAGE
添加splash中间件
DOWNLOADER_MIDDLEWARES = {
'scrapy_splash.SplashCookiesMiddleware': 723,
'scrapy_splash.SplashMiddleware': 725,
'scrapy.downloadermiddlewares.httpcompression.HttpCompressionMiddleware': 810,
}
SPIDER_MIDDLEWARES = {
'scrapy_splash.SplashDeduplicateArgsMiddleware': 100,
}
SplashRequest请求的使用
使用splash请求的说明
配置完成之后我们就可以利用Splash来抓取页面了，例如我们可以直接生成一个SplashRequest对象并传递相应的参数，Scrapy会将此请求转发给Splash
Splash对页面进行渲染加载，然后再将渲染结果传递回来，此时Response的内容就是渲染完成的页面结果了，最后交给Spider解析即可。
使用请求的方法
第一种方法
通过SplashRequest发送请求

第二种方法
scrapy.Request对象发送请求给splash服务器，只需将配置属性给meta参数即可

通过lua源码控制splash服务的示例
我们把Lua脚本定义成长字符串，通过SplashRequest的args来传递参数，同时接口修改为execute，另外args参数里还有一个lua_source字段用于指定Lua脚本内容，这样我们就成功构造了一个SplashRequest，对接Splash的工作就完成了。
​                                from scrapy import Spider
​                                from urllib.parse import quote
​                                from scrapysplashtest.items import ProductItem
​                                from scrapy_splash import SplashRequest

                                script = """
                                function main(splash, args)
                                splash.images_enabled = false
                                assert(splash:go(args.url))
                                assert(splash:wait(args.wait))
                                js = string.format("document.querySelector('#mainsrp-pager div.form > input').value=%d;document.querySelector('#mainsrp-pager div.form > span.btn.J_Submit').click()", args.page)
                                splash:evaljs(js)
                                assert(splash:wait(args.wait))
                                return splash:html()
                                end
                                """
    
                                class TaobaoSpider(Spider):
                                name = 'taobao'
                                allowed_domains = ['www.taobao.com']
                                base_url = 'https://s.taobao.com/search?q='
    
                                def start_requests(self):
                                for keyword in self.settings.get('KEYWORDS'):
                                for page in range(1, self.settings.get('MAX_PAGE') + 1):
                                url = self.base_url + quote(keyword)
                                yield SplashRequest(
                                        url,
                                        callback=self.parse,
                                        endpoint='execute',
                                        args={'lua_source': script, 'page': page, 'wait': 7})
使用scrapy-splash比使用selenium的优点
由于Splash和Scrapy都支持异步处理，我们可以看到同时会有多个抓取成功的结果，而Selenium的对接过程中每个页面渲染下载过程是在Downloader Middleware里面完成的，所以整个过程是堵塞式的，Scrapy会等待这个过程完成后再继续处理和调度其他请求，影响了爬取效率。
使用Splash，是在中间件中将请求和渲染等工作交给了splash服务器, 各请求之间是异步的，因此使用Splash爬取效率上比Selenium高出很多。
因此，在Scrapy中要处理JavaScript渲染的页面建议使用Splash，这样不会破坏Scrapy中的异步处理过程，会大大提高爬取效率，而且Splash的安装和配置比较简单，通过API调用的方式也实现了模块分离，大规模爬取时部署起来也更加方便。