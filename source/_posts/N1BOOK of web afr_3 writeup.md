---
title: N1BOOK of web afr_3 writeup
date: 2022-04-11 14:34:00
updated: 2022-04-11 22:35:00
tags: [ctf,security,<<从0到1：CTFer成长之路>>]
description: <<从0到1：CTFer成长之路>>配套题目 web相关题目的做题记录 N1BOOK of web afr_3 writeup
---

Nu1L Team写的\<\<[从0到1：CTFer成长之路](https://book.nu1l.com/tasks/)\>\>的题目在BUUOJ上都有了复现环境，简单看了看发现WEB的题目wp和环境都比较齐全，决定做一下并且写博客记录一下。

>参考:https://blog.csdn.net/AAAAAAAAAAAA66/article/details/121490787

首页有输入框可以提交name

![](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220411143954822.png)

提交后给出一个article让我们看，先看看里面都是啥

![](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220411143954822.png)

url中可以传参数，随便传一个看看

## 任意文件读取漏洞

![image-20220411144131085](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220411144131085.png)

发现可以任意文件读取，但是尝试读flag是不能直接读的，这里需要对linux的文件目录有一定的了解。

## 利用任意文件读取漏洞结合proc读取源码

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



## SSTI模板注入漏洞

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

## 伪造session完成SSTI利用

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

