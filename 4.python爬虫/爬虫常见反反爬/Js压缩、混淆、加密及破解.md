Js压缩、混淆、加密及破解
js压缩、混淆、加密介绍
三者的定义与区别
压缩：删除 Javascript 代码中所有注释、跳格符号、换行符号及无用的空格，从而压缩 JS 文件大小，优化页面加载速度。
混淆：经过编码将变量和函数原命名改为毫无意义的命名（如function(a,b,c,e,g)等），以防止他人窥视和窃取 Javascript 源代码，也有一定压缩效果。
加密：一般用eval方法加密，效果与混淆相似，也做到了压缩的效果。
三者的作用
压缩的主要目的是消除注释等无用字符，达到精简js代码，减小js文件大小的目的，这也是页面优化的一种方式；
混淆和加密的目的比较接近，都是为了防止他人直接查看源码，对代码（如重要的api等）起保护作用，但这也只是增加了阅读代码的代价，也就是所谓的防君子不防小人。
加密就是通过已经有的或者自己编写的加密方法，对js代码进行加密转换，但是当混淆和加密联合使用时，如先混淆在加密（或者先加密再混淆）时，破解时间就会增加。关于js的加密
浏览器是怎么解析混淆和加密后的js代码的
其实变量名只要是Unicode字符就行了，对于js引擎来说都是一样的，只是人类觉得他们不同而已。
是否可以破解压缩\混淆\加密后的代码
压缩和混淆后的js代码是不可以被还原的，但是我们可以将js加密的代码进行解密，后面的几篇是介绍如何进行js解密的。
常见的加密和解密方式介绍
加密方式的分类
一：最简单的加密解密  
二：转义字符"\"的妙用  
三：使用Microsoft出品的脚本编码器Script Encoder来进行编码   （自创简单解码）  
四：任意添加NUL空字符（十六进制00H）    （自创）  
五：无用内容混乱以及换行空格TAB大法  
六：自写解密函数法  
一、escape()和unescape()
加密方式的介绍
大家对于JAVASCRIPT函数escape()和unescape()想必是比较了解啦（很多网页加密在用它们），分别是编码和解码字符串
比如例子代码用escape()函数加密后变为如下格式：alert%28%22%u9ED1%u5BA2%u9632%u7EBF%22%29%3B
如果愿意我们可以写点JAVASCRIPT代码重新把它加密如下%61%6C%65%72%74%28%22%u9ED1%u5BA2%u9632%u7EBF%22%29%3B
这样加密后的代码是不能直接运行的，可以使用eval(codeString)，这个函数的作用就是检查JavaScript代码并执行，必选项 codeString 参数是包含有效 JavaScript 代码的字符串值，加上上面的解码unescape()
解密方式
直接将eval换成alert配合unescape便实现了解密
加密示例
加密方法
            <SCRIPT LANGUAGE="JavaScript">  
            varcode=unescape("%61%6C%65%72%74%28%22%u9ED1%u5BA2%u9632%u7EBF%22%29%3B");  
            eval(varcode)  
             </SCRIPT>
二、转义字符"\"加密
加密方式介绍
JavaScript提供了一些特殊字符如：\n （换行）、 \r （回车）、\’ （单引号 ）等应该是有所了解
其实"\"后面还可以跟八进制或十六进制的数字，如字符"a"则可以表示为："\141"或"\x61"（注意是小写字符"x"），至于双字节字符如汉字"黑"则仅能用十六进制表示为"\u9ED1"（注意是小写字符"u"），其中字符"u"表示是双字节字符；
解密示例
直接将eval换成alert便实现了解密
                            # 八进制转义字符串如下:  
                            <SCRIPT LANGUAGE="JavaScript">  
                            eval("\141\154\145\162\164\50\42\u9ED1\u5BA2\u9632\u7EBF\42\51\73")  
                            </SCRIPT>
                          # 十六进制转义字符串如下:  
                            <SCRIPT LANGUAGE="JavaScript">  
                            eval("\x61\x6C\x65\x72\x74\x28\x22\u9ED1\u5BA2\u9632\u7EBF\x22\x29\x3B")  
                            </SCRIPT>
三：使用Microsoft出品的脚本编码器Script Encoder来进行编码
加密方式的介绍
直接使用Microsoft出品的JavaScript调用控件Scripting.Encoder完成的编码！
如果你觉得这样编码得到的代码LANGUAGE属性是JScript.Encode，很容易让人识破，那么还有一个几乎不为人知的window对象的方法execScript()，其原形为：
window.execScript( sExpression, sLanguage )  
sExpression: 必选项。字符串(String)。要被执行的代码。  
sLanguage　: 必选项。字符串(String)。指定执行的代码的语言。默认值为 Microsoft JScript
解密方式
编码后的代码运行前IE会先对其进行解码，如果我们先把加密后的代码放入一个自定义函数如上面的decode()中，然后对自定义函数decode调用toString()方法，得到的将是解码后的代码！
加密示例
加密方法
                        # 使用Scripting.Encoder
                         <SCRIPT LANGUAGE="JavaScript">  
                         var Senc=new ActiveXObject("Scripting.Encoder");  
                         var code='<SCRIPT LANGUAGE="JavaScript">\r\nalert("黑客防线");\r\n<\/SCRIPT>';  
                         var Encode=Senc.EncodeScriptFile(".htm",code,0,"");  
                         alert(Encode);  
                         </SCRIPT>
                         #  使用execScript()
                         <SCRIPT LANGUAGE="JavaScript">  
                         execScript("#@~^FgAAAA==@#@&ls DD`J黑客防线r#p@#@&FgMAAA==^#~@","JScript.Encode")  
                         </SCRIPT>
                         # 加密结果
                          <SCRIPT LANGUAGE="JScript.Encode">#@~^FgAAAA==@#@&ls DD`J黑客防线r#p@#@&FgMAAA==^#~@</SCRIPT> 
解密示例
解密方法
                        <SCRIPT LANGUAGE="JScript.Encode">  
                        function decode(){};
                        alert(decode.toString());  
                        </SCRIPT>
四：任意添加NUL空字符（十六进制00H）
加密方式介绍
一次偶然的实验，使我发现在HTML网页中任意位置添加任意个数的"空字符"，IE照样会正常显示其中的内容，并正常执行其中的JavaScript 代码。
而添加的"空字符"我们在用一般的编辑器查看时，会显示形如空格或黑块，使得原码很难看懂，如用记事本查看则"空字符"会变成"空格"，利用这个原理加密结果如下：（其中显示的"空格"代表"空字符"）
如果不知道方法的人很难想到要去掉里面的"空字符"（00H）的
解密方式
去掉代码中的00H
加密示例
加密方法
            <S    C    RI   P T L    ANG U   A        GE      
            ="    J   a    v a S    c r    i p t">
            a    l er    t   (" 黑    客 防 线")   ;    

            <    /    SC   R    I    P    T>  
五：无用内容混乱以及换行空格TAB大法
加密方式介绍
在JAVASCRIPT代码中我们可以加入大量的无用字符串或数字，以及无用代码和注释内容等等
这使真正的有用代码埋没在其中，并把有用的代码中能加入换行、空格、TAB的地方加入大量换行、空格、TAB，并可以把正常的字符串用""来进行换行，这样就会使得代码难以看懂！如我加密后的形式如下：
解密方式
慢慢的分析吧
加密示例
加密方法
​            <SCRIPT LANGUAGE="JavaScript">  
​            "xajgxsadffgds";1234567890  
​            625623216;var $=0;alert//@$%%&*()(&(^%^  
​            //cctv function//  
​            (//hhsaasajx xc  
​            /*  
​            asjgdsgu*/  
​            "黑
​             客  
​            防线"//ashjgfgf  
​            /*  
​            @#%$^&%$96667r45fggbhytjty  
​            */  
​            //window  
​            )  
​            ;"#@$#%@#432hu";212351436  
​            </SCRIPT>
六：自写解密函数法
加密方式介绍
这个方法和一、二差不多，只不过是自己写个函数对代码进行加密
很多VBS病毒使用这种方法对自身进行加密，来防止特征码扫描！下面是我写的一个简单的加密解密函数，
加密示例  
加密方法
​                            # 加密方法
​                            <SCRIPT LANGUAGE="JavaScript">  
​                            function compile(code)  
​                            {    
​                              var c=String.fromCharCode(code.charCodeAt(0)+code.length);  
​                               for(var i=1;i<code.length;i++){  
​                              c+=String.fromCharCode(code.charCodeAt(i)+code.charCodeAt(i-1));  
​                               }  
​                               alert(escape(c));  
​                            }  
​                            compile(’alert("黑客防线");’)  
​                            </SCRIPT>
​                            # 加密结果
​                            o%CD%D1%D7%E6%9CJ%u9EF3%uFA73%uF1D4%u14F1%u7EE1Kd          
解密示例
解密方法
​                    <SCRIPT LANGUAGE="JavaScript">  
​                    function uncompile(code)  
​                    {  
​                       code=unescape(code);  
​                       var c=String.fromCharCode(code.charCodeAt(0)-code.length);  
​                       for(var i=1;i<code.length;i++){  
​                      c+=String.fromCharCode(code.charCodeAt(i)-c.charCodeAt(i-1));  
​                       }  
​                       return c;  
​                    }  
​                    eval(uncompile("o%CD%D1%D7%E6%9CJ%u9EF3%uFA73%uF1D4%u14F1%u7EE1Kd"));  
​                    </SCRIPT>