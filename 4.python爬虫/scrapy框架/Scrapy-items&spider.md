Scrapy-items/spider

Items的配置
定义Item的作用
提前定义需要获取的数据，可以很清楚想要爬取的信息
没有定义的item属性不能被spider使用，键命名错误会报错，可以防止使用错误的键来存储数据
不同的爬虫使用不同的Item来存放不同的数据，在把数据交给pipeline的时候，可以通过isinstance(item, MyspiderItem) 来判断数据是属于哪个item，进行不同的数据(item)处理方式
item实例对象可以通过和字典相同的数据处理方式，来处理item对象的属性，所以我们可以直接将Myspider理解为一个字典
定义Item的方法（在item文件中）
​    class MyspiderItem(scrapy.Item):  # 创建一个item类
​        name = scrapy.Field()    # scrapy.Field( )只是占了一个位，并没有值
​                title = scrapy.Field( )
在Spider中使用item的方法（在spider文件中）
​    from Python import MyspiderItem
​    item = MyspiderItem()    # 创建一个自定义的item实例，item的操作和字典是一样的
​    item['title'] = ###     # 向item对象中插入数据
实现item对象和字典类型数据的转化
dict(item)      将item对象转化为dict类型数据
在将数据进行写入文件中的时候，一定要将其转化为json字符串，所以要先转化为字典
spider爬虫文件Spider类
scrapy_spider类的作用
传递给引擎request对象，或者需要处理的数据，接收引擎传递过来的response对象
配置spider类及类属性（在spider文件中）
​    class PythonSpider(scrapy.Spider)     # 自定义spider类，继承自scrapy.Spider类
​        name = 'python'       # 创建爬虫时候的名字，不可以修改，和爬虫文件名称一致
​        # 创建爬虫时允许爬虫爬取范围，防止爬虫爬取到其他的网站，可以在一个列表中放置多个域名
​               #  从start_url后面爬取的url地址，必须要属于allow_domin域名下的地址
​        allowed_domain = ['python.cn']   
​         # 定义开始爬取的网址，默认是可以被反复请求的
​        start_urls =  'http://www.itcast.cn/channel/teacher.shtml';
配置__init__方法（使用Redis_spider框架爬虫类继承crawlredisSpider的时候要动态获取allow_domin）
​        # 增加__init__()方法，根据从redis中获取的start_url动态设置allow_domin的方法
​    def __init__(self, *args, **kwargs):
​        domain = kwargs.pop('domain', '')
​        self.allowed_domains = filter(None, domain.split(','))
​            # 要继承上方定义的爬虫类的原始的__init__方法
​        super(youyuanSpider, self).__init__(*args, **kwargs)
配置spider类的实例方法实现请求下一页
​    def parse(self,  response):  # 定义数据提取的方法，接收中间件传递过来的response对象（方法名不能改）
​       tr_list = response.xpath("//table[@class='tablelist']/tr")[1:-1]          # xpath分组提取
​              for tr in tr_list:
​                      item = {}
​                      item["title"] = tr.xpath("./td[1]/a/text()").extract_first()
​                      item["position_categary"] = tr.xpath("./td[2]/text()").extract_first()
​                      item["url"] = response.url
​                      yield item        # 将item通过引擎传递给pipeline进行处理

              next_url_temp = response.xpath("//a[@id='next']/@href").extract_first()
              if next_url_temp is not None and next_url_temp != "javascript:;":
                      next_url = "http://hr.tencent.com/"; + next_url_temp
    
           # scrapy.Request构造requests对象给引擎，callback表示response交给哪个函数处理
           yield scrapy.Request(next_page_url, callback=self.parse)  
多个解析函数中如何传递参数示例
​    def parse(self, response):
​        tr_list = response.xpath('//div[@class="greyframe"]/table//table/tr')   # xpath分组提取
​        for i in tr_list:
​            item = MyspiderItem（）  # 实例化item对象
​            ........
​            .......
​                        #   将item参数封装成request对象通过引擎传递给调度器，引擎传递回来的Response对象给get_content函数
​            yield scrapy.Request(item["href"],  callback=self.get_content,  meta{"item":item})   

    def get_content(self, response):
        item = response.meta['item']    # 获取response.meta属性的item对象
        item['text'] = response.xpath("//div[@class='content_text14_2']//text( )").extract( )
        yield  item    # 统一将数据传递给pipeline
start_requests方法
作用：
我们定义的start_urls都是默认交给start_requests处理的，所以如果我们想要在处理在请求之前处理request对象，那么我们可以重写start_requests方法，实现指定其响应的解析函数等功能
示例：
​         import scrapy
​            def start_requests(self):
​               for url in self.start_urls:
​                    yield scrapy.Request(
​                        url,
​                        callback = self.parse
​                )
​            def parse(self, response)
​                self.settings.get('KEY', '')
​                pass
spider中用到的方法/属性：
response对象的属性
response.url：当前响应的url地址
response.request.url：当前响应对应的请求的url地址
response.status: 响应的状态码
response.headers：获取响应头
response.request.headers：当前响应的请求头
response.body：响应体，也就是html代码，默认是bytes类型
response.meta: 解析函数之间传递的数据，字典类型
response.request.headers.getlist('Cookie')    请求的cookie
response.headers.getlist('Set-Cookie')   响应的cookie
response.xpath( )   方法 
xpath( )  response.xpath( )  返回的是一个含有多个匹配结果的selector对象的列表，其具有如下方法
extract( )返回一个包含字符串数据的列表，将列表中所有select对象中的data属性的值提取出来
如果xpath（）提取的数据不存在，返回一个空列表
extract_first  返回列表中的第一个字符串，将列表第一个对象元素的data属性值提取出来
如果xpath（）提取的数据不存在，返回一个None类型数据
scrapy.Request(url, callback, method='GET', headers, body, cookies, meta, dont_filter=False )   get请求方法
url必须传递，为请求的url，将请求的结果作为响应传递给callback解析函数
callback: 指定传入的参数交给那个解析函数去处理
meta: 在不同的解析函数中传递数据， meta默认还会携带部分信息， 比如下载延迟，请求深度等
cookiejar ： 给对应的request一个cookiejar表示，任意的数字，在之后的request也携带该标示，那么将会自动把request对象携带对应的request的获取到的所有cookie内容
dont_filter: 默认url会经过allow_domain过滤，如果dont_filter设置为True，则已经爬过的地址不会被过滤
scrapy默认有url去重的功能,该功能对需要重复请求的url有重要用途
如果请求的页面是实时在变的，在有需要要抓实时的页面内容的时候，需要设置为true
start_urls的请求，dont_filter参数默认为true
method： 请求的方法，当使用POST的时候要传递body参数，在scrapy中body为字符串的形式
cookies： 传递的cookie，注意，在scrapy中cookie不能放在header中进行传递
headers： 使用自定义的header，优先级高于在setting中设置的默认headers
body：请求体，当使用post请求，发送数据的时候使用，并且当发送payload类型参数的时候，一定要使用
并且还要加上请求头 ： 'Content-Type': 'application/json'
注意：
当请求被重定向后，redict中间件会自动进行重定向请求，并返回重定向后的响应
yield 
将数据返回给引擎，并将方法挂起；
原理上，让这个函数成为一个生成器，每次遍历的时候会将每个结果读取到内存中，不会导致没存的占用量瞬间变高
yield只能返回[BaseItem， dict， None， Request ]类型的数据；
yield 的不是Request对象，表示将变量的数据传递给pipeline，接scrapy.Request方法表示将request传递给调度器，进行请求返回给callback函数response
scrapy.FormRequest（）      post表单提交
formdata：携带的post请求的参数，字典类型，只能发送formdata类型的参数
其他参数和scrapy.Request（）方法的参数使用基本一致
scrapy.FormRequest.from_resopnse（）    对响应中的表单自动进行表单提交
response：   解析函数接收到的响应
formdata：  需要填写的表单内容，，只能发送formdata类型的参数
其他参数和scrapy.Request（）方法的参数使用基本一致
Scrapy中的CrawlSpider类
CrawlSpider的作用
是scrapy的一个子类，应用CrawlSpider，我们可以从start_url返回的response中自动提取url地址
scrapy会自动的构造requests请求，发送给引擎，进行请求
还可以指定response是否传递给解析函数，或者是否需要继续提取响应中的url进行请求
使用CrawlSpider的方法
CrawlSpider和普通爬虫的区别
CrawlSpider爬虫类继承自CrawlSpide
在爬虫中定义一个rules对象，scrapy会自动根据rules定义的规则，过滤出符合规则的url，自动进行请求，将响应传递给解析函数，或者继续经过rules过滤符合规律的url进行请求获取响应
不需要自定义parse方法，因为scrapy自动的使用parse方法去请求rules过滤出来的url地址
生成crawlspider的命令
scrapy genspider -t crawl <爬虫名字>  <允许爬取的域名>
配置CrawlSpider爬虫示例
​         form scrapy.linkextractors import LinkExtractor
​         from scrapy.spiders import CrawSpider, Rule

         class CsdnspiderSpider(CrawlSpider):      # CrawlSpider要继承CrawlSpider
                        name = 'csdnspider'
                        allowed_domains = ['suning.com']
                        start_urls = ['http://snbook.suning.com/web/trd-fl/100301/46.htm']
    
                        # 提取url，自动构造请求，把请求交给引擎获取响应
                        rules = (
                              Rule(linkExtractor(allow=r'/web/trd-fl/\d+.htm$'), callback='parse_next_url' follow=True),
                              Rule(linkExtractor(allow=r'/web/prd/\d+.htm$'), callback='parse_item' follow=True),
spiders.Rule常见参数
linkextractor：是 一个Link Extractor对象，用于定义需要提取的链接，交给parse方法；
linkextractor可以接收正则、xpath、css等匹配连接方式，可以查看linkextractor的源码来查看其可以接收的匹配方式
callback：从link_extractor中每获取到连接的时候，参数所指定的值作为回调函数
follow：是一个布尔值，指定了根据改规则从response提取的链接是否需要linkextractor继续跟进（如果callback为None，follow默认设置为True，否则默认为False）
process_links：指定该spider中哪个函数将会被调用，从link_extractor中获取到连接列表时将会调用该函数，该方法主要用来过滤url
process_request：指定该Spider中哪个函数将会被调用，当构造完request列表时都会调用该函数，用来过滤request，必须返回request\None
LinkExtracort常见的参数
allow：满足括号中正则匹配的url会被提取，如果为空，则全部匹配(也可以是xpath匹配)
可以使用其他参数，可以是xpath、css等匹配方法
restrict_xpaths：使用xpath表达式，和allow共同作用过滤链接，及xpath满足范围内的url地址会被提取
deny：满足括号中的正则表达式的url一定不提取（优先级高于allow）
allow_domains：会被提取的连接的domain
deny_domains: 一定不会被提取连接的domains
使用ScrawlSpider注意点
当LinkExtractor提取到一个满足匹配规则的URL地址，后面的规则讲不会匹配该url地址，所以，注意不要将前面url地址匹配规则写的太详细，会被前面的的规则提取完不会经过后面的规则提取；
scrapy会自动的将LinkExtractor提取到的所有url地址给补全协议和域名、端口等，进行请求，不需要我们手动补全； 
scrapy模拟登录
scrapy模拟登录有两种方法
直接携带cookie
找到发送post请求的url地址，带上信息，发送请求
使用scrapy框架自动登录
使用scrapy模拟登录注意点
当需要获取登录后才能获取到数据的时候
一个cookie一般对一个网站访问的时候，服务器可能设置阈值，这时候我们要限制访问的延时
注意有时候服务器会对我们的ip和cookie对应结果做判断，所以我们最好不要频繁的更换ip和cookie的搭配关系
如果想要持续的状态保持可以在setting中设置COOKIES_ENABLE = True
如果想要使用多cookie，多ip对数据进行爬取，可以使用分布式，一个ip搭配一个cookie对数据进行请
scrapy携带cookie模拟登录
应用场景
1、cookie过期时间很长，常见于一些不规范的网站    
2、能在cookie过期之前把搜有的数据拿到    
3、配合其他程序使用，比如其使用selenium把登陆之后的cookie获取到保存到本地，scrapy发送请求之前先读取本地cookie
模拟登录的示例（使用start_request来使请求携带cookie）
​        import scrapy
​                import re
​                class RenrenSpider(scrapy.Spider):
​                        name = 'renren'
​                        allowed_domains = ['renren.com']
​                        start_urls = ['http://www.renren.com/327550029/profile';]

                        def start_requests(self):                # 0.在start_process中对start_urls进行请求
                                cookies_str = "*******"         # 1.获取到cookie,一般情况下我们也可以使用selenium进行获取
                                cookies = {i.split("=")[0]:i.split("=")[-1] for i in cookies_str.split("; ")}
                                yield scrapy.Request(
                                        self.start_urls[0],
                                        callback=self.parse,
                                        cookies=cookies            # 2.直接带上cookie进行页面的请求
                                        )
                        def parse(self, response):          # 接下来的请求scrapy会直接将其带上cookie
                            ret = re.findall(r"你瞅啥",response.body.decode())
                            print(ret)
                            yield scrapy.Request(
                                    "http://www.renren.com/327550029/profile?v=info_timeline";,
                                    callback=self.parse1
scrapy发送post请求模拟登录
使用场景
模拟的POST请求可以获取到请求参数和实际请求的URL地址（可以在之前请求的响应找到，或者可以通过解析js等方式获取到post请求的所需参数）
发送post请求获取cookie的示例
​        import scrapy
​        import re
​        class GithubSpider(scrapy.Spider):
​            name = 'github'
​            allowed_domains = ['github.com']
​            start_urls = ['https://github.com/login';;]
​            def parse(self, response):
​                form_data = {}
​                # 1.获取并构建post请求需要的参数
​                form_data["authenticity_token"] =      # 可以在之前的响应中获取，或js解析
​                   response.xpath("//input[@name='authenticity_token']/@value").extract_first()
​                form_data["commit"] = "Sign in"
​                form_data["utf8"] = "✓"
​                form_data["login"] = "noobpythoner"
​                form_data["password"] = "zhoudawei123"
​            
                # 2.使用scrapy.FormRequest携带参数对登录的url地址进行请求
                yield scrapy.FormRequest(
                   "https://github.com/session",      #post请求的url地址
                    formdata = form_data,
                    callback = self.after_login
                )
            def after_login(self,response):
                ret = re.findall(r"noobpythoner",response.body.decode())
                print(ret)
scrapy自动模拟登录
使用场景
在进行提交post请求的表单中，的form表单具有action属性
可以使用scrapy.FormRequest.from_resopnse方法，会自动帮我们获取form表单的提交地址
我们提供表单中需要填写的内容，可表单提交请求的响应交给哪个函数处理
scrapy自动模拟登录示例
​            import scrapy
​            import re
​            class RenrenSpider(scrapy.Spider):
​                 name = 'renren'
​                 allowed_domains = ['renren.com']
​                 start_urls = ['http://www.renren.com/327550029/profile']

                 def parse(self, response):
                                 # 使用FormRequest.from_resopnse对表单地址进行请求
                 yield scrapy.FormRequest.from_resopnse(
                      # 传入包含post请求表单的响应，表单内容，回调函数
                      response,
                      formdata={'email': 'user_name', 'password': 'password'},
                      callback=self.parse_page
                      )
                 def parse_page(self, response):
                     print(response.url, '*', * 100, response.starus)
                     print('*' * 100)
                     print(re.findall(r'user_name', response.body.decode()))                    
设置cookie允许在函数中传递
在setting中配置cookie的传递
​            COOKIES_ENABLE = True       # 设置允许cookie在不同的解析函数中传递，默认是允许的
​            COOKIES_DEBUG = True        # 可以看带cookie在函数中传递的过程

终端效果如下