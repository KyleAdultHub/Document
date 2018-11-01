mitmproxy的介绍和安装配置
mitmproxy的作用
mitmproxy是一个支持HTTP和HTTPS的抓包程序，类似Fiddler、Charles的功能，只不过它通过控制台的形式操作。
mitmproxy还有两个关联组件，一个是mitmdump，它是mitmproxy的命令行接口，利用它可以对接Python脚本，实现监听后的处理；另一个是mitmweb，它是一个Web程序，通过它以清楚地观察到mitmproxy捕获的请求。
可以保存Http会话并进行分析
模拟客户端发起请求，模拟服务器端返回的响应
利用反向代理将流量发给指定的服务器
支持Mac和Linux上的透明代理
利用Python对Http请求和响应进行实时的处理
mitmproxy和mitmdump的区别
mitmproxy起到了代理服务的功能，手机和PC在同一个局域网内，可以将mitmproxy设置为手机的代理，这样数据都是通过mitmproxy妆发出去的，起到了中间人的作用
mtmdump可以实现将抓取到的请求和响应直接交给某个Python程序进行处理，比如提取和入库操作
mitmproxy相关链接
GitHub：https://github.com/mitmproxy/mitmproxy
官方网站：https://mitmproxy.org
PyPI：https://pypi.python.org/pypi/mitmproxy
官方文档：http://docs.mitmproxy.org
mitmdump脚本：http://docs.mitmproxy.org/en/stable/scripting/overview.html
下载地址：https://github.com/mitmproxy/mitmproxy/releases
DockerHub：https://hub.docker.com/r/mitmproxy/mitmproxy
mitmproxy的安装
linux下使用pip安装
pip3 install mitmproxy
这是最简单和通用的安装方式，执行完毕之后即可完成mitmproxy的安装，另外还附带安装了mitmdump和mitmweb这两个组件。
windows下安装
获取安装包，下载地址: https://github.com/mitmproxy/mitmproxy/releases/
在Windows上不支持mitmproxy的控制台接口，但是可以使用mitmdump和mitmweb。
Linux下源码安装
下载源码包，下载地址: https://github.com/mitmproxy/mitmproxy/releases/
它包含了最新版本的mitmproxy和内置的Python 3环境，以及最新的OpenSSL环境
tar -zxvf mitmproxy-2.0.2-linux.tar.gz
sudo mv mitmproxy mitmdump mitmweb /usr/bin
证书配置
证书配置的说明
对于mitmproxy来说，如果想要截获HTTPS请求，就需要设置证书。
mitmproxy在安装后会提供一套CA证书，只要客户端信任了mitmproxy提供的证书，就可以通过mitmproxy获取HTTPS请求的具体内容，否则mitmproxy是无法解析HTTPS请求的。
证书配置步骤
启动mitmdump
mitmdump
找到家目录.mitmproxy目录里面的CA证书
mitmproxy-ca.pem        PEM格式的证书私钥
mitmproxy-ca-cert.pem       PEM格式证书，适用于大多数非Windows平台
mitmproxy-ca-cert.p12        PKCS12格式的证书，适用于Windows平台
mitmproxy-ca-cert.cer           与mitmproxy-ca-cert.pem相同，只是改变了后缀，适用于部分Android平台
mitmproxy-dhparam.pem     PEM格式的秘钥文件，用于增强SSL安全性
在各平台上配置证书的过程
Windows平台
1.双击mitmproxy-ca.p12，会出现导入证书的页面向导
2.直接点击下一步，会出现密码设置的提示，这里不需要设置密码，直接点击下一步按钮即可
3.接下来选择证书的存储区域。这里点击第二选项，将所有的证书都放入下列存储，然后点击浏览按钮，选择证书的存储位置为收信人的根证书颁发急购，接着点击确定按钮，点击下一步
4.过程中出现安全警告直接点击是
Android
1.在Android手机上，需要将证书mitmproxy-ca-cert.pem文件发送到手机上，例如直接复制文件。
2.接下来，点击证书，便会出现一个提示窗口，这时输入证书的名称，然后点击“确定”按钮即可完成安装。
iOS
1.将mitmproxy-ca-cert.pem文件发送到iPhone上，推荐使用邮件方式发送，然后在iPhone上可以直接点击附件并识别安装。
点击“安装”按钮之后，会跳到安装描述文件的页面，点击“安装”按钮，此时会有警告提示，继续点击右上角的“安装”按钮，安装成功之后会有已安装的提示。
如果你的iOS版本是10.3及以上版本，还需要在“设置”→“通用”→“关于本机”→“证书信任设置”将mitmproxy的完全信任开关打开。
mitmproxy的使用
启动mitmproxy服务
mitmproxy    这样就会在8080端口上运行一个代理服务
将mitmproxy设置为手机端的代理
将PC的ip设置为手机的代理ip，Settings > Wi-Fi > hold current Wi-Fi network > Modify Network > Show advanced options > Proxy settings
发送请求
在手机端进行网络请求，便可以在mitmproxy的界面上看到对应的请求
查看/处理请求和响应
查看请求的详情
光标移动到对应的请求位置，点击ENTER
在请求中查看Request/Response/Detail
将光标移动到对应的分栏上，点击Tab
重新编辑请求
1.在请求中，点击e键进行编辑
2.按照高亮的部分，选择想要编辑的内容(比如: q：修改请求方参数，m：修改请求方式)
进入修改页面后,可以直接对内容进行修改
点击a可以增加一行参数
修改完成后点击Esc退出修改，q返回上一级页面
3.敲击a进行修改的保存
4.对修改后的请求重新发起请求
mitdump的使用
编写请求和响应的处理脚本
日志输出
介绍
ctx模块提供了不同等级的log将会打印不一样的颜色
使用示例
from mitmproxy import ctx 
def request(flow):
flow.request.headers['User-Agent'] = 'MitmProxy'
ctx.log.info("info")
ctx.log.warn("warn")
ctx.log.error("error")
Request对象处理
介绍
我们可以通过request() 方法实现对请求进行修改
request对象包含的属性: url、headers、cookies、host、method、scheme、port
使用示例
def request(flow):
request = flow.request
print(request.url)
Response对象处理
介绍
可以通过response() 方法实现对响应的操作，比如入库等操作
response对象包含的属性: status_code、deaders、cookies、text
使用示例
def response(flow):
response = flow.response
print(response.text)
启动mitmproxy服务
mitmdump  [OPTIONS]
-w: 可以指定将接货的数据都保存到此文件中
-s: 可以指定scripts.py 脚本文件，用来处理请求和响应，它需要放在当前命令的执行目录下
将mitmdump设置为手机端的代理
将PC的ip设置为手机的代理ip，Settings > Wi-Fi > hold current Wi-Fi network > Modify Network > Show advanced options > Proxy settings
发送请求
在手机端进行网络请求，便可以在mitmdump的日志中便可以看见对应的请求