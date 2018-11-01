CloudFlare邮箱加密(cfemail)

爬虫与CloudFlare邮箱加密(cfemail)-反爬与反反爬的奇技淫巧
发布于

由于Javascript这块的内容很多，难度也不一样。所以这一篇文章希望给大家讲一些入门级别的。本来找来找去感觉都是高段位玩家，然而今天在做邮箱地址爬虫的过程中一段代码突然跳到了我的面前-邮箱地址加密JS。恰好难度适中，就今天拿出来跟大家一起聊一聊：


下面我们就进入正题，先来看看这个文章标题上的内容跟我们这篇文章到底有啥关系：
CloudFlare是一家美国的跨国科技企业，总部位于旧金山，在英国伦敦亦设有办事处。CloudFlare以向客户提供网站安全管理、性能优化及相关的技术支持为主要业务。通过基于反向代理的内容传递网络(ContentDeliveryNetwork,CDN)及分布式域名解析服务(DistributedDomainNameServer)，CloudFlare可以帮助受保护站点抵御包括拒绝服务攻击(DenialofService)在内的大多数网络攻击，确保该网站长期在线，同时提升网站的性能、访问速度以改善访客体验。
看完了好像还是没关系，别急，我们今天要讲的是CloudFlare中的一个功能，叫做Email Obfuscation，也就是邮箱混淆。
当我们使用了 CloudFlare 的服务，如果开启了 Email Obfuscation ，页面里真正的 Email 地址会被隐藏，具体隐藏的代码如下：
<a class="__cf_email__" href="/cdn-cgi/l/email-protection" data-cfemail="a89b9b9c919d9b909a989ae8d9d986cbc7c5">[email&#160;protected]</a><script data-cfhash='f9e31' type="text/javascript">/* <![CDATA[ */!function(t,e,r,n,c,a,p){try{t=document.currentScript||function(){for(t=document.getElementsByTagName('script'),e=t.length;e--;)if(t[e].getAttribute('data-cfhash'))return t[e]}();if(t&&(c=t.previousSibling)){p=t.parentNode;if(a=c.getAttribute('data-cfemail')){for(e='',r='0x'+a.substr(0,2)|0,n=2;a.length-n;n+=2)e+='%'+('0'+('0x'+a.substr(n,2)^r).toString(16)).slice(-2);p.replaceChild(document.createTextNode(decodeURIComponent(e)),c)}p.removeChild(t)}}catch(u){}}()/* ]]> */</script>
做过爬虫的朋友应该都对这段代码不陌生，非常常见的一个邮箱混淆服务。
一.为什么要使用邮箱混淆服务。
事实上很多站长可能并不知道自己使用了这样的服务，因为CloudFlare主要还是以CDN为主，这个服务是附加的，而且CloudFlare也做的很贴心，几乎不用什么设置，就可以自动混淆。（实际上是识别了输出的页面里是否有邮箱，有邮箱就替换成[email protected]并在下面添加一段JS代码）。
无论邮箱和电话号码，即使主动留在互联网上，也不希望被人批量获取，所以邮箱混淆一直是件很重要的事情。包括用at代替@，用#代替@，还有生成图片的。而CloudFlare提供了一种完全不需要修改代码的方案，确实给了大家很大的方便。
二.CloudFlare邮箱混淆服务的优劣
那么我们使用这种方案有什么优点和缺点呢？
我们先说说优点：首先不用改代码，很方便；其次全局替换，不会有遗漏；最后JS混淆，比at和#更加彻底也对显示影响最小，毕竟at和#用的人多了，就和@一样了，有时候还会让真正想联系的客户摸不着头脑。
我们再说说缺点：我在验证码那篇文章中就提到过的成本回报比概念，当很多人用一个方案的时候，由于回报无限增大导致无论这个方案有多么好，他被破解的概率都会大大增加。当然CloudFlare混淆远远不止这个问题这么简单。他的这个混淆有时候恰恰使得这个页面中的邮箱标志更加明显，有点类似本来你把金子放在地上，很危险。现在把金子埋在地下，然后为了自己能找到，又画了一个此处有金子的标记。事实上在真正识别邮箱的爬虫中，CloudFlare反而降低获取Email的难度。所以先提前说反爬的结论，严重不推荐大家使用这个方案！
三.写爬虫时遇到CloudFlare邮箱混淆，如何解密？
说完了反爬，再说反反爬。这个我就不得不提工具的重要性了。有时候你遇到一个好工具，那真的一身轻松。这里需要再次强调，写爬虫最好的语言是JS，因为JS对抗是爬虫中最难的部分，框架本身就是JS的环境将使得事半功倍：
世界上最好的JS爬虫开发框架- 在线网络爬虫/大数据分析/机器学习开发平台-神箭手云。
好了，今天的盒饭有着落了。当然我们这篇文章还是得给爬虫工程师来点干货的。
我们今天就通过这个简单的例子来看看写爬虫时遇到复杂的JS到底怎么分析。
1.格式化代码
JS分析最重要的是格式，因为大部分JS代码都是混淆且压缩的。基本是不具备任何可读性，先格式化成可读的形式最重要。格式化的工具很多，这里我最推荐Chrome浏览其中的Snippet。因为格式化完了之后还可以调试，简直是神器。
我们把前面例子中HTML代码部分删除掉，留下JS部分，贴进Snippet，点击左下角的{}按钮格式化：

2.分析代码
是不是看着舒服太多了，已经到了人眼能看的级别了。我们贴出来看看：
!function(t, e, r, n, c, a, p) { try { t = document.currentScript || function() { for (t = document.getElementsByTagName('script'), e = t.length; e--; ) if (t[e].getAttribute('data-cfhash')) return t[e] }(); if (t && (c = t.previousSibling)) { p = t.parentNode; if (a = c.getAttribute('data-cfemail')) { for (e = '', r = '0x' + a.substr(0, 2) | 0, n = 2; a.length - n; n += 2) e += '%' + ('0' + ('0x' + a.substr(n, 2) ^ r).toString(16)).slice(-2); p.replaceChild(document.createTextNode(decodeURIComponent(e)), c) } p.removeChild(t) } } catch (u) {} }()
我们先大概看下整个代码，首先外层就是一个函数的定义及直接调用。大部分的JS库都会采用这种方法，既可以保证代码模块之间变量不相互污染，又可以通过传入参数实现内外部变量的传输。
再读代码第一段：获取变量t，这个过程我们结合前面的HTML代码可以看出，这个是在获取Email被加密后的Dom元素，为了后面获取data-cfemail做准备。
最后看第二段：显然就是从Dom元素中获取data-cfemail并解密出真实的Email并替换到页面显示中去。
3.整合进爬虫
一般来说，对于复杂的JS，我们还会有断点调试和其他分析的过程，这里的JS很简单，所以咱直接开始写代码。我们怎么在像神箭手这样的JS爬虫框架中处理呢，同样也非常简单：
var cfemails = extractList(content, "//*[@data-cfemail]/@data-cfemail"); for(var c in cfemails){ var a = cfemails[c]; for (e = '', r = '0x' + a.substr(0, 2) | 0, n = 2; a.length - n; n += 2) e += '%' + ('0' + ('0x' + a.substr(n, 2) ^ r).toString(16)).slice(-2); var emailDecoded = decodeURIComponent(e); console.log(emailDecoded); }
可以看到，我们先通过xpath直接获取所有的data-cfemail的值，然后直接把CloudFlare这段解密JS复制过来就行了。运行后就可以直接获取该页面所有被CloudFlare混淆过的邮箱，简直比直接用正则提取邮箱还要简单还要准确。
————————————————最后再说两句——————————————————–
通过这篇文章中这个非常常见却又相对入门的例子，我们一起看了下Javascript在反爬与反反爬中扮演的角色，同时也看到了一个JS爬虫框架的重要性，如果是其他爬虫框架，要么需要自己整合JS引擎，要么就得读懂整段解密代码再翻译成那种语言，这个难度在我们后面会提到的一些JS加密中，将不敢想象。同时我们也可以看到类似CloudFlare这类通用邮箱混淆的脆弱性。
文章导航