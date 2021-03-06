---
title: Android逆向-动态分析破解Apk(破解加固)
date: "2019-03-01 00:01:00"
categories:
- python爬虫
- 爬虫拓展功能
tags:
- 爬虫
- Android逆向
toc: true
typora-root-url: ..\..\..
---

转载: http://www.520monkey.com/

## **一、前言**

介绍一下如何应对现在市场中一些加固的apk的破解之道，现在市场中加固apk的方式一般就是两种：一种是对源apk整体做一个加固，放到指定位置，运行的时候在解密动态加载，还有一种是对so进行加固，在so加载内存的时候进行解密释放。我们今天主要看第一种加固方式，就是对apk整体进行加固。

## **二、案例分析**

按照国际惯例，咋们还是得用一个案例来分析讲解，这次依然采用的是阿里的CTF比赛的第三题：

![1551411964089](/img/1551411964089.png)

题目是：**要求输入一个网页的url，然后会跳转到这个页面，但是必须要求弹出指定内容的Toast提示，这个内容是：祥龙！**

了解到题目，我们就来简单分析一下，这里大致的逻辑应该是，输入的url会传递给一个WebView控件，进行展示网页，如果按照题目的逻辑的话，应该是网页中的Js会调用本地的一个Java方法，然后弹出相应的提示，那么这里我们就来开始操作了。

按照我们之前的破解步骤：

**第一步：肯定是先用解压软件搞出来他的classes.dex文件，然后使用dex2jar+jd-gui进行查看java代码**

![1551411972337](/img/1551411972337.png)

擦，这里我们看到这里只有一个Application类，从这里我们可以看到，这个apk可能被加固了，为什么这么说呢？因为我们知道一个apk加固，外面肯定得套一个壳，这个壳必须是自定义的Application类，因为他需要做一些初始化操作，那么一般现在加固的apk的壳的Application类都喜欢叫StubApplication。而且，这里我们可以看到，除了一个Application类，没有其他任何类了，包括我们的如可Activity类都没有了，那么这时候会发现，很蛋疼，无处下手了。

**第二步：我们会使用apktool工具进行apk的反编译，得到apk的AndroidManifest.xml和资源内容**

![1551411980295](/img/1551411980295.png)

反编译之后，看到程序会有一个入口的Activity就是MainActivity类，我们记住一点就是，不管最后的apk如何加固，即使我们看不到代码中的四大组件的定义，但是肯定会在AndroidManifest.xml中声明的，因为如果不声明的话，运行是会报错的。那么这里我们也分析完了该分析的内容，还是没发现我们的入口Activity类，而且我们知道他肯定是放在本地的一个地方，因为需要解密动态加载，所以不可能是放在网上的，肯定是本地，所以这里就有一些技巧了：

当我们发现apk中主要的类都没有了，肯定是apk被加固了，加固的源程序肯定是在本地，一般会有这么几个地方需要注意的：

**1、应用程序的asset目录，我们知道这个目录是不参与apk的资源编译过程的，所以很多加固的应用喜欢把加密之后的源apk放到这里**

**2、把源apk加密放到壳的dex文件的尾部，这个肯定不是我们这里的案例，但是也有这样的加固方式，这种加固方式会发现使用dex2jar工具解析dex是失败的，我们这时候就知道了，肯定对dex做了手脚**

**3、把源apk加密放到so文件中，这个就比较难了，一般都是把源apk进行拆分，存到so文件中，分析难度会加大的。**

一般都是这三个地方，其实我们知道记住一点：就是不管源apk被拆分，被加密了，被放到哪了，只要是在本地，我们都有办法得到他的。

好了，按照这上面的三个思路我们来分析一下，这个apk中加固的源apk放在哪了？通过刚刚的dex文件分析，发现第二种方式肯定不可能了，那么会放在asset目录中吗？我们查看asset目录：

![1551411990583](/img/1551411990583.png)

看到asset目录中的确有两个jar文件，而且我们第一反应是使用jd-gui来查看jar，可惜的是打开失败，所以猜想这个jar是经过处理了，应该是加密，所以这里很有可能是存放源apk的地方。但是我们上面也说了还有第三种方式，我们去看看libs目录中的so文件：

![1551411997754](/img/1551411997754.png)**

擦，这里有三个so文件，而我们上面的Application中加载的只有一个so文件：libmobisec.so，那么其他的两个so文件很有可能是拆分的apk文件的藏身之处。

通过上面的分析之后，我们大致知道了两个地方很有可能是源apk的藏身地方，一个是asset目录，一个是libs目录，那么分析完了之后，我们发现现在面临两个问题：

**第一个问题：asset目录中的jar文件被处理了，打不开，也不知道处理逻辑**

**第二个问题：libs目录中的三个so文件，唯一加载了libmobisec.so文件了**

那么这里现在的唯一入口就是这个libmobisec.so文件了，因为上层的代码没有，没法分析，下面来看一下so文件：

![1551412011735](/img/1551412011735.png)

擦，发现蛋疼的是，这里没有特殊的方法，比如Java_开头的什么，所以猜测这里应该是自己注册了native方法，混淆了native方法名称，那么到这里，我们会发现我们遇到的问题用现阶段的技术是没法解决了。

## **三、获取正确的dex内容**

分析完上面的破解流程之后，发现现在首要的任务是先得到源apk程序，通过分析知道，处理的源apk程序很难找到和分析，所以这里就要引出今天说的内容了，使用动态调试，给libdvm.so中的函数：dvmDexFileOpenPartial 下断点，然后得到dex文件在内存中的起始地址和大小，然后dump处dex数据即可。

那么这里就有几个问题了：

**第一个问题：为何要给dvmDexFileOpenPartial 这个函数下断点？**

因为我们知道，不管之前的源程序如何加固，放到哪了，最终都是需要被加载到内存中，然后运行的，而且是没有加密的内容，那么我们只要找到这的dex的内存位置，把这部分数据搞出来就可以了，管他之前是如何加固的，我们并不关心。那么问题就变成了，如何获取加载到内存中的dex的地址和大小，这个就要用到这个函数了：dvmDexFileOpenPartial 因为这个函数是最终分析dex文件，加载到内存中的函数：

int dvmDexFileOpenPartial(const void* addr, int len, DvmDex** ppDvmDex);

第一个参数就是dex内存起始地址，第二个参数就是dex大小。

**第二个问题：如何使用IDA给这个函数下断点**

我们在之前的一篇文章中说到了，在动态调试so，下断点的时候，必须知道一个函数在内存中的绝对地址，而函数的绝对地址是：这个函数在so文件中的相对地址+so文件映射到内存中的基地址，这里我们知道这个函数肯定是存在libdvm.so文件中的，因为一般涉及到dvm有关的函数功能都是存在这个so文件中的，那么我们可以从这个so文件中找到这个函数的相对地址，运行程序之后，在找到libdvm.so的基地址，相加即可，那么我们如何获取到这个libdvm.so文件呢？这个文件是存放在设备的/system/lib目录下的：

![1551412020437](/img/1551412020437.png)

那么我们只需要使用adb pull 把这个so文件搞出来就可以了。

好了，解决了这两个问题，下面就开始操作了：

**第一步：运行设备中的android_server命令，使用adb forward进行端口转发**

![1551412032149](/img/1551412032149.png)

这里的android_server工具可以去ida安装目录中dbgsrv文件夹中找到

![1551412042421](/img/1551412042421.png)

**第二步：使用命令以debug模式启动apk**

**adb shell am start -D -n com.ali.tg.testapp/.MainActivity**

![1551412051232](/img/1551412051232.png)

因为我们需要给libdvm.so下断点，这个库是系统库，所以加载时间很早，所以我们需要像之前给JNI_OnLoad函数下断点一样，采用debugger模式运行程序，这里我们通过上面的AndroidManifest.xml中，得到应用的包名和入口Activity：

![1551412058854](/img/1551412058854.png)

而且这里的android:debuggable=true，可以进行debug调试的。

**第三步：双开IDA，一个用于静态分析libdvm.so，一个用于动态调试libdvm.so**

![1551412066303](/img/1551412066303.png)

通过IDA的Debugger菜单，进行进程附加操作：

![1551412075783](/img/1551412075783.png)



**第四步：使用jdb命令启动连接attach调试器**

```
jdb -connect com.sun.jdi.SocketAttach:hostname=127.0.0.1,port=8700
```

但是这里可能会出现这样的错误：

![1551412082688](/img/1551412082688.png)

这个是因为，我们的8700端口没有指定，这时候我们可以通过Eclipse的DDMS进行端口的查看：

![1551412091795](/img/1551412091795.png)

看到了，这里是8600端口，但是基本端口8700不在，所以这里我们有两种处理方式，一种是把上面的命令的端口改成8600，还有一种是选中这个应用，使其具有8700端口：

![1551412100337](/img/1551412100337.png)

点击这个条目即可，这时候我们在运行上面的jdb命令：

![1551412107970](/img/1551412107970.png)

处于等待状态。

**第四步：给dvmDexFileOpenPartial函数下断点**

使用一个IDA静态分析得到这个函数的相对地址：43308

![1551412119327](/img/1551412119327.png)

在动态调试的IDA解密，使用Ctrl+S键找到libdvm.so的在内存中的基地址：41579000

![1551412126706](/img/1551412126706.png)

然后将两者相加得到绝对地址：43308+41579000=415BC308，使用G键，跳转：

![1551412134873](/img/1551412134873.png)

跳转到dvmDexFileOpenPartial函数处，下断点：

![1551412141684](/img/1551412141684.png)



**第五步：点击运行按钮或者F9运行程序**

之前的jdb命令就连接上了：

![1551412152284](/img/1551412152284.png)

IDA出现如下界面，不要理会，一路点击取消按钮即可

![1551412160530](/img/1551412160530.png)

运行到了dvmDexFileOpenPartial函数处：

![1551412168174](/img/1551412168174.png)

使用F8进行单步调试，但是这里需要注意的是，只要运行过了PUSH命令就可以了，记得不要越过下面的BL命令，因为我们没必要走到那里，当执行了PUSH命令之后，我们就是使用脚本来dump处内存中的dex数据了，这里有一个知识点，就是R0~R4寄存器一般是用来存放一个函数的参数值的，那么我们知道dvmDexFileOpenPartial函数的第一个参数就是dex内存起始地址，第二个参数就是dex大小：

![1551412175294](/img/1551412175294.png)

那么这里就可以使用这样的脚本进行dump即可：

**static main(void){    auto fp, dex_addr, end_addr;    fp = fopen(“F:\\dump.dex”, “wb”);    end_addr = r0 + r1;    for ( dex_addr = r0; dex_addr < end_addr; dex_addr ++ )        fputc(Byte(dex_addr), fp);}**

脚本不解释了，非常简单，而且这个是固定的格式，以后dump内存中的dex都是这段代码，我们将dump出来的dex保存到F盘中。

然后这时候，我们使用：Shirt+F2 调出IDA的脚本运行界面：

![1551412182305](/img/1551412182305.png)

点击运行，这里可能需要等一会，运行成功之后，我们去F盘得到dump.dex文件，其实这里我们的IDA使命就完成了，因为我们得到了内存的dex文件了，下面开始就简单了，只要分析dex文件即可

## **四、分析正确的dex文件内容**

我们拿到dump.dex之后，使用dex2jar工具进行反编译：

![1551412188951](/img/1551412188951.png)

可惜的是，报错了，反编译失败，主要是有一个类导致的，开始我以为是dump出来的dex文件有问题，最后我用baksmali工具得到smali文件是可以的，所以不是dump出来的问题，我最后用baksmali工具将dex转化成smali源码：

j

```
ava -jar baksmali-2.0.3.jar -o C:\classout/ dump.dex
```

![1551412199808](/img/1551412199808.png)

得到的smali源码目录classout在C盘中：

![1551412207234](/img/1551412207234.png)

我们得到了指定的smali源码了。

那么下面我们就可以使用静态方式分析smali即可了：

首先找到入口的MainActivity源码：

![1551412215225](/img/1551412215225.png)

这里不解释了，肯定是找按钮的点击事件代码处，这里是一个btn_listener变量，看这个变量的定义：

![1551412224196](/img/1551412224196.png)

是MainActivity$1内部类定义，查看这个类的smali源码，直接查看他的onClick方法：

![1551412231725](/img/1551412231725.png)

这里可以看到，把EditText中的内容，用Intent传递给WebViewActivity中，但是这里的intent数据的key是加密的。

下面继续看WebViewActivity这个类：

![1551412242134](/img/1551412242134.png)

我们直接查找onCreate方法即可，看到这里是初始化WebView，然后进行一些设置，这里我们看到一个@JavascriptInterface

这个注解，我们在使用WebView的时候都知道，他是用于Js中能够访问的设置了这个注解的方法，没有这个注解的方法Js是访问不了的

**注意：**

我们知道这个注解是在SDK17加上的，也就是Android4.2版本中，那么在之前的版本中没有这个注解，任何public的方法都可以在JS代码中访问，而Java对象继承关系会导致很多public的方法都可以在JS中访问，其中一个重要的方法就是  getClass()。然后JS可以通过反射来访问其他一些内容。那么这里就有这个问题了：比如下面的一段JS代码：

<script>
	function findobj(){
		for (var obj in window) { 
			if ("getClass" in window[obj]) { 
				return window[obj] 
			} 
		} 
	}
</script>

看到了，这段js代码很危险的，使用getClass方法，得到这个对象(java中的每个对象都有这个方法的)，用这个方法可以得到一个java对象，然后我们就可以调用这个对象中的方法了。这个也算是WebView的一个漏洞了。

所以通过引入 @JavascriptInterface注解，则在JS中只能访问 @JavascriptInterface注解的函数。这样就可以增强安全性。

回归到正题，我们上面分析了smali源码，看到了WebView的一些设置信息，我们可以继续往下面看：

![1551412252134](/img/1551412252134.png)

这里的我们看到了一些重要的方法，一个是addJavascriptInterface，一个是loadUrl方法。

我们知道addjavaascriptInterface方法一般的用法：

mWebView.addJavascriptInterface(new JavaScriptObject(this), "jiangwei");

第一个参数是本地的Java对象，第二个参数是给Js中使用的对象的名称。然后js得到这个对象的名称就可以调用本地的Java对象中的方法了。

看了这里的addjavaascriptInterface方法代码，可以看到，这里用

ListViewAutoScrollHelpern;->decrypt_native(Ljava/lang/String;I)Ljava/lang/String;

将js中的名称进行混淆加密了，这个也是为了防止恶意的网站来拦截url，然后调用我们本地的Java中的方法。

**注意：**

这里又存在一个关于WebView的安全问题，就是这里的js访问的对象的名称问题，比如现在我的程序中有一个Js交互的类，类中有一个获取设备重要信息的方法，比如这里获取设备的imei方法，如果我们的程序没有做这样名称的混淆的话，破解者得到这个js名称和方法名，然后就伪造一个恶意url，来调用我们程序中的这个方法，比如这样一个例子：

![1551412261132](/img/1551412261132.png)

然后在设置js名称：

![1551412270357](/img/1551412270357.png)

我们就可以伪造一个恶意的url页面来访问这个方法，比如这个恶意的页面代码如下：

![1551412279648](/img/1551412279648.png)

运行程序：

![1551412287311](/img/1551412287311.png)

看到了，这里恶意的页面就成功的调用了程序中的一个重要方法。

所以，我们可以看到，对Js交互中的对象名称做混淆是必要的，特别是本地一些重要的方法。

回归到正题，我们分析完了WebView的一些初始化和设置代码，而且我们知道如果要被Js访问的方法，那么必须要有@JavascriptInterface注解 因为在Java中注解也是一个类，所以我们去注解类的源码看看那个被Js调用的方法：

![1551412295016](/img/1551412295016.png)

这里看到了有一个showToast方法，展示的内容：\u7965\u9f99\uff01 ，我们在线转化一下：

![1551412304306](/img/1551412304306.png)

擦，这里就是题目要求展示的内容。

好了，到这里我们就分析完了apk的逻辑了，下面我们来整理一下：

1、在MainActivity中输入一个页面的url，跳转到WebViewActivity进行展示

2、WebViewActivity有Js交互，需要调用本地Java对象中的showToast方法展示消息

问题：

因为这里的js对象名称进行了加密，所以这里我们自己编写一个网页，但是不知道这个js对象名称，无法完成showToast方法的调用

## **五、破解的方法**

下面我们就来分析一下如何解决上面的问题，其实解决这个问题，我们现有的方法太多了

**第一种方法：**修改smali源码，把上面的那个js对象名称改成我们自己想要的，比如：jiangwei，然后在自己编写的页面中直接调用：jiangwei.showToast方法即可，不过这里需要修改smali源码，在使用smali工具回编译成dex文件，在弄到apk中，在运行。方法是可行的，但是感觉太复杂，这里不采用

**第二种方法：**利用Android4.2中的WebView的漏洞，直接使用如下Js代码即可

![1551412314129](/img/1551412314129.png)

这里根本不需要任何js对象的名称，只需要方法名就可以完成调用，所以这里可以看到这个漏洞还是很危险的。

**第三种方法：**我们看到了那个加密方法，我们自己写一个程序，来调用这个方法，尽然得到正确的js对象名称，这里我们就采用这种方式，因为这个方式有一个新的技能，所以这里我就讲解一下了。

那么如果用第三种方法的话，就需要再去分析那个加密方法逻辑了：

![1551412323606](/img/1551412323606.png)

android.support.v4.widget.ListViewAutoScrollHelpern在这个类中，我们再去查找这个smali源码：

![1551412332017](/img/1551412332017.png)

这个类加载了libtranslate.so库，而且加密方法是native层的，那么我们用IDA查看libtranslate.so库：

![1551412341893](/img/1551412341893.png)

我们搜一下Java开头的函数，发现并没有和decrypt_native方法对应的native函数，说明这里做了native方法的注册混淆，我们直接看JNI_OnLoad函数：

![1551412350326](/img/1551412350326.png)

这里果然是自己注册了native函数，但是分析到这里，我就不往下分析了，为什么呢？因为我们其实没必要搞清楚native层的函数功能，我们知道了Java层的native方法定义，那么我们可以自己定义一个这么个native方法来调用libtranslate.so中的加密函数功能：

![1551412360214](/img/1551412360214.png)

我们新建一个Demo工程，仿造一个ListViewAutoScrollHelpern类，内部在定义一个native方法：

![1551412369218](/img/1551412369218.png)

然后我们在MainActivity中加载libtranslate.so：

![1551412377849](/img/1551412377849.png)

然后调用那个native方法，打印结果：

![1551412386861](/img/1551412386861.png)

这里的方法的参数可以查看smali源码中的那个方法参数：

![1551412394814](/img/1551412394814.png)

点击运行，发现有崩溃的，我们查看log信息：

![1551412404048](/img/1551412404048.png)

是libtranslate.so中有一个PagerTitleStripIcsn类找不到，这个类应该也有一个native方法，我们在构造这个类：

![1551412413355](/img/1551412413355.png)

再次运行，还是报错，原因差不多，还需要在构造一个类：TaskStackBuilderJellybeann

![1551412424135](/img/1551412424135.png)

好了，再次点击运行：

![1551412433726](/img/1551412433726.png)

OK了，成功了，从这个log信息可以看出来了，解密之后的js对象名称是：SmokeyBear，那么下面就简单了，我们在构造一个url页面，直接调用：SmokeyBear.showToast即可。

**注意：**

这里我们看到，如果知道了Java层的native方法的定义，那么我们就可以调用这个native方法来获取native层的函数功能了，这个还是很不安全的，但是我们如何防止自己的so被别人调用呢？可以在so中的native函数做一个应用的签名校验，只有属于自己的签名应用才能调用，否则直接退出。

## **六，开始测试**

上面已经知道了js的对象名称，下面我们就来构造这个页面了：

![1551412441417](/img/1551412441417.png)

那么这里又有一个问题了，这个页面构造好了？放哪呢？有的同学说我有服务器，放到服务器上，然后输入url地址就可以了，的确这个方法是可以的，但是有的同学没有服务器怎么办呢？这个也是有方法的，我们知道WebView的loadUrl方法是可以加载本地的页面的，所以我们可以把这个页面保存到本地，但是需要注意的是，这里不能存到SD卡中，因为这个应用没有读取SD的权限，我们可以查看他的AndroidManifest.xml文件：

![1551412449805](/img/1551412449805.png)

我们在不重新打包的情况下，是没办法做到的，那么放哪呢？其实很简单了，放在这个应用的/data/data/com.ali.tg.testapp/目录下即可，因为除了SD卡位置，这个位置是最好的了，那么我们知道WebView的loadUrl方法在加载本地的页面的格式是：

**file:///data/data/com.ali.tg.testapp/crack.html**

那么我们直接输入即可

**注意：**

这里在说一个小技巧：就是我们在一个文本框中输入这么多内容，是不是有点蛋疼，我们其实可以借助于命令来实现输入的，就是使用：adb shell input text ”我们需要输入的内容“。

具体用法很简单，打开我们需要输入内容的EditText，点击调出系统的输入法界面，然后执行上面的命令即可：

![1551412458167](/img/1551412458167.png)

不过这里有一个小问题，就是他不识别分号：

![1551412466571](/img/1551412466571.png)

不过我们直接修改成分号点击进入：![1551412480302](/img/1551412480302.png)

运行成功，看到了toast的展示。

## **七、内容整理**

到这里我们就破解成功了，下面来看看整理一下我们的破解步骤：

**1、破解的常规套路**

我们按照破解惯例，首先解压出classses.dex文件，使用dex2jar工具查看java代码，但是发现只有一个Application类，所以猜测apk被加壳了，然后用apktool来反编译apk，得到他的资源文件和AndroidManifest.xml内容，找到了包名和入口的Activity类。

**2、加固apk的源程序一般存放的位置**

知道是加固apk了，那么我们就分析，这个加固的apk肯定是存放在本地的一个地方，一般是三个地方：

1》应用的asset目录中

2》应用的libs中的so文件中

3》应用的dex文件的末尾

我们分析了一下之后，发现asset目录中的确有两个jar文件，但是打不开，猜测是被经过处理了，所以我们得分析处理逻辑，但是这时候我们也没有代码，怎么分析呢？所以这时候就需要借助于dump内存dex技术了：

不管最后的源apk放在哪里，最后都是需要经历解密动态加载到内存中的，所以分析底层加载dex源码，知道有一个函数：dvmDexFileOpenPartial 这个函数有两个重要参数，一个是dex的其实地址，一个是dex的大小，而且知道这个函数是在libdvm.so中的。所以我们可以使用IDA进行动态调试获取信息

**3、双开IDA开始获取内存中的dex内容**

双开IDA，走之前的动态破解so方式来给dvmDexFileOpenPartial函数下断点，获取两个参数的值，然后使用一段脚本，将内存中的dex数据保存到本地磁盘中。

**4、分析获取到的dex内容**

得到了内存中的dex之后，我们在使用dex2jar工具去查看源码，但是发现保存，以为是dump出来的dex格式有问题，但是最后使用baksmali工具进行处理，得到smali源码是可以的，然后我们就开始分析smali源码。

**5、分析源码了解破解思路**

通过分析源码得知在WebViewActivity页面中会加载一个页面，然后那个页面中的js会调用本地的Java对象中的一个方法来展示toast信息，但是这里我们遇到了个问题：Js的Java对象名称被混淆加密了，所以这时候我们需要去分析那个加密函数，但是这个加密函数是native的，然后我们就是用IDA去静态分析了这个native函数，但是没有分析完成，因为我们不需要，其实很简单，我们只需要结果，不需要过程，现在解密的内容我们知道了，native方法的定义也知道了，那么我们就去写一个简单的demo去调用这个so的native方法即可，结果成功了，我们得到了正确的Js对象名称。

**6、了解WebView的安全性**

WebView的早期版本的一个漏洞信息，在Android4.2之前的版本WebView有一个漏洞，就是可以执行Java对象中所有的public方法，那么在js中就可以这么处理了，先获取geClass方法获取这个对象，然后在调用这个对象中的一些特定方法即可，因为Java中所有的对象都有一个getClass方法，而这个方法是public的，同时能够返回当前对象。所以在Android4.2之后有了一个注解：

@JavascriptInterface ，只有这个注解标识的方法才能被Js中调用。

**7、获取输入的新技能**

验证结果的过程中我们发现了一个技巧，就是我们在输入很长的文本的时候，比较繁琐，可以借助adb shell input text命令来实现。

## **八、技术点概要**

1、通过dump出内存中的dex数据，可以佛挡杀佛了，不管apk如何加固，最终都是需要加载到内存中的。

2、了解到了WebView的安全性的相关知识，比如我们在WebView中js对象名称做一次混淆还是有必要的，防止被恶意网站调用我们的本地隐私方法。

3、可以尝试调用so中的native方法，在知道了这个方法的定义之后

4、adb shell input text 命令来辅助我们的输入

## **九、总结**

这里就介绍了Android中如何dump出那些加固的apk程序，其实核心就一个：不管上层怎么加固，最终加载到内存的dex肯定不是加固的，所以这个dex就是我们想要的，这里使用了IDA来动态调试libdvm.so中的dvmDexFileOpenPartial函数来获取内存中的dex内容，同时还可以使用gdb+gdbserver来获取，这个感兴趣的同学自行搜索吧。结合了之前的两篇文章，就算善始善终，介绍了Android中大体的破解方式，当然这三种方式不是万能的，因为加固和破解是相生相克的，没有哪个有绝对的优势，只是两者相互进步罢了，当然还有很多其他的破解方式，后面如果遇到的话，会在详细说明。
