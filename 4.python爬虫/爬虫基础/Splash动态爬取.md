Splash的功能介绍和运行
Splash的简介
Splash是一个JavaScript渲染服务，是一个带有HTTP API的轻量级浏览器，同时它对接了Python中的Twisted和QT库。利用它，我们同样可以实现动态渲染页面的抓取。
Splash文档
Splash官方文档
https://splash.readthedocs.io/en/stable/scripting-ref.html
Splash-api官方文档
https://splash.readthedocs.io/en/stable/api.html
Splash的功能
异步方式处理多个网页渲染过程；
获取渲染后的页面的源代码或截图；
通过关闭图片渲染或者使用Adblock规则来加快页面渲染速度；
可执行特定的JavaScript脚本；
可通过Lua脚本来控制页面渲染过程；
获取渲染的详细过程并通过HAR（HTTP Archive）格式呈现。
Splash服务的运行
docker run -d -p 8050:8050 scrapinghub/splash
打开http://localhost:8050/即可看到其Web页面
Splash-Lua脚本介绍
Splash的运行机制
Splash加载和操作渲染过程，可以理解是由一段Lua代码来操控
Splash留出来了一个main() 函数的接口，我们可以通过编写该段函数，操作其传入的splash对象，Splash会调用其指令并执行对应的操作
可以在浏览器界面编写并点击render me来启动，也可以通过调用splash-api的方式来调用
Splash Lua脚本示例
function main(splash, args)
assert(splash:go(args.url))         请求url
assert(splash:wait(0.5))       等待0.5s
return {       返回结果
html = splash:html(),            获取页面源码
png = splash:png(),        获取页面截图
har = splash:har(),        获取请求的详细har信息
}
end
Splash异步处理
Splash异步的介绍
在脚本内调用的wait()方法类似于Python中的sleep()，其参数为等待的秒数。当Splash执行到此方法时，它会转而去处理其他任务，然后在指定的时间过后再回来继续处理。
另外，这里做了加载时的异常检测。go()方法会返回加载页面的结果状态，如果页面出现4xx或5xx状态码，ok变量就为空
Splash异步脚本示例
function main(splash, args)
local example_urls = {"www.baidu.com", "www.taobao.com", "www.zhihu.com"}
local urls = args.urls or example_urls
local results = {}
for index, url in ipairs(urls) do
local ok, reason = splash:go("http://" .. url)
if ok then
splash:wait(2)
results[url] = splash:png()
end
end
return results
end
Splash负载均衡的配置
搭建多个splash服务节点
docker run -d -p 8050:8050 scrapinghub/splash
配置负载均衡服务(使用nginx)
nginx配置节点
​            http {
​                upstream splash {
​                    least_conn;             
​                    server 41.159.27.223:8050 weight=4;
​                    server 41.159.27.221:8050 weight=2;
​                    server 41.159.27.9:8050 weight=2;
​                    server 41.159.117.119:8050 weight=1;
​                }
​                server {
​                    listen 8050;
​                    location / {
​                        proxy_pass http://splash;
​                    }
​                }
​            }
nginx配置认证
现在Splash是可以公开访问的，如果不想让其公开访问，还可以配置认证，这仍然借助于Nginx。可以在server的location字段中添加auth_basic和auth_basic_user_file字段，具体配置如下：
这里使用的用户名和密码配置放置在/etc/nginx/conf.d目录下，我们需要使用htpasswd命令创建。例如，创建一个用户名为admin的文件，相关命令如下
htpasswd -c .htpasswd admin
接下来就会提示我们输入密码，输入两次之后，就会生成密码文件，其内容如下：
配置完成后，重启一下Nginx服务： sudo nginx -s reload
​                server {
​                    listen 8050;
​                    location / {
​                        proxy_pass http://splash;
​                        auth_basic "Restricted";
​                        auth_basic_user_file /etc/nginx/conf.d/.htpasswd;
​                    }
​                }
Splash对象的常用属性
加载参数
获取方法一
​            function main(splash, args)
​                local url = args.url
​            end
获取方法二
​            function main(splash)
​                local url = splash.args.url
​            end
开启关闭javascript
splash.js_enabled = false/true
设置加载的超时时间
splash.resource_timeout = 2
单位是秒。如果设置为0或nil（类似Python中的None），代表不检测超时。
禁止加载图片
splash.images_enabled = true/false
禁止开启插件(flash等)
splash.plugins_enabled = true/false
滚动条
splash.scroll_position = {x=100, y=400}
滚动条向右滚动100像素，向下滚动400像素
Splash对象常用方法
请求某链接
使用方法
splash:go{url, baseurl=nil, headers=nil, http_method="GET", body=nil, formdata=nil}
url：请求的URL。
baseurl：可选参数，默认为空，表示资源加载相对路径。
headers：可选参数，默认为空，表示请求头。
http_method：可选参数，默认为GET，同时支持POST。
body：可选参数，默认为空，发POST请求时的表单数据，使用的Content-type为application/json。
formdata：可选参数，默认为空，POST的时候的表单数据，使用的Content-type为application/x-www-form-urlencoded。
return: 结果ok和原因reason的组合，如果ok为空，代表网页加载出现了错误，此时reason变量中包含了错误的原因
使用示例
​            function main(splash, args)
​                local ok, reason = splash:go{"http://httpbin.org/post", http_method="POST", body="name=Germey"}
​                if ok then
​                    return splash:html()
​                end
​            end
设置定时任务
使用示例(0.2s后获取网页截图)
​                    function main(splash, args)
​                             local snapshots = {}
​                             local timer = splash:call_later(function()
​                                 snapshots["a"] = splash:png()
​                                 splash:wait(1.0)
​                                 snapshots["b"] = splash:png()
​                                 end, 0.2)
​                             splash:go("https://www.taobao.com")
​                             splash:wait(3.0)
​                             return snapshots
​                    end
模拟发送get请求
使用方法
response = splash:http_get{url, headers=nil, follow_redirects=true}
url：请求URL。
headers：可选参数，默认为空，请求头。
follow_redirects：可选参数，表示是否启动自动重定向，默认为true
使用示例
​            function main(splash, args)
​                local treat = require("treat")
​                local response = splash:http_get("http://httpbin.org/get")
​                    return {
​                        html=treat.as_string(response.body),
​                        url=response.url,
​                        status=response.status
​                    }
​            end
模拟发送post请求
使用方法
response = splash:http_post{url, headers=nil, follow_redirects=true, body=nil}
url：请求URL。
headers：可选参数，默认为空，请求头。
follow_redirects：可选参数，表示是否启动自动重定向，默认为true。
body：可选参数，即表单数据，默认为空。
使用示例
​            function main(splash, args)
​                local treat = require("treat")
​                local json = require("json")
​                local response = splash:http_post{"http://httpbin.org/post",    
​                    body=json.encode({name="Germey"}),
​                    headers={["content-type"]="application/json"}
​                }
​                return {
​                    html=treat.as_string(response.body),
​                    url=response.url,
​                    status=response.status
​                }
​            end
获取页面加载过程描述等信息
使用方法
splash:har()        获取加载过程详细信息
splash:url()        获取正在访问的url
页面cookie相关操作
使用方法
splash:get_cookies()       获取当前页面的cookie 
cookies = splash:add_cookie{name, value, path=nil, domain=nil, expires=nil, httpOnly=nil, secure=nil}    给页面添加cookie
splash:clear_cookies()    清空当前页面所有cookie
设置页面大小
使用方法
splash:get_viewport_size()    获取页面宽高
splash:set_viewport_size(width, height)    设置页面宽高
splash:set_viewport_full()    设置页面铺满
设置User-Agent
使用示例
​                    function main(splash)
​                            splash:set_user_agent('Splash')
​                            splash:go("http://httpbin.org/get")
​                            return splash:html()
​                    end
设置请求头
使用示例
​                    function main(splash)
​                        splash:set_custom_headers({
​                                ["User-Agent"] = "Splash",
​                                ["Site"] = "Splash",
​                        })
​                        splash:go("http://httpbin.org/get")
​                        return splash:html()
​                    end
节点操作
使用方法
element = splash:select("#kw")    通过css选择器选择元素, 返回选中的第一个元素
element = splash:select_all("#kw")    通过css选择器选择元素, 返回选中所有元素
element.send_text('test')     向节点输入文本
element.mouse_click()    点击节点
等待wait
使用方法
ok, reason = splash:wait{time, cancel_on_redirect=false, cancel_on_error=true}
time：等待的秒数。
cancel_on_redirect：可选参数，默认为false，表示如果发生了重定向就停止等待，并返回重定向结果。
cancel_on_error：可选参数，默认为false，表示如果发生了加载错误，就停止等待。
return: 结果ok和原因reason的组合。
使用示例
​            function main(splash)
​                splash:go("https://www.taobao.com")
​                splash:wait(2)
​                return {html=splash:html()}
​            end
调用JavaScript定义的方法
使用示例
​                    function main(splash, args)
​                            local get_div_count = splash:jsfunc([[
​                                    function () {
​                                    var body = document.body;
​                                    var divs = body.getElementsByTagName('div');
​                                    return divs.length;
​                                    }
​                                    ]])
​                            splash:go("https://www.baidu.com")
​                            return ("There are %s DIVs"):format(get_div_count())
​                    end
执行JavaScript代码(表达式类)
使用示例
local title = splash:evaljs("document.title")  执行JavaScript代码并返回最后一条JavaScript语句的返回结果
执行JavaScript代码(声明类)
使用示例
​                    function main(splash, args)
​                         splash:go("https://www.baidu.com")
​                          splash:runjs("foo = function() { return 'bar' }")
​                          local result = splash:evaljs("foo()")
​                          return result
​                    end
加载js代码
使用方法
此方法只负责加载JavaScript代码或库，不执行任何操作。如果要执行操作，可以调用evaljs()或runjs()
ok, reason = splash:autoload{source_or_url, source=nil, url=nil}
source_or_url：JavaScript代码或者JavaScript库链接。
source：JavaScript代码。
url：JavaScript库链接
使用示例
​                    function main(splash, args)
​                          splash:autoload([[
​                                  function get_document_title(){
​                                         return document.title;
​                            }
​                          ]])
​                          splash:go("https://www.baidu.com")
​                          return splash:evaljs("get_document_title()")
​                    end
页面内容的操作方法
使用方法
splash:set_content("<html><body><h1>hello</h1></body></html>")     添加页面内容
splash:html()    获取当前页面源码
获取png/jpeg格式的页面截图
使用方法
splash:png()
splash:jpeg()
设置代理
使用示例
​                    function main(splash, args)
​                           request = splash:on_request(function(request)
​                                    request:set_proxy{host, port, username=nil, password=nil, type='HTTP/SOCKS5')
​                                    end)
​                    end
Splash-API的使用
Splash API的介绍
Splash Lua脚本的用法，但这些脚本是在Splash页面中测试运行的
Splash API就是我们可以通过python程序使用Splash服务的方法
Splash给我们提供了一些HTTP API接口，我们只需要请求这些接口并传递相应的参数即可
常用Splash API
render.png
接口介绍
通过width和height来控制宽高，它返回的是PNG格式的图片二进制数据。示例如下
常用参数
width、height、render_all
使用示例
​                        import requests
​                        url = 'http://localhost:8050/render.png?url=https://www.jd.com&wait=5&width=1000&height=700'
​                        response = requests.get(url)
​                        with open('taobao.png', 'wb') as f:
​                               f.write(response.content)
render.html
接口介绍
此接口用于获取JavaScript渲染的页面的HTML代码，接口地址就是Splash的运行地址加此接口名称
常用参数
url 、baseurl、timeout、resource_timeout、wait、proxy、js、js_source、filter、allowed_domin
allowed_content_types、viewport、images、headers、body、heep_method、save_args、load_args
使用示例
​            import requests
​            url = 'http://localhost:8050/render.html?url=https://www.baidu.com'
​            response = requests.get(url)
​            print(response.text)
render.har
接口介绍
此接口用于获取页面加载的HAR数据
使用示例
curl http://localhost:8050/render.har?url=https://www.jd.com&wait=5
render.json
接口介绍
此接口包含了前面接口的所有功能，返回结果是JSON格式
以JSON形式返回了相应的请求数据
可以通过传入不同参数控制其返回结果。比如，传入html=1，返回结果即会增加源代码数据；传入png=1，返回结果即会增加页面PNG截图数据；传入har=1，则会获得页面HAR数据
使用示例
curl http://localhost:8050/render.json?url=https://httpbin.org
execute
接口介绍
用此接口便可实现与Lua脚本的对接
可以用该接口，通过原生的Lua脚本来调用splash
可以通过lua_source参数传递Lua脚本给splash服务器进行请求和渲染服务
常用参数
timeout、allowed_domains、proxy、filters、save_args、load_args
使用示例
​                        import requests
​                        from urllib.parse import quote
​                        lua = '''
​                            function main(splash)
​                                return 'hello'
​                            end
​                         '''
​                        url = 'http://localhost:8050/execute?lua_source=' + quote(lua)
​                        response = requests.get(url)
​                        print(response.text)