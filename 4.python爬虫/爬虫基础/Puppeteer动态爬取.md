Puppeteer简介
Puppeteer介绍
Puppeteer 是一个node库，他提供了一组用来操纵Chrome的API, 通俗来说就是一个 headless chrome浏览器 (当然你也可以配置成有UI的，默认是没有的)。既然是浏览器，那么我们手工可以在浏览器上做的事情 Puppeteer 都能胜任, 另外，Puppeteer 翻译成中文是”木偶”意思，所以听名字就知道，操纵起来很方便，你可以很方便的操纵她去实现：
运行环境

1. Nodejs 的版本不能低于 v7.6.0, 需要支持 async, await.
2. 需要最新的 chrome driver, 这个你在通过 npm 安装 Puppeteer 的时候系统会自动下载的
  npm install puppeteer
  3.安装pyppeteer
  pip install pyppeteer
  Puppeteer的官方文档
  https://github.com/miyakogi/pyppeteer
  知识前提
  Puppeteer 几乎所有的操作都是 异步的, 为了使用大量的 then 使得代码的可读性降低，本文所有 demo 代码都是用 async, await 方式实现
  由于文档上大量使用async和await关键字,  需要提前了解Async/Await 异步编程
  Puppeteer的基本用法
  python示例

3. 先通过launch() 创建一个浏览器实例 Browser 对象
4. 然后通过 Browser 对象创建页面 Page 对象
5. 然后 page.goto() 跳转到指定的页面
6. 调用 page.screenshot() 对页面进行截图
7. 关闭浏览器
  Puppeteer的常用属性/方法/对象
  launch(options)方法
  介绍
  使用launch 方法会返回一个browser对象, 创建browser对象的时候可以选择性的配置如下选项
  常用参数

Browser对象
介绍
Browser对象相当于selenium的driver对象, 常用的有如下方法
当返回值是promise的时候, 在python中返回的是异步函数, 需要这样的函数需要使用await关键字调用
常用方法

browser.userAgent()    设置browser的请求头
Page对象
介绍
Page对象相当于selenium的每一个标签页
我们对页面的元素进行操作就是操作page对象
常用方法
常用方法
page.content()     获取页面的源码
page.cookies(...urls)      获取当前的cookies
page.goto()     请求某个页面
page.click(selector[, options])     点击元素
page.focus(selector)    聚焦某元素
page.close(options)     关闭当前标签页
page.url()   当前页面的url
page.reload(options)     重新加载页面
page.type(selector, text[, options])    向聚焦的元素框中输入内容
获取元素
Page.quirySelector()      选择一个元素
Page.querySelectorAll()     选择一组元素
Page.xpath()     通过xpath选择元素
evaluate()方法(运行js脚本/获取元素的属性)
evaluate接收字符串为参数， 字符串的内容可以使javascript的原生函数， 或者是表达式， 当字符串的内容是表达式的时候我们通常可以用来获取元素的属性， 当字符串的内容是js函数的时候通常可以用来在当前页面执行js代码
当字符串的内容是表达式的时候可以用force_expr属性来强调, 不然可能会识别失败报错
示例：

修改driver的配置
1. Page.setViewport() 修改浏览器视窗大小
2. Page.setUserAgent() 设置浏览器的 UserAgent 信息
3. Page.emulateMedia() 更改页面的CSS媒体类型，用于进行模拟媒体仿真。 可选值为 “screen”, “print”, “null”, 如果设置为 null 则表示禁用媒体仿真。
  键盘keyboard
  keyboard.down(key[, options]) 触发 keydown 事件
  keyboard.press(key[, options]) 按下某个键，key 表示键的名称，比如 ‘ArrowLeft’ 向左键，详细的键名映射请戳这里
  keyboard.sendCharacter(char) 输入一个字符
  keyboard.type(text, options) 输入一个字符串
  keyboard.up(key) 触发 keyup 事件
  鼠标mouse
  mouse.click(x, y, [options]) 移动鼠标指针到指定的位置，然后按下鼠标，这个其实 mouse.move 和 mouse.down 或 mouse.up 的快捷操作
  mouse.down([options]) 触发 mousedown 事件，options 可配置:
  options.button 按下了哪个键，可选值为[left, right, middle], 默认是 left, 表示鼠标左键
  options.clickCount 按下的次数，单击，双击或者其他次数
  delay 按键延时时间
  mouse.move(x, y, [options]) 移动鼠标到指定位置， options.steps 表示移动的步长
  mouse.up([options]) 触发 mouseup 事件
  waitFor()等待方法
  page.waitFor(selectorOrFunctionOrTimeout[, options[, …args]]) 下面三个的综合 API
  page.waitForFunction(pageFunction[, options[, …args]]) 等待 pageFunction 执行完成之后
  page.waitForNavigation(options) 等待页面基本元素加载完之后，比如同步的 HTML, CSS, JS 等代码
  page.waitForSelector(selector[, options]) 等待某个选择器的元素加载之后，这个元素可以是异步加载的，这个 API 非常有用，你懂的。
  事件的监听
  使用方法
  Page.on("event", callback)
  event: 事件名称
  callback: 监听到事件后执行的回调函数
  所有可以监听的事件


官方文档介绍的Python使用Puppeteer的区别
Keyword arguments for options
Puppeteer uses object (dictionary in python) for passing options to functions/methods. Pyppeteer accepts both dictionary and keyword arguments for options.
Dictionary style option (similar to puppeteer):
browser = await launch({'headless': True})
Keyword argument style option (more pythonic, isn't it?):
browser = await launch(headless=True)
Element selector method name ($ -> querySelector)
In python, $ is not usable for method name. So pyppeteer usesPage.querySelector()/Page.querySelectorAll()/Page.xpath() instead of Page.$()/Page.$$()/Page.$x(). Pyppeteer also has shorthands for these methods, Page.J(), Page.JJ(), and Page.Jx().
Arguments of Page.evaluate() and Page.querySelectorEval()
Puppeteer's version of evaluate() takes JavaScript raw function or string of JavaScript expression, but pyppeteer takes string of JavaScript. JavaScript strings can be function or expression. Pyppeteer tries to automatically detect the string is function or expression, but sometimes it fails. If expression string is treated as function and error is raised, add force_expr=True option, which force pyppeteer to treat the string as expression.
Example to get page content:
content = await page.evaluate('document.body.textContent', force_expr=True)
Example to get element's inner text:
element = await page.querySelector('h1')
title = await page.evaluate('(element) => element.textContent', element)
python实现puppeteer同步使用库
github  ：  https://github.com/issacLiuUp/puppeteer