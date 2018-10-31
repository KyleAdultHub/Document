---
title: githubpage + hexo + yilia 搭建个人博客
date: "2018-10-17 19:23:11"
categories:
- 开发工具
- 博客
tags:
- 博客
toc: true
typora-root-url: ..\..
---

#### 0.本博客的由来

本来感觉写博客很费时间，但是最近感觉这两年攒手里的笔记太多了，不方便整理和分享

所以打算以后就干脆直接将笔记整理到github，这样也比自己维护一个博客省心

下面就将搭建本博客的步骤也同步上来，当做一个hello-world吧。

<!-- more -->
#### 1.环境准备

电脑环境是Windows，安装好git后，所有搭建操作均在git bash内完成

**1）安装hexo(首先要安装git, node.js, npm)**

注意：首次安装git 要配置user信息

```shell
$git config --global user.name "yourname"   #（yourname是git的用户名）

$git config --global user.email email）
```

**2）使用npm安装hexo**

```shell
 $npm install -g hexo
```

**3）创建hexo文件夹**

```shell
$mkdir hexo_blog
$cd hexo_lobg
```

**4）初始化框架**

```shell
$hexo init #hexo   #会自动创建网站所需要的文件

$npm install    #安装依赖包

$hexo generate 

$hexo server   #现在可以用127.0.0.1:4000访问hexo默认的hello world界面 ,hexo s = hexo server
```

#### 2.部署到github

**1）首次使用github需要配置密钥**

```shell
ssh-keygen -t rsa -C "email" 
```

生成ssh密钥，按三次回车键，密码为空,这边会生成id_rsa和_rsa.pub文件

打开id_rsa.pub，复制全文添加到GitHub 的Add SSH key中。

**2）创建Respository， 并开启githubPage**

首先注册登录github,然后创建页面仓库，Repository name 命名应该是 youname.github.io 

在setting界面， 配置

![1539839479905-1539839725571](/img/1539839479905-1539839725571.png)

**3）安装hexo-deployer-git**

```shell
$npm install hexo-deployer-git --save     用来推送项目到github
```

**4）生成博客，并push到github**

```shell
$hexo generate

$hexo deploy
```

**5）验证结果**

通过https://youname.github.io 进行访问

#### 3.更换博客模板

目前访问的博客模板比较简略，下面介绍使用：yilia模板

**1）拉取模板文件**

```shell
$git clone https://github.com/litten/hexo-theme-yilia.git themes/yilia
```

**2）更改配置文件修改模板为yilia**

打开项目目录下的_config.yml文件，更改主题theme;   `theme: yilia`
然后配置yilia文件下的_config.yml（目录：`hexo/themes/yilia/_config.yml`） 配置文件

```con
# Header
menu:
  主页: /
  归档: /archives
  #分类: /categories
  #标签: /tags

# SubNav
subnav:
  github: "https://github.com/KyleAdultHub"
  #weibo: "#"
  #rss: "#"
  #zhihu: "#"
  qq: "/information"
  #weixin: "#"
  #jianshu: "#"
  #douban: "#"
  #segmentfault: "#"
  #bilibili: "#"
  #acfun: "#"
  mail: "/information"
  #facebook: "#"
  #google: "#"
  #twitter: "#"
  #linkedin: "#"
  
  
rss: /atom.xml

# 是否需要修改 root 路径
# 如果您的网站存放在子目录中，例如 http://yoursite.com/blog，
# 请将您的 url 设为 http://yoursite.com/blog 并把 root 设为 /blog/。
root: /

# Content
# 文章太长，截断按钮文字
excerpt_link: more
# 文章卡片右下角常驻链接，不需要请设置为false
show_all_link: '展开全文'
# 数学公式
mathjax: false
# 是否在新窗口打开链接
open_in_new: false

# 打赏
# 打赏type设定：0-关闭打赏； 1-文章对应的md文件里有reward:true属性，才有打赏； 2-所有文章均有打赏
reward_type: 0
# 打赏wording
reward_wording: '谢谢你'
# 支付宝二维码图片地址，跟你设置头像的方式一样。比如：/assets/img/alipay.jpg
alipay: 
# 微信二维码图片地址
weixin: 

# 目录
# 目录设定：0-不显示目录； 1-文章对应的md文件里有toc:true属性，才有目录； 2-所有文章均显示目录
toc: 1
# 根据自己的习惯来设置，如果你的目录标题习惯有标号，置为true即可隐藏hexo重复的序号；否则置为false
toc_hide_index: true
# 目录为空时的提示
toc_empty_wording: '目录，不存在的…'

# 是否有快速回到顶部的按钮
top: true

# Miscellaneous
baidu_analytics: ''
google_analytics: ''
favicon: /favicon.png

#你的头像url
avatar: /img/header.jpg

#是否开启分享
share_jia: true

#评论：1、多说；2、网易云跟帖；3、畅言；4、Disqus；5、Gitment
#不需要使用某项，直接设置值为false，或注释掉
#具体请参考wiki：https://github.com/litten/hexo-theme-yilia/wiki/

#1、多说
duoshuo: false

#2、网易云跟帖
wangyiyun: false

#3、畅言
changyan_appid: *** #这个畅言id和conf写自己的
changyan_conf: ***

#4、Disqus 在hexo根目录的config里也有disqus_shortname字段，优先使用yilia的
disqus: false

#5、Gitment
gitment_owner: false      #你的 GitHub ID
gitment_repo: ''          #存储评论的 repo
gitment_oauth:
  client_id: ''           #client ID
  client_secret: ''       #client secret

# 样式定制 - 一般不需要修改，除非有很强的定制欲望…
style:
  # 头像上面的背景颜色
  header: '#4d4d4d'
  # 右滑板块背景
  slider: 'linear-gradient(200deg,#a0cfe4,#e8c37e)'

# slider的设置
slider:
  # 是否默认展开tags板块
  showTags: false

# 智能菜单
# 如不需要，将该对应项置为false
# 比如
#smart_menu:
#  friends: false
smart_menu:
  innerArchive: '所有文章'
  friends: '友链'
  aboutme: '关于我'

friends:
  #友情链接1: http://localhost:4000/
  
aboutme: 
  程序猿一枚<br>
```
