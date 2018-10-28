Appium的介绍和安装
Appium的简介
Appium是移动端的自动化测试工具，类似于前面所说的Selenium，利用它可以驱动Android、iOS等设备完成自动化测试，比如模拟点击、滑动、输入等操作
Appium负责驱动移动端来完成一系列操作，对于iOS设备来说，它使用苹果的UIAutomation来实现驱动；对于Android来说，它使用UIAutomator和Selendroid来实现驱动。
同时Appium也相当于一个服务器，我们可以向它发送一些操作指令，它会根据不同的指令对移动设备进行驱动，以完成不同的动作。
Appium相关链接
GitHub：https://github.com/appium/appium
官方网站：http://appium.io
官方文档：http://appium.io/introduction.html
下载链接：https://github.com/appium/appium-desktop/releases
Python Client：https://github.com/appium/python-client
Appium Api：https://testerhome.comm/topics/3711
Appium的安装
通过Appium Desktop安装
Appium Desktop介绍
Appium Desktop支持全平台的安装，我们直接从GitHub的Releases里面安装即可
链接为https://github.com/appium/appium-desktop/releases。目前的最新版本
各平台的安装方法
下载exe安装包appium-desktop-Setup-1.1.0.exe
Mac平台可以下载dmg安装包如appium-desktop-1.1.0.dmg
Linux平台可以选择下载源码
通过Node.js安装
1.首先安装node.js
2.npm install -g appium
python-api安装
pip install Appium-Python-Client
开发环境的配置
Android开发环境配置
下载和配置Android Studio，下载地址为https://developer.android.com/studio/index.html?hl=zh-cn。下载后直接安装即可。
打开Android Studio，直接打开首选项里面的Android SDK设置页面，勾选要安装的SDK版本，点击OK按钮即可下载和安装勾选的SDK版本。
添加环境变量，添加ANDROID_HOME为Android SDK所在路径，然后再添加SDK文件夹下的tools和platform-tools文件夹到PATH中。
更详细的配置可以参考Android Studio的官方文档：https://developer.android.com/studio/intro/index.html。
Appium的使用(有界面版示例)
启动Appium服务
点击Appium
输入主机和端口号，点击Start Server按钮即可启动Appium服务
将手机连接到PC端
将手机通过数据线和PC相连
打开USB调试功能，确保PC可以连接到手机
可以在PC端测试是否连接成功
adb  devices  -l   查看连接的设备列表
如果找不到adb命令，请检查Android开发环境和环境变量是否配置成功
如果adb命令不显示设备信息，检查手机和PC的连接情况
启动App并操作
通过Appium内置驱动来操作
打开app
点击Appium中的Start New Session
配置启动App时的参数
platformName: 它是平台名称，需要区分Android或iOs，此处填Android
deviceName: 它是设备名称， 此处是手机的具体类型
appPackage： 它是App程序的包名
AppActivity: 它是入口Activity名，这里通常需要.开头
点击保存按钮，可以将此配置进行保存
点击Start Session，便会启动Android手机上的App，同时PC上会弹出一个调试窗口，里面包含了页面源码
操作应用App
选择元素
和浏览器的调试一样，只要点击作业页面中的元素，对应的元素就会高亮，中间栏会显示对应的源码
右侧是和元素包含的属性和事件等信息
操作元素
点击右侧元素的属性，便可以实现对元素的操作
操作的录制
点击中间栏的眼睛按钮，Appium会开始录制操作的动作，这时我们可以在窗口中操作APp的行为都会记录下来
Recorder处可以自动生成对应的语言代码。

通过Python代码来操作
打开app(初始化应用)
打开方法
用字典来配置Desired Capabilities 参数，新建一个Session
示例代码
server = "http://localhost:4723/wd/hub"
desired_caps = {
"platformName": "Android",
"deviceName": "MI_NOTE_Pro",
"appPackage": "com.tencent.mm",
"appActivity": ".ui.LauncherUI"
}
from appium import Webdriver
from selenium.webdriver.support.ui import WebDriverWait
driver = wevdriver.Remote(server, desired_caps)    # 启动app
查找元素
element = driver.find_element_by_android_uiautomator('new UiSelector().description("Animation")')
elements = driver.find_elements_by_android_uiautomator('new UiSelector().description("Animation")')
 点击操作
多点点击方法
driver.tap(self, positions, duration=None)
positions: 它是点击的位置组成的列表
duration: 它是点击持续的时间
通过tap方法实现触摸点击
该方法可以模拟手指点击，最多五个手指，也可以设置触摸的时间长短
单点点击方法
element.click()    对元素进行点击
示例代码
driver.tap([(100, 20), (100, 60), (100, 100)], 500)
屏幕拖动
元素间拖动方法
driver.scroll(self, origin_el, destination_el)
original_el: 拖动的起始元素
destination_el:拖动的终止元素
两点间的拖动方法
driver.swipe(self, start_x, start_y, end_x, end_y, duration=None)
start_x: 它是开始位置的横坐标
start_y: 它是开始位置的纵坐标
end_x: 它是终止位置的横坐标
end_y: 它是终止位置的纵坐标
duration: 它是持续时间，单位是毫秒
两点间的快速的滑动
driver.swipe()
start_x: 它是开始位置的横坐标
start_y: 它是开始位置的纵坐标
end_x: 它是终止位置的横坐标
end_y: 它是终止位置的纵坐标
示例代码
driver.scroll(el1, el2)
driver.swipe(100, 100, 100, 400, 5000)
driver.flick(100, 100, 100, 400)
元素的拖拽
拖拽方法
driver.drag_and_drop(self, origin_el, destination_el)
origin_el: 被拖拽的元素
destination_el：拖拽的目标元素
示例代码
driver.drag_and_drop(el1, el2)
文本的输入
输入的方法
ele.set_text('Hello')
示例代码
element = driver.find_element_by_id('name')
element.set_text("Hello")
动作链
使用动作链的方法
from appium import TouchAction()
action = TouchAction()
action.tap(element).perfoem()
常用的动作链还有: tap、press、long_press、release、move_to、wait、cancel
示例代码
from appium import TouchAction()
action = TouchAction()
a_ele = self.driver.find_element_by_class_name("listView")
action.press(a_ele).move_to(x=10, y=0), move_to(x=10, y=-600).release()
action.perform
示例代码
Appium爬取微信
MomentsAppium.zip
Appium+ mitmdump 爬取京东
MitmAppiumJD.zip