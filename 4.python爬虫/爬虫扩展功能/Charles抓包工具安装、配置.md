Charles介绍
Charles的作用
Charles是一个网络抓包工具，相比Fiddler，其功能更为强大，而且跨平台支持得更好，
可以选用它来作为主要的移动端抓包工具。
相关链接
官方网站：https://www.charlesproxy.com
下载链接：https://www.charlesproxy.com/download
Charles的配置
HTTPS证书的配置
Windows系统的证书配置
1.首先打开Charles，点击Help→SSL Proxying→Install Charles Root Certificate，即可进入证书的安装页面。
2.接下来，会弹出一个安装证书的页面，点击“安装证书”按钮，就会打开证书导入向导。
3.直接点击“下一步”按钮，此时需要选择证书的存储区域，点击第二个选项“将所有的证书放入下列存储”，然后点击“浏览”按钮，从中选择证书存储位置为“受信任的根证书颁发机构”，再点击“确定”按钮，然后点击“下一步”按钮。
4.再继续点击“下一步”按钮完成导入。
Mac系统的证书配置
1.如果你的PC是Mac系统，可以按照下面的操作进行证书配置。
2.同样是点击Help→SSL Proxying→Install Charles Root Certificate，即可进入证书的安装页面。
3.接下来，找到Charles的证书并双击，将“信任”设置为“始终信任”即可，如图1-48所示。

代理配置
具体操作是点击Proxy→Proxy Settings，打开代理设置页面，确保当前的HTTP代理是开启的，如图1-49所示。这里的代理端口为8888，也可以自行修改。

将手机连接到Charles并安装证书
ios系统
1.将手机和电脑连在同一个局域网下，可以将PC设置为热点，手机连接其热点
2.将PC的ip设置为手机的代理ip，Settings > General > Network > Wi-Fi.

3.设置完毕后，电脑上会出现一个提示窗口，询问是否信任此设备，此时点击Allow按钮即可。这样手机就和PC连在同一个局域网内了，而且设置了Charles的代理，即Charles可以抓取到流经App的数据包了。
4.安装Charles的HTTPS证书，在电脑上打开Help→SSL Proxying→Install Charles Root Certificate on a Mobile Device or Remote Browser

5.在手机浏览器中打开chls.pro/ssl下载证书，点击“设置”→“通用”→“关于本机”→“证书信任设置”中将证书的完全信任开关打开。
 Android系统
1.将手机和电脑连在同一个局域网下，可以将PC设置为热点，手机连接其热点
2.将PC的ip设置为手机的代理ip，Settings > Wi-Fi > hold current Wi-Fi network > Modify Network > Show advanced options > Proxy settings
3.设置完毕后，电脑上就会出现一个提示窗口，询问是否信任此设备, 此时直接点击Allow按钮即可。
4.安装Charles的HTTPS证书，在电脑上打开Help→SSL Proxying→Install Charles Root Certificate on a Mobile Device or Remote Browser

5. 在手机浏览器上打开chls.pro/ssl，这时会出现一个提示框，为证书添加一个名称，然后点击“确定”按钮即可完成证书的安装。