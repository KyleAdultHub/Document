Request模块的介绍
request 库的特点
requests的底层实现就是urllib
requests在python2 和python3中通用，方法完全一样
requests简单易用
requests能够自动帮助我们解压(gzip压缩的等)网页内容
requests的作用
发送网络请求，返回响应数据
中文文档 API：http://docs.python-requests.org/zh_CN/latest/user/quickstart.html
request/response的常用方法和属性
requests.get()    get请求
使用方法：
response = requests.get(url, headers=headers， params=params)    返回响应对象
get方法的参数
url：为请求的url地址
headers：字典形式的命名参数，传递请求头。模拟浏览器获取想获取的数据，防止反扒检测；
示例：headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/54.0.2840.99 Safari/537.36"}
params：字典形式命名参数，传递请求参数，获取想要获取的数据
示例：params = {'wd':'长城'}
request.post()     post请求
使用方法：
response = requests.post(url, headers=headers, data= data)
post方法的参数
url：为请求的url地址（不包含锚点）
headers：字典形式的命名参数，传递请求头。模拟浏览器获取想获取的数据，防止反扒检测；
示例：headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/54.0.2840.99 Safari/537.36"}
data：
当时普通的post请求的时候，传递的参数是字典形式命名参数
Formdata参数示例：data = {'wd':'长城'}
当时ajax异步post请求的时候，传递的的字符串形式的参数
Payload参数示例：data = json.dumps({'wd': '长城'})
还要加上请求头 ： 'Content-Type': 'application/json'
requests.utils   工具
requests.utils.qoute(url)     将url编码
requests.utils.unqoute(url)     将url进行解码
parsed = requests.utils.urlparse(url)     将url进行  协议、ip+端口、文件路径、参数等的拆分
parsed.scheme       获取协议名
parsed.netloc    获取ip+端口（或者是域名+端口）
parsed.path    获取路径
parsed.params     获取问号后面的参数
response 响应的常用属性和方法
response.text               获取响应内容html字符串
获取html字符串，结果是`str`类型
其编码方式，是requests根据响应头做出的有根据的推测，尝试使用这个编码方式来解码
response.encoding               规定解码的格式
可以在使用text属性获取响应字符串之前先规定编码格式，按照设定的编码格式进行解码
respones.content         获取响应内容的二进制(bytes)数据
通常我们在获取到内容的时候尝试对二进制数据进行解码，下面三种方法能够解决后续我们100%的编解码的需求
response.content.deocde()           获取二进制响应内容并解码成为utf-8编码格式的字符串
response.content.deocde("gbk")         获取二进制响应内容并解码成为gbk编码格式的字符串
response.text      在使用content解码失败的时候，可以尝试让request去根据推测进行解码
response.status_code       查看响应状态码
response.request.headers     查看请求头
response.headers      查看响应头
response.request.url     查看响应url
response.url       查看请求url
响应的url可能与请求url不同，比如304重定向，响应的url为重定向的url

***找到想要请求url和请求参数的方法***
寻找想要的url
第一种情况：表单提交请求，并且表单有action属性
通过form表单的action属性/a标签的href属性来找到对应的请求url
form表单提交时获取表单中所有的name、value键值对
构造成字典，通过requests函数的data参数进行请求
第二种情况：通过ajax异步发出的请求（非action表单提交请求）
查看抓包结果，存在我们想要最多的数据会是我们想要的响应，其url为主要的请求url
当有一些数据不存在该主要的请求中，那说明有js进行了其他异步请求获取了数据
我们可以先找到其他请求的响应，找到其url地址，其url地址一般有两种构建途径
1.一般的情况下和url相关的信息会存在主请求的响应中，我们可以通过在主响应中获取其他请求的请求url关键字，进行url的构建
2.有的时候url地址或参数是通过js动态生成的，这时候，我们需要去寻找对应的js文件来观察js是怎样生成的动态url和参数等（参考下面）
寻找想要的js文件（当不是通过form表单的action/a标签href属性提交请求的时候）
找到进行url请求的js文件
第一种方法
使用开发者选择工具，点击会触发请求js的元素
点击右边的event Listener，会获取到对应的点击事件
点击事件所在的js文件
第二种方法
先了解js文件中会出现的关键字，一般可以查找url中动态参数名称；
通过点击右上角的菜单search all file
查找关键字，找到后点击进入对应的js文件
会跳转到source界面，点击{}展开代码，显示文件全部的代码
找到想要找的事件函数，并在想要查看实现方法的函数前边加上断点
通过点击右边的调试工具，可以查看函数的具体执行步骤和实现方法
了解了实现的方法后，可以在爬虫请求的时候，实现同样的方法，然后生成参数和url地址，进行请求
使用代理获取响应
什么叫代理
在对网站爬取的时候，通过访问代理服务器，让代理服务器帮我们对目标服务器进行请求，然后通过代理服务器将响应返回给我们；
代理的作用
在需要大量的进行数据的爬取的时候
防止在同一时间，使用同一个ip地址对同一个服务器进行大量的访问，被反扒出来
在爬取网站的时候，通过代理服务器可以隐藏我们的真实ip地址，隐藏我们的身份，但是有的代理服务器不能隐匿我们的mac地址，如果想要隐匿我们的mac地址需要使用高匿代理服务器
代理服务器的原理

正向代理/反向代理
正向代理
在我们请求代理服务器的时候，我们知道我们的最终目标ip地址，比如我们使用代理服务器爬取百度服务器的内容
反向代理
我们在请求服务器的时候，不知道最终目标ip地址，比如我们在访问nginx反向代理的服务器的时候，我们只是在访问nginx服务器，我们并不知道nginx去访问哪一个ip地址
requests使用代理
使用方法：
requests.get(url, proxies = proxies)
需要参数
proxies：字典类型命名参数，指定代理服务器支持的协议，和代理服务器的请求地址
示例：proxies = {"协议":"协议://ip:port"}，传递代理服务器支持协议类型（http/https），和代理服务器的ip和端口
备注：proxies字典参数最多只能接受两个键值对，一个键是http，一个键是https，定义在进行http/https请求的时候分别使用的代理服务器
代理服务器注意点：
http的url地址要使用http的代理，https的要使用https的代理
透明度低的代理能够被对方服务器找到我们的真实的ip，可能会导致代理的效果不明显
cookie与session的请求
cookie和session的区别
cookie存在浏览器本地，session在服务端
cookie不安全，session不会将数据暴露在客户端，比较安全
session占用性能，会加长请求的时间
cookie存储是有上限的，session没有
请求带上cookies的好处
能够请求登陆后的页面
带上cookie反反扒，用登录成功的cookie来进行伪造
伪造请求带上cookie的不好的地方
使用同一个cookie，不间断的访问同一个服务器的时候，可以被对方识别为爬虫
解决方法：使用多个用户名密码，多账号，随机选用账号进行服务器的访问，模拟多人访问
模拟登录cookie/session请求的方法
第一种
session请求登录接口
实例化一个session对象
session = requests.session()
使用session请求登录接口，session对象会将响应中的cookie进行保存
session.post(url, data=data, headers=headers)
也可以使用get请求
再使用session对象请求其他需要登录的url地址，会自动带上cookie进行访问
response =session.get(url, params=params, headers=headers)
session对象的作用
对响应的cookie进行保存，后面的访问会自动携带cookie
第二种
要获取了登录后的cookie请求字符串
headers中放cookie请求头字符串
第三种
把cookie的每一个name和value组成一个字典
将字典传给requests请求中的cookies参数接收
获取response中的cookie的方法
获取response中的cookie对象
response.cookies      获取respone的cookie对象
response.cookies只能获取服务器主动设置的cookie，不能获取我们手动创建的cookie
返回的数据类型是列表嵌套字典，每一个字典包含一个cookie的所有信息
将cookie对象转化为字典类型的方法
requests.utils.dict_from_cookiejar(response.cookies)
将python字典类型转化为cookie对象类型
requests.utils.cookiejar_from_dict( {'key': value } )
requests请求常见问题
 SSl证书验证问题
问题产生原因
请求协议为https的网站需要向机构申请证书，这样用户才能直接通过https访问，但是有的网站（比如12306）的整数是自己研发的，这样浏览器不会在进行请求的时候，会产生一个证书异常，我们需要点击浏览器上的继续访问，来请求服务器，当我们使用爬虫来进行访问的时候，会直接报出异常
解决办法：
response = requests.get("https://www.12306.cn/mormhweb/ ", verify=False)
在请求非机构证书的网站的时候，我们需要在请求方法中加上verify=False 的参数，就不会报异常
请求超时问题
问题产生原因
当我们进行请求的时候，我们可能因为在请求摸一个url的时候产生一些异常，导致长时间没有请求成功，由于请求一直在进行，导致后面的程序无法正常执行，验证影响程序的效率或者导致程序终止
解决办法
应用到的方法
from retrying import retry   
@retry(stop_max_attempt_number=n)      装饰函数，表示函数如果报错将会再次执行，直到第n次如果依然报错，那将抛出异常
 response = requests.get(url,timeout=10)     设置请求的超时时间，如果限定时间内没有请求成功，将会抛出异常
assert response.status_code == m      assert为断言关键字，如果后面的条件表达式为false将会排出异常
示例：

数据处理的技巧
字符串的格式化
"abc{}abc".format()
不同于%号的格式化，{ }方式的格式化，可以接收任意类型的格式化
format  接收的参数与格式化{ }的数量相等
在通过字符串格式化构建url进行requests请求时候，尽量使用{ }来格式化，因为在浏览器会将url格式化，%有时候会按照特殊字符处理；
 json数据与字符串之间的转化
import json
json.dumps( dict)    把python类型字典数据转化为json字符串
json.loads('json' )    把json数据转化为python的字典类型
扁平化赋值表达式
name = "a" if lang==b else "c"  
if后面的条件如果成立，那么就把if前的值赋给str
否则if否面的条件不成立，就把else后的值赋给str
name = a and 'b' or 'c' 
如果a为true，name等于b
如果a为false，name等于c
列表推导式和字典推导式
列表推倒式
[i for i in range(10) if i%2==0]
字典推导式
{i+1:i for i in range(10) if i%3==0}
注意：
在列表推导式中可以使用if判断，但是不能够使用else
tips
linux命令重命名
作用
可以将一段的linux命令，重名为其他比较短而且容易记忆的命令，方便我们的调用
设置方法
修改家目录下的.bashrc文件
添加  alias  新的命令='原命令 -选项/参数'
保存退出   source .bashrc
已经可以使用新的命令了 
将数据写入文件
文件存储内容的方法
with open('文件路径', encoding='utf-8') as f:
f.write(content)
encoding 如果文件不存在，相当于我们创建一个文件进行写入，我们可以通过encoding来指定写入内容的编码格式，如果不指定，可能会报错

常见的反扒思路
尽量减少请求的次数
能抓列表页不抓详细页
保存html页面，有利于重复使用
多分析一个网站的不同类型页面
手机极速版页面
wap手机版页面
web网页
app抓包软件
进行请求伪装
多带一些请求头，有的时候请求头带的不同，服务器返回的结果不同
代理ip，设置代理ip池，定期更新代理ip池
在浏览器会根据cookie内容判断爬虫的时候，携带cookie（但是注意不要一直携带同一个cookie）
需要获取登录之后的数据的时候，要进行模拟登录，获取cookie后，在进行数据的获取
利用多线程/分布式（尽可能的快速抓取）
在可能的情况下尽可能的使用多线程和分布式爬虫