动态HTML相关技术
 JavaScript
JavaScript 是网络上最常用也是支持者最多的客户端脚本语言。它可以收集 用户的跟踪数据,不需要重载页面直接提交表单，在页面嵌入多媒体文件，甚至运行网页游戏。
jQuery
jQuery 是一个十分常见的库,70% 最流行的网站(约 200 万)和约 30% 的其他网站(约 2 亿)都在使用。一个网站使用 jQuery 的特征,就是源代码里包含了 jQuery 入口。如果你在一个网站上看到了 jQuery，那么采集这个网站数据的时候要格外小心。jQuery 可 以动态地创建 HTML 内容,只有在 JavaScript 代码执行之后才会显示。如果你用传统的方 法采集页面内容,就只能获得 JavaScript 代码执行之前页面上的内容。
Ajax  / DHTML
我们与网站服务器通信的唯一方式，就是发出 HTTP 请求获取新页面。如果提交表单之后，或从服务器获取信息之后，网站的页面不需要重新刷新，那么你访问的网站就在用Ajax 技术。
Ajax 其实并不是一门语言,而是用来完成网络任务(可以认为 它与网络数据采集差不多)的一系列技术。Ajax 全称是 Asynchronous JavaScript and XML(异步 JavaScript 和 XML)，网站不需要使用单独的页面请求就可以和网络服务器进行交互 (收发信息)。
动态HTML技术的影响
动态技术的效果对爬虫的影响（如下情况我们最好使用selenium来解决）
动态生成HTML内容，在获取内容的时候，只能获取到js处理之前的内容（可能是js再次发起异步请求，或者是对内容传输过程加密，js解密显示）
动态的生成url地址，在请求的时候，不知道其js的处理逻辑，不能解析js处理后的url地址
动态的生成请求参数，在请求的时候 ，不知道其js的处理逻辑，不知道js处理后的请求参数
服务器端动态生成的验证码（请求同一验证码url地址，返回图片不同），这样无法通过请求url下载图片来获取当前请求的验证码
怎么解决
直接从 JavaScript 代码里采集内容（费时费力）
分析js代码的功能逻辑
用python来重新实现其功能，完成参数或者内容的解析
用 Python 的 第三方库运行 JavaScript，直接模拟浏览器操作，并获取element中的js渲染后的代码内容。
通过使用selenium库来实现直接在浏览器上运行的效果，可以获取浏览器中elements中的源码（js解析后的内容）
通过模拟浏览器进行请求，不需要我们构建请求的url地址和请求参数等内容
获取模拟登录后的cookie等，保持登录状态
Selenium的使用
所需工具介绍
selenium的介绍
Selenium是一个Web的自动化测试工具，最初是为网站自动化测试而开发的，Selenium 可以直接运行在浏览器上，它支持所有主流的浏览器（包括PhantomJS这些无界面的浏览器），可以接收指令，让浏览器自动加载页面，获取需要的数据，甚至页面截屏
使用selenium的场景
在模拟登录的时候，可以模拟获取到动态的url、请求参数、等进行登录，还可以获取到cookie
获取动态HTML页面内容，即请求页面中的内容是经过js修改过动态计算生成的内容
各种driver的下载
chromedriver官网网站
https://sites.google.com/a/chromium.org/chromedriver
chromedriver下载地址
https://chromedriver.storage.googleapis.com/index.html
geckodriver官方网址
https://github.com/mozilla/geckodriver
geckodriver下载地址
https://github.com/mozilla/geckodriver/releases
selenium官方文档
http://selenium-python.readthedocs.io/api.html
创建driver对象、并设置窗口大小
from selenium import webdriver 
from selenium.webdriver import ActionChains
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WevDriverWait
from selenium.webdriver.support import export expected_conditions as EC
from selenium.webdriver.remote.command import Command
driver = webdriver.PhantomJS( )
driver.set_window_size(width, height)      设置浏览器的窗口大小，截图可以根据窗口大小来截取
driver.max_windiw( )       让窗口显示最大
加载网页
driver = webdriver.PhantomJS( )      加载浏览器driver
driver.get("http://www.baidu.com/")    请求url地址
寻找元素
driver.find_element_by_id()                                  通过id找到元素
driver.find_element_by_name()                           通过name属性寻找元素
driver.find_element_by_xpath()                            通过xpath来寻找元素
driver.find_element_by_link_text()                       通过文本内容寻找a标签
driver.find_element_by_partial_link_text()            通过文本内容包含内容寻找a标签       
driver.find_element_by_tag_name()                     通过标签名获取元素
driver.find_element_by_class_name()                  通过类型获取元素
driver.find_element_by_css_selector()                  通过css选择器选择元素
备注：
find_element 和find_elements的区别：find_elements用来寻找多个元素, 返回一个和返回一个列表
by_css_selector的用法示例： #food span.dairy.aged
元素的交互
element.send_keys(“长城”)            在input元素value中输入内容
element.clear()                                  清空input标签中输入的内容
element.click( )                                  触发元素的事件
动作链的使用
source_ele = driver.find_element_by_css_selector('#draggable')
target_ele = driver.find_element_by_css_selector('#droppable')
actions = ActionChains(driver)         创建动作链
actions.drag_and_drop(source_ele, target_ele)         调用动作链的方法
actions.persorm()     执行动作链
指定原生Js代码
js_code = "window.scrollTo(0, document.body.scrollHeight)"    定义js代码
driver.execute_script(js_code)          指定js代码
获取节点元素的属性和文本值等
element.get_attribute('属性名')     可以获取元素的属性
element.text                                    可以获取文本内容
element.id                                        获取元素的id属性
element.size                                     获取元素的长宽
element.size['width']                    获取元素的宽
element.size['height']                  获取元素的高
element.tag_name                       获取元素的标签名称
element.location                           获取元素的坐标
切换Frame
driver.switch_to.frame('frame_id' )            切换到iframe标签页面中
driver.switch_to.parent_frame()                      切换到父frame中
driver.switch_to.default_content()             切换到默认的frame中
等待
隐式等待
等待规则
如果selenium没有在DOM中找到节点元素，将等待一段时间
时间到达设定的时候后, 再次查找元素，若找不到元素则抛出找不到节点的异常，默认时间是0
等待设置方法
driver = webdriver.Chrome()
driver.implicity_wait(10)     给driver设置了10秒的隐式等待时间
driver.get('https://www.baidu.com')
ele = driver.find_element_by_class_name('kw')      寻找元素, 找不到元素的话会等待10秒钟, 若还找不到元素则抛出异常
显示等待
等待规则
规定了一个最长的等待时间，如果在规定的时间内加载出来了这个节点, 就返回找到节点
如果没有找点节点则抛出超时异常
等待设置方法
driver = webdriver.Chrome()
driver.get('https://www.baidu.com')
wait = WebDriverWait(driver, 10)        设置了10秒的最大显示等待时间
ele = wait.until(EC.presence_of_element_located((By.ID, 'kw')))        设置等待条件，并等待寻找到节点
ele_1 = wait.until(EC.element_to_be_clickable((By.CSS_SELECTOR, '#bt')))    设置等待条件，并等待寻找到节点
常用的等待条件
title_is             标题是某内容
title_contains             标题包含某内容
presence_of_element_located               节点加载出来
visibility_of_element_located                 节点课件, 传入定位元组
visibility_of                    节点可见，传入节点对象
presence_of_all_elements_located          所有节点加载出来
text_to_be_present_in_element             某个节点中包含某文字
text_to_be_present_in_element_value          某个节点值包含某文字
frame_to_be_available_and_switch_to_it      加载并切换
invisibility_of_element_located             节点不可见
element_to_be_clickable     节点可点击
element_to_be_seleted    节点可选择，传入节点对象
element_located_to_be_selected     节点可选择，传入定位元组
element_selection_state_to_be          传入节点对象以及状态，相等返回True, 否则返回false
element_located_selection_state_to_be    传入定位元组以及状态，相等返回True，否则返回False
alert_is_present    是否出现警告
其他的等待
driver.set_page_load_timeout(10)         设置页面加载的超时时间
driver.set_script_timeout(10)          设置执行script脚本的超时时间
鼠标的移动和点击
鼠标的移动
actions = ActionChains(driver)
actions.move_to_element(element).move_by_offset( lx , ly )
actions.perform()
element   移动到哪个元素里面
lx  相对元素左边界的距离
ly 相对元素上边界的距离
鼠标的点击
from selenium.webdriver.remote.command import Command
driver.execute(Command.MOUSE_DOWN, pamas)
第一个参数为时间类型
第二个参数为命令的附加命令，为字典类型
操作driver
页面请求参数相关
driver.page_source                获取浏览器中element中的内容（即页面的源码）
driver.current_url()                获取当前页面url地址
driver.back()                          退回上一个页面
driver.forward()                     返回到前面一个页面
driver.flush()                          刷新当前页面
driver.get_cookies()               获取所有的cookie，返回的是列表嵌套字典（存储的每个cookie信息）
driver.delete_cookie("key")    删除一条cookie
driver.delete_all_cookies()      删除所有的cookie内容
{cookie['name']: cookie['value'] for cookie in driver.get_cookies( )}   将cookie构造请requests请求参数
driver.execute_script('window.open()')     开启一个浏览器的选项卡
driver.switch_to_window(driver.window_handles[1])         切换到第二个选项卡窗口
窗口截图相关
from PIL import Image
driver.save_screenshot("长城.png")     截图浏览器，没有图形界面可以通过截图来观察界面
img = Image.open(io.BytesIO(driver.get_screenshot_as_png()))            获取截图并直接读取到内存
img_small = img.crop((x0, y0, x1, y1)).convert('L')     对图片进行截图，四个参数代表(左、上、右、下)坐标，并将彩图转化为灰度图
img_small.save('--path--')    将图片进行保存
width  =  img.size[0]    获取图片的宽
height  =  img.size[1]    获取图片的高
img1.load( )[ i, j ]       获取图片该像素点的rgb值
浏览器开关相关
driver.close()    关闭当前页面，当页面全部都关闭后，退出浏览器driver
driver.quit()      退出浏览器driver
selenium 的driver配置
代理的设置
phantomjs设置代理ip
browser=webdriver.PhantomJS(PATH_PHANTOMJS)
proxy=webdriver.Proxy()
proxy.proxy_type=ProxyType.MANUAL
proxy.http_proxy='1.9.171.51:800'
proxy.add_to_capabilities(webdriver.DesiredCapabilities.PHANTOMJS)
browser.start_session(webdriver.DesiredCapabilities.PHANTOMJS)
browser.get('http://1212.ip138.com/ic.asp';)
Chrome设置代理ip

firefox设置代理ip
方法一

方法二

禁用插件和无头浏览器的设置
chrome配置方法
设置无头浏览器

设置禁用图片和javascript

firefox配置方法
firefox_options = webdriver.FirefoxOptions()
firefox_options.set_preference(u"permissions.default.stylesheet", 2)   # 禁用css
firefox_options.set_preference(u"permissions.default.image", 2)       # 禁用图片
firefox_options.set_preference(u"dom.ipc.plugins.enabled.libflashplayer.so", "false")       # 禁用flash插件
firefox_options.set_headless()     # 设置为无头浏览器
driver = webdriver.Firefox(firefox_options=firefox_options)
设置请求头的方法