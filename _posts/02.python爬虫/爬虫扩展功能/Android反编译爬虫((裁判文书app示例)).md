---

title: Android反编译尝试(裁判文书app示例)
date: "2019-02-27 00:01:00"
categories:
- python爬虫
- 爬虫拓展功能
tags:
- 爬虫
- Android反编译
toc: true
typora-root-url: ..\..\..
---

## 一、工具准备

- fiddler
- 安卓模拟器
- android studio
- enjarify
- jd-gui
- python3
- jdk
- apktool
- adb
- SignApk.jar apk签名工具
- IDA

## 二、 抓包查看apk的请求逻辑

#### 1）设置fiddler

下载fiddler后，打开菜单栏 Tools=》Options=》HTTPS， 勾选【Decrypt HTTPS traffic】选项，对于【Ignore server certificate errors (unsafe)】选项可以不必勾选，然后点击【Actions】点击【Export Root Certificate to Desktop】这时候就会将Fiddler根证书FiddlerRoot.cer保存到桌面上，这个根证书在如果开启了Fiddler的HTTPS解密的时候火狐浏览器访问HTTPS地址时候出现【您的连接并不安全】的错误页面时候使用。

![1551354228029](/img/1551354228029.png)

然后点击HTTPS标签栏旁边的Connections标签， 这里我们要记得【Fiddler listen on port】中显示的端口号（关于这个端口号，如果当前默认的8888端口号已经被占用了，那么需要重新设置另外的端口号），然后将【Allow remote computers to connect】前面的勾打上。点击确定，然后重新启动Fiddler。
重新启动后，打开Fiddler后，在Fiddler界面的右上角的三角形上点击就会显示一个【Online】图标，把鼠标放到【Online】图标上，会显示当前机器的IP地址， 该地址直接用来和后面安卓设备的链接

![1551354288639](/img/1551354288639.png)

#### 2） 设置安卓设备

不同的安卓设备会有一些区别，但是设置的思路一样

点击桌面上的【系统应用】=》【设置】=》【WLAN】， 鼠标放到当前已经连接的网络上长按， 在弹出的消息窗口中点击【修改网络】，输入上面我们得到的IP地址和端口号，点击保存。

![1551354455210](/img/1551354455210.png)

然后在模拟器中打开浏览器，输入：http://ipv4.fiddler:8888 ，出现下面的页面说明我们刚刚设置的http代理正确，然后点击红线框的【FiddlerRoot certificate】，下载Fiddler的根证书：

![1551354492354](/img/1551354492354.png)

然后我们来到桌面【系统应用】=》【设置】=》【安全】=》【从SD卡安装】中找到我们刚刚下载的证书：

![1551354516192](/img/1551354516192.png)

点击证书，然后输入证书名称点击【确定】

![1551354534913](/img/1551354534913.png)

安卓设备设置完成，打开浏览器可以检验一下fiddler中是否可以检测到网络请求，如果检测到说明配置成功

#### 3）操作apk并进行请求和抓包

**抓包目的**

下载apk，并进行相应的请求，查找所需数据的网络请求包和必要的请求参数、以及数据书否被加密等

**找到数据包**

以裁判文书Apk为例， 找到按关键字搜索文书的数据结果

Apk主页

![1551432342503](/img/1551432342503.png)

在搜索框中输入关键字 "阿里", 点击搜索按钮，在fiddler中找到包含数据的请求

第一个主要的请求主要是从服务端获取reqtoken，该参数作为后面请求的表单参数

![1551434558532](/img/1551434558532.png)

第二个主要的请求，便是从服务端获取搜索的结果

![1551434696566](/img/1551434696566.png)

点击列表页的item，获取详情内容，抓取数据包；

可以确定获取详情内容的数据包为红色框内请求，需携带详情的fieldId;

![1551667457491](/img/1551667457491.png)

**确定破解目标**

综上分析，从抓包上来看，在请求中不确定的参数主要有，headers中的devid，signature，nonce， timespan参数，并且第二次对列表页的请求的响应数据并不是明文，也就是加密后的结果，所以只有经过解密后才能获得真正的结果

接下来将要对其apk代码进行分析和debug调试，需要完成以下两个任务：

1.获取请求参数的生成方式，devid，signature，nonce, timespan

2.详情页的fieIdId生成方式；

3.列表页和详情页响应内容的解密方式

## 三、反编译获取Java代码

关于apk获取静态java代码以及分析请参考《Android逆向-静态分析破解Apk》

#### 1）配置环境，apk下载

使用enjarify进行反编译， 需要提前配置好python3运行环境

配置好python3开发环境后，下载将要进行反编译的apk

#### 2）运行enjarify进行反编译

将下载好的apk导入到同enjarify同一级目录下

输入命令

```shell
python3 -O -m enjarify.main yourapp.apk
```

反编译效果， 反编译生成的文件就是 wenshuapp-enjarify.jar

![1551498638957](/img/1551498638957.png)

#### 3）JD-GUI查看java源码

这一步不用过多介绍，将反编译生成的jar文件导入进来就可以查看其源码部分

如图就是通过反编译工具获取的java  class 源代码， 开始寻找我们需要分析的代码

![1551498880234](/img/1551498880234.png)

可以看到大部分的逻辑源码都在红色区域内

![1551667639036](/img/1551667639036.png)

下面开始动态调试，通过和源码内容的结合来分析整个apk请求和解析数据的逻辑

> enjarify工具下载地址：<https://github.com/google/enjarify>
>
> jd-gui工具下载地址: http://jd.benow.ca/

## 四、制作debug模式Apk

关于debug模式的apk制作和smali的基础语法，和调试方法请参考《Android逆向-动态分析破解Apk(Debug)》

#### 1）使用apktool来破解apk

```shell
java -jar apktool/apktool.jar d wenshu/wenshuapp.apk -o wenshu_out
```

![1551680174362](/img/1551680174362.png)

这里的命令不做解释了。

反编译成功之后，我们得到了一个out目录，如下：

![1551680233433](/img/1551680233433.png)

源码都放在smail文件夹中，我们进入查看一下文件：

![1551668524965](/img/1551668524965.png)

#### 2）修改AndroidManifest.xml中的debug属性和在入口代码中添加waitDebug

上面我们反编译成功了，下面我们为了后续的调试工作，所以还是需要做两件事：

**1》修改AndroidManifest.xml中的android:debuggable="true"**

![1551669525123](/img/1551669525123.png)

关于这个属性，我们前面介绍run-as命令的时候，也提到了，他标识这个应用是否是debug版本，这个将会影响到这个应用是否可以被调试，所以这里必须设置成true。

> 当然还有其他方式，比如aapt查看apk的内容方式，或者是安装apk之后用
>
> adb dumpsys activity top 命令查看都是可以的。

**2》在入口处添加waitForDebugger代码进行调试等待。**

这里说的入口处，就是程序启动的地方，就是我们一般的入口Activity，查找这个Activity的话，方法太多了，比如我们这里直接从上面得到的AndroidManifest.xml中找到，因为入口Activity的action和category是固定的, 如上图第二个红框。

找到入口Activity之后，我们直接在他的onCreate方法的第一行加上waitForDebugger代码即可，找到对应的MainActivity的smali源码，然后添加一行代码：

```java
invoke-static{}, Landroid/os/Debug;->waitForDebugger()V
```

这个是smali语法的，其实对应的Java代码就是：**android.os.Debug.waitForDebugger();**

![1551680457305](/img/1551680457305.png)

这里把Java语言翻译成smali语法的，网上有smali的语法解析。

#### 3）回编译apk并且进行签名安装

```shell
java -jar apktool.jar b out -o wenshu_debug.apk
```

还是使用apktool进行回编译

![1551680637021](/img/1551680637021.png)

编译完成之后，将得到debug.apk文件，但是这个apk是没有签名的，所以是不能安装的，那么下面我们需要在进行签名，这里我们使用Android中的测试程序的签名文件和sign.jar工具进行签名：

![1551409870379](/img/1551409870379.png)

```shell
java -jar signApk/signapk.jar ./signApk/testkey.x509.pem ./signApk/testkey.pk8 wenshu_debug.apk wenshu_debug.sig.apk
```

签名之后，我们就可以进行安装了。

## 五、进行调试

### 1) android studio配置

**下载 smalidea**

下载地址: https://bitbucket.org/JesusFreke/smali/downloads/

**安装smalidea**

打开AndroidStudio，点击Preferences... | Plugins, 选择Install plugin from disk

![1551683929897](/img/1551683929897.png)

### 2）将反编译的文件夹导入Android Studio

**选择Import Project**

  ![1551684206428](/img/1551684206428.png)

**选择Create preject from existing sources**

![1551684229133](/img/1551684229133.png)

一直选择“Next”，直至导入工程完成

然后在AndroidStudio工程中右键点击smali文件夹，设定Mark Directory as -> Sources Root

**AndroidStudio的File -> Project Structure, 配置JDK。**

![1551685126020](/img/1551685126020.png)

### 3）配置项目远程调试

**在AndroidStudio里面配置远程调试的选项，选择Run -> Edit Configurations， 并增加一个Remote调试的调试选项，端口选择:8700**

![1551685020495](/img/1551685020495.png)

### 4） 将进程和android studio之间建立连接

**启动apk**

启动模拟器上重新打包并安装好的apk，  apk将会卡在启动过程中等待接入debug

**查看进程端口信息**

找到apk进程的端口号

```shell
adb shell ps | grep com.lawyee.wenshuapp
```

![1551686293965](/img/1551686293965.png)

**建立进程和android studio之间的链接**

```shell
adb forward tcp:5005 jdwp:29685
```



### 5) 开始调试

**启动程序调试**

点击程序调试按钮, 控制台如果显示connect 成功，表示成功启动

![1551686666636](/img/1551686666636.png)

## 六、参数破解

代码都是混淆过后的，所以没什么办法，耐心的调试吧，找到想要破解的各个参数的生成位置， 找到生成参数的位置

### 获取请求参数生成方法

因为参数最终都是要经过网络请求发包，所以可以从网络请求发起处入手，然后反向追溯参数的生成过程

比如文书app的网络请求所在位置为com.lawyee.wenshuapp.util.a.a

可以通过调试找到参数生成的位置，再去生成的方法里面去看详细的生成逻辑就好，每个参数的生成

![1551775663911](/img/1551775663911.png)

**各参数对应的生成方法**

timespan 就是当前的datetime去掉所有符号格式

nonce生成方法

![1551865838331](/img/1551865838331.png)

devid生成方法

![1551865999458](/img/1551865999458.png)

signature生成方法

![1551866107208](/img/1551866107208.png)

### 获取列表\详情的解密方法

**获取解密方法**

找到解密响应数据的位置: 方法: com.lawyee.wenshuapp.util.aa.a， 用jd-gui查看一下该方法的源码 位置: 

粗略一看可以看出是通过AES进行的解密，但是方法接收了3个参数现在还不确定其生成方式，下面寻找三个参数的生成方法

![1551841278781](/img/1551841278781.png)

![1551865570153](/img/1551865570153.png)

**获取方法入参**

paramString1：通过ide，通过调试窗口可获得其为请求的响应值，即加密后待解密的内容

paramString2：为以下函数的返回值 com.lawuee.wenshuapp.vo.DevInfoVO.getIvParameter， 作为aes解密参数iv param；

![1551865640302](/img/1551865640302.png)

paramString3：为以下方法的返回值  com.lawyee.wenshuapp.config.ApplicationSet.a， 作为aes解密参数key， 接收发送请求的timespan参数为入参

![1551865687727](/img/1551865687727.png)

this实例属性

![1551842831298](/img/1551842831298.png)

更新this.f 属性的方法i，通过方法i 向this.f绑定20个生成key的方法；

![1551865433689](/img/1551865433689.png)

this.f是一个解密方法的集合，在初始化时更新this.f，其内容为20个获取key的方法，在解密的时候，会选择其中一个用来生成key，20个方法就不贴上了

在这20个生成key的方法中，其中用到了三个native方法，如下。

![1551939442758](/img/1551939442758.png)

这三个静态方法来自静态库 lienutil-lib, 下面需要从apktool反编译出来的文件中找到该静态文件，将其反编译成汇编，破解这三个方法 

![1551939470914](/img/1551939470914.png)

**分析so文件**

关于IDA的使用方法参考《Android逆向-动态分析破解Apk(调试so)》

用IDA打开  libenutil.so文件， 分别选中三个需要分析的方法

按F5键，将方法翻译成c代码， 然后再翻译成目标打码即可

![1551950318723](/img/1551950318723.png)

**StrToLong方法**

![1551950265245](/img/1551950265245.png)

**StrToLong2方法**

![1551950457850](/img/1551950457850.png)

**StrToLong3方法**

![1551950547874](/img/1551950547874.png)

目前所有的请求参数和解密过程中涉及的方法都找到了，下面可以实现破解了

## 七、破解代码生成

直接将上一步找到的参数生成方法，和解密方法等翻译成目标代码，python为例

**请求参数生成方法**

```python
import hashlib
import datetime
import random
import hashlib

# 生成signature
def get_signature(timespan, nonce, devid):
    string_buffer = [timespan, nonce, devid]
    string_buffer.sort()
    str_join = "".join(string_buffer)
    m = hashlib.md5()
    m.update(str_join.encode())
    return m.hexdigest()

# 生成nonce
def get_nonce():
    str_array = "abcdefghijklmnopqrstuvwxyz0123456789"
    nonce = ""
    for _ in range(4):
        index_choose = random.randint(0, len(str_array) - 1)
        nonce = nonce + str_array[index_choose]
    return nonce

# 生成devid
def get_uuid():
    return uuid.uuid4().replace("-", "")

# 生成timespan
def get_timespan():
    return datetime.datetime.now().strftime("%Y%m%d%H%M%S")
```

**解密数据方法**

```python
# coding=utf-8
from appcrack.f_methods import e, f, g, h, k, l, m, n, o, p, q, r, s, t, u, v, w, x, j  # 获取key参数的20的方法，暂不列举
from appcrack.f_methods import i as ii
from Crypto.Cipher import AES
import base64
import hashlib

# 3个native方法的python实现
def str2long(param_str, param_int):
    v10 = param_int
    v4 = 0
    v5 = param_str
    if not param_str.strip():
        return v4
    v6 = len(v5)
    v4 = v10
    if v6 >= 2:
        v7 = 1
        while v7 < v6:
            v4 += v10 + v7 + ord(v5[v7] % 256 << (v7 & 7)) - ord(v5[v7]) % 256
            v7 += 1
    return v4

def str2long2(param_str, param_int):
    v10 = param_int
    v4 = 0
    v5 = param_str
    if not param_str.strip():
        return v4
    v6 = len(v5)
    v4 = v10
    if v6 >= 2:
        v7 = 1
        while v7 < v6:
            v4 += v10 + v7 + ord(v5[v7] % 256 << (v7 & 15)) - ord(v5[v7]) % 256
            v7 += 1
    return v4

def str2long3(param_str, param_int):
    v10 = param_int
    v4 = 0
    v5 = param_str
    if not param_str.strip():
        return v4
    v6 = len(v5)
    v4 = v10
    if v6 >= 2:
        v7 = 1
        while v7 < v6:
            v4 += v10 + v7 + ord(v5[v7] % 256 << (v7 & 15)) + v7 % 4 - ord(v5[v7]) % 256
            v7 += 1
    return v4

# 数据解密iv param(paramString2)参数生成方法
def get_iv_param(devid):
    if len(devid.strip()) < 16:
        return None
    return devid[len(devid) - 16]

# key(paramString3)的生成方法
def get_key(timespan, token):
    key_producer_list = [e, f, q, r, s, t, u, v, w, x, g, h, ii, j, k, l, m, n, o, p]
    i = str2long(token, 1) % 20
    return key_producer_list[i](token + timespan)

# aes解密方法实现
def aes_decrypt(param_str1, param_str2, param_str3):
    block_size = AES.block_size
    pkc5Padding = lambda s: s + (block_size - len(s) % block_size) * chr(block_size - len(s) % block_size) \
        if (block_size - len(s) % block_size) else s + block_size * chr(16)
    unpkc5Padding = lambda s: s[:-ord(s[len(s) - 1:])]
    mode = AES.MODE_CBC
    localCipher = AES.new(key=param_str2.encode('ascii'), mode=mode, iv=param_str3.encode('utf8'))
    cryptedStr = base64.b64decode(param_str1)
    recovery = unpkc5Padding(localCipher.decrypt(cryptedStr))
    return recovery.decode('utf-8')

# 数据解密方法接口
def decrypt(devid, token, timespan, context):
    iv_param = get_iv_param(devid)
    key = get_key(timespan, token)
    return aes_decrypt(context, iv_param, key)
```

好了，目前app请求过程中的所有参数的生成方法，和响应数据的解密方法都已经构建好了，接下来就可以随意的编写爬虫进行数据爬取了。










