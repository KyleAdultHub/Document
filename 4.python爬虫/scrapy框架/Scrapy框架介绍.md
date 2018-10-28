scrapy框架介绍
scrapy框架的介绍
什么是scrapy框架
Scrapy是一个为了爬取网站数据，提取结构性数据而编写的应用框架，我们只需要实现少量的代码，就能够快速的抓取
Scrapy 使用了 Twisted['twɪstɪd]异步网络框架，可以加快我们的下载速度和数据在模块中间的转载速度。
http://scrapy-chs.readthedocs.io/zh_CN/1.0/intro/overview.html
异步和非阻塞的概念
非阻塞
关注的是程序在等待调用结果（消息，返回值）时的状态，指在不能立刻得到结果之前，该调用不会阻塞当前线程。
异步
如果整个程序没有中介的等待过程，我们就说整个过程是一个异步的过程
多线程爬取数据的流程

scrapy的爬虫流程

scrapy框架各模块功能

创建爬虫项目（命令）
创建项目
命令
scrapy startproject +<项目名字>
创建后项目目录结构

创建爬虫
命令
scrapy genspider  +<爬虫名字> + <允许爬取的域名>          生成普通的spider爬虫
scrapy genspider -t crawl <爬虫名字>  <允许爬取的域名>        生成crawl_Spider爬虫
示例：
scrapy genspider itcast “itcast.cn”
爬虫应用创建的位置

启动爬虫应用
命令
scrapy crawl 爬虫名   执行单个爬虫
-o: 将爬虫的item输出存储到文件中.
quotes.jl : 将每一个item输出为一行的jsonline
quotes.csv: 将每一个item输出为每一行的csv格式文件
quotes.pickle: 存储为二进制文件
Scrapy--scrapy shell
什么是scrapy shell
Scrapy shell是一个交互终端，我们可以在未启动spider的情况下尝试及调试代码，也可以用来测试XPath表达式
查看实例属性/方法
在scrapy shell中查看
进行请求
scrapy shell ‘url’   对url地址进行请求获取响应
查看请求的属性
可使用任意scrapy实例 + TAB查看实例有哪些属性或者方法
response.url：当前响应的url地址
response.request.url：当前响应对应的请求的url地址
response.headers：响应头
response.body：响应体，也就是html代码，默认是byte类型
response.requests.headers：当前响应的请求头
response.meta: 下载延迟，请求深度等信息
response.xpath(***).extract()    验证xpath的正确性
在程序中
print（dir（实例对象））  查看实例对象的所有属性和方法