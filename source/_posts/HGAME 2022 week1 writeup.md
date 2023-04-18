---
title: HGAME 2022 Week1 writeup
date: 2022-01-28 20:00:00
updated: 2022-01-28 20:00:00
tags: [ctf,security]
category: CTF
---
# HGAME 2022 Week1 writeup by ek1ng

## WEB

### easy_auth

easy_auth的意思是简单的身份验证，题目描述admin在todo上记录了重要内容即flag，考察的是网页登录使用JWT鉴权的知识，当Token的signature采用弱校验方式也就是secrect为空时出现漏洞。

首先我们点击给出的网站看一下，如果直接访问登陆后页面会因为没有token跳转到login.html，题目在login.html中提供了登录和注册功能，那么第一步的话我们先注册一个账户登陆进去看一下，提示success后我们登录注册的账号。

![image-20220127114949682](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127114949682.png)

登录成功后由于这是我们注册的账户，里面什么都没有很正常，然后我们在浏览器的调试工具devtools里查看cookie，发现网页使用JWT的方式进行登录鉴权，那么JWT的鉴权上应该是存在漏洞的，我们需要将Token的形式伪装成admin的身份，通过校验。

![image-20220127115257245](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127115257245.png)

Token是分为三部分的，分别是JWT一共由三部分组成，分别为头部(header),载荷(payload)，签证(signature)。三部分用.进行分割，header中是声明类型和加密的算法，payload中是标准中注册的声明和公共、私有声明，签证部分是将base64编码后的header与base64编码后的payload通过符号.相连，然后通过header中声明的加密方式进行secret组合加密。

我们在jwt.io这样的jwt解密网站上可以查看Token信息

![uTools_1642923037381](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/uTools_1642923037381.png)经过多次尝试后发现将token的ID改为1，UserName的改为admin，将浏览器中暂存的token更改成我们修改后的token我们就可以以admin的形式成功登录，拿到flag啦。

此外需要注意的是，我也尝试了爆破Token，但是注册Token的有效期网站提示只有3分钟，而爆破jwt token 需要好几个小时才能跑完。

![image-20220127114718766](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127114720007.png)



### 蛛蛛...嘿嘿♥我的蛛蛛 

​		题目名字和题目描述在没开始做题前的话我们可以通过我的蛛蛛正在满地找**头**这个题目描述猜测会考察HTTP请求头响应头的知识，题目需要使用python脚本进行自动化点击（或者手动点100个）以及HTTP响应头的知识。

​		我们点击进入给出的网站后出题人提示flag不知道被丢在哪一关了，我们尝试点击按钮后发现，第n关跳转第n+1关有n个按钮只一个按钮是正确的，通过f12我们可以看到哪个按钮会是正确的，但是我也不知道flag会在哪一关呀然后就这么尝试点了100关我还真的点到了...

![image-20220127120745011](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127120745011.png)

​		然后提示flag就在当前页面时，查看http请求头的内容就能发现flag啦，写题解的时候觉得靠手点始终不是个办法，毕竟这次是因为才100关我点了3分钟就看到了但是如果设置的多一些的话就会比较绝望的所以说正儿八经的解法应该是用python脚本去点的，然后我就写了个python脚本，重新做了一遍这题，python代码如下

```python
from time import sleep
from selenium import webdriver

# 打开浏览器
wd = webdriver.Chrome(r'chromedriver.exe')
wd.get('https://hgame-spider.vidar.club/915929bc23')
# 让程序停1s保证打开页面后再自动点击
sleep(1)

# 自动点击
for i in range(100):
    # 点击href值不为空的按钮
    for link in wd.find_elements_by_xpath("//*[@href]"):
        try:
            if link.get_attribute('href') != '':
                wd.get(link.get_attribute('href'))
        except:
            pass

```

![image-20220127154009481](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127154009481.png)

​		运行完python脚本后我们发现从100关直接跳转到了这个页面，看起来flag就被落（藏）在了这里

![image-20220127153726303](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127153726303.png)

​		找到flag落下的地方后，按钮点击也是不会有什么事情发生，f12查看一下前端源码也是啥也没有。

![image-20220127154116916](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127154116916.png)

​		结合题目我的蛛蛛正在满地找**头**，在HTTP响应头中我们发现了自定义的fi4g这个响应头，就找到flag啦

![image-20220127154223419](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127154223419.png)

```
hgame{d76358d92407d56f48a7dd72e34497de5d788abc46b267e2a88b3f974f564b63}
```

### Tetris plus

​		Tetris plus意思是加强版俄罗斯方块，点击页面发现是个小游戏，题目描述告诉我们需要达到3000分，解题需要用到JS审计、Base64、JSfuck编码的知识。

​		题目提示了3000分就会给flag，然后简单玩了玩游戏感觉挺有意思的还，F12查看源代码后，在checking.js文件中找到了校验分数的部分。

![image-20220127155724708](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127155724708.png)

​		发现两个地方，首先是未被注释的部分，当score变量的值大于3000时，页面会弹窗将ZmxhZyDosozkvLzooqvol4/otbfmnaXkuobvvIzlho3mib7mib7lkKch进行Base64解码后的内容给玩家，那么我们直接Base64解码看看，发现这是假的flag。（万恶的出题人即便你玩到了3000分也不会给你flag hhhh）

![image-20220127155434003](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127155434003.png)

​		那么另一个可疑的地方就是下面注释的部分了，经过搜索我们发现这是JSfuck编码，JSfuck编码只需要在控制台中运行就可以解码了，放在控制台中跑一下然后我们也就得到了真的flag。

![image-20220127155452871](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127155452871.png)

### Fujiwara Tofu Shop

​		题目意思是想成为车神需要做一些事情吧大概hhh用的是头文字D的背景，考察的是HTTP请求头、响应头的知识，以及需要能够使用Postman或者Burpsuite这类工具。

​		一开始我还没看懂什么叫去一趟qiumingshan.net，我还以为是需要访问qiumingshan.net什么的，然后发现这个qiumingshan.net的DNS解析不到对应主机总之就是非常迷惑然后问了问出题人发现原来想复杂了，访问的其实就是题目给出的shop.summ3r.top，那么根据描述去一趟qingmingshan.net，我们给HTTP请求头添加Referer，Referer的作用是在客户端向web服务器发送请求时，告诉服务器该网页是从哪个页面链接过来的，服务器因此可以对访客信息做一些处理统计什么的工作。

```
Referer: qiumingshan.net
```

![image-20220123173532745](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220123173532745.png)

网页又提示需要借助AE86才能成为车神，所以将User-Agent的内容改为Hachi-Roku

User-Agent的作用是向访问网站提供你所使用的浏览器类型、操作系统及版本、CPU 类型、浏览器渲染引擎、浏览器语言、浏览器插件等信息的标识，我们原先发送请求时默认会带上的，更改为Hachi-Roku后我们就得到了进一步提示。

```
User-Agent: Hachi-Roku
```

![image-20220123173419360](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220123173419360.png)

题目提示放一盒Raspberry味的曲奇，HTTP请求会使用Cookie做持久化访问，然后我们看了看响应头发现响应头中Cookie有个叫flavor的字段,初始值是Stawberry，恰好符合提示，我们用请求头将其设置为Raspberry就可以了

```
Cookie: flavor=Raspberry
```

![image-20220123174421637](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220123174421637.png)

更改后作者再次提示我们需要将汽油加到100，我们可以在响应头中发现Gasoline:0,Gasoline应该是自定义在响应头中的，非常奇怪，我们翻译一下发现这个单词刚好是汽油的意思，所以说将这个值修改成100就可以了。

```
Gasoline: 100
```

不过我很奇怪的遇到在burpsuite给http请求头因为我中间发过一些什么奇奇怪怪的请求，添加Gasoline: 100无法得到响应，我还以为是不能这么加，但是搜了老半天也是没发现问题

![image-20220123183527273](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220123183527273.png)

所以说怀着疑惑的心情又在postman中尝试了，神奇的发现可以

![image-20220123183732708](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220123183732708.png)

然后再回到burpsuite中尝试了一下，大概是我之前发在加Gasoline: 100之前还添加了什么burpsuite有缓存啥的？？？总之重新试了一下发现可以了

![image-20220123183840576](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220123183840576.png)

​		这里作者提示要从本地访问页面，我们先尝试XFF，XFF是识别客户端最原始的IP地址的，它通过三个Proxy把请求转发给服务端，改成X-Forwarded-For: 127.0.0.1发现不行并且被出题人嘲讽了，然后做到这里我就卡住了，在问了出题人后也是得知本地访问的话其实有好几种请求头的方式，然后出题人把XFF禁用了所以在网上搜了搜发现了X-Real-IP也可以起同样的效果

```
X-Forwarded-For: 127.0.0.1
```

![image-20220123184229239](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220123184229239.png)

所以我们换了一种方式，添加X-Real-IP请求头，发现拿到了flag

```
X-Real-IP:127.0.0.1
```

![image-20220123192904941](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220123192904941.png)

在写题解的时候在写XFF要经过三个proxy转发的时候想到出题人禁用XFF这个问题，所以说去尝试了在后面的proxy里面写127.0.0.1，发现出题人的这种禁用只禁用了第一个127.0.0.1这个proxy，那么这种方法也是可以神奇拿到flag的，然后问了问出题人发现这貌似还是预期之外的（

![image-20220127175452627](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127175452627.png)

## MISC

### 欢迎欢迎！热烈欢迎！

快乐签到( 连签到都没有别人手快

### 这个压缩包有点麻烦

​		主要是使用ARCHPR这个工具，压缩包总共有6位数字爆破 + 字典 + 明文 + binwalk提取结束位后的隐藏zip + 伪加密这么5层加密，需要注意的是压缩包这个题还更新过附件，我觉得应该是旧压缩包在明文这一层有点问题来着所以一直0人解然后我也是卡在明文这个地方，后面的话就看到更新附件想抢一二三血结果被binwalk这一层卡住了然后好像第二天才搞定hhh

​		第一层的加密提示用bandzip打开压缩包就能看到，使用ARCHPR成功解密

![image-20220127180009119](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127180009119.png)

字典加密的话从README.txt里面可以看出，题目给的这个password-not.dic就是密码字典，使用密码字典也是成功解密

![image-20220127180230287](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127180230287.png)

第三层的明文加密，一开始这个给出的README.txt是68B但是压缩包里面的却有73B，不符合明文加密的条件所以也是解不出来，更新附件后就成功解密了，以及README中告诉我们压缩包采用了仅存储的压缩方式，因为明文加密是将README.txt这个压缩包内也有的文件采用相同的方式压缩后，由于有相同的文件，两个压缩包的二进制值会有一部分相等，利用这一点来进行爆破的。

![image-20220127180319561](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127180319561.png)

![image-20220127180556272](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127180556272.png)

解密后我们得到一张flag.jpg，使用winhex查看发现在jpg的结束位后隐藏了信息，我们使用binwalk工具将隐藏的压缩包提取出来，发现仍然是个压缩包，使用winhex打开后，找到504B0102后面的这一串，从50开始数第9，10位是0900，改成0000后保存一下，我们就拿到了flag.jpg

![image-20220127180726846](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127180726846.png)

![image-20220127180847024](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127180847024.png)

### 好康的流量

这题考察了Wireshark流量包分析和使用StegSlove分析LSB隐写图片、文本，这题我是真的尝试了很多种后来才发现是LSB隐写文本拿到了另外半个flag，搜索引擎发现的各种方法都试了一遍

![image-20220127181055179](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127181055179.png)

首先我们打开Wireshark，左上角导出然后另存为，我们发现这是个.eml，然后发现系统自带的邮件和outlook可以打开但是windows自带的邮件我以前也没添加过账户然后我添加账户还添加不上去，然后换了outlook打开了这个邮件，那么看到两个提示，第一个是LSB，提示使用了LSB隐写，第二个就是这个图片了我们把它保存下来

![image-20220124163829914](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220124163829914.png)

用StegSlove打开后在绿色通道2上我们发现这里隐写了一个条形码，打开支付宝扫码发现这里有半个flag

```
hgame{ez_1mg_
```

![image-20220124174402014](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220124174402014.png)

然后另外半个flag需要使用stegslove打开图片后，在Extract By选择按列计算才能够拿到这半个，一开始选的是按行然后通道换来换去也是没找到隐写的信息。

![image-20220124174347283](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220124174347283.png)

拼凑起来我们就拿到了flag啦

```
hgame{ez_1mg_Steg4n0graphy}
```

### 群青(其实是幽灵东京)

这是考察音频隐写的一个题。

![image-20220127181553303](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127181553303.png)

搜索后发现SilentEye是个隐写的工具，下载后尝试解密但是啥也没得到，于是用audacity打开，查看频谱图发现隐写了文本Yoasobi

![image-20220127181818314](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127181818314.png)

在SilentEye里将Yoasobi填为密码，发现可以成功解密

![image-20220127181914229](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127181914229.png)

下载新的音频后，搜索SSTV发现 慢扫描电视（Slow-scan television 简称*SSTV*）是业余无线电爱好者的一种主要图片传输方法那么可以判断这里隐藏了图片，我们使用RX-SSTV这类的解密SSTV的软件，发现了隐写的二维码，扫码就能够得到flag啦

![image-20220123200358935](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220123200358935.png)

## CRYPTO

### Dancing Line

题目和描述给出的信息不多，打开发现就是以Dancing Line这款游戏为背景的一个图片，用一条线的横和竖表示了0，1，那么黑点有什么用呢，我数了数发现有37个好像（，然后一个flag的长度也差不多，那我们就可以猜测这个解出来就是flag，每两个黑点间的一段就是表示flag的一个字符，但是如何清楚的看清图片走向呢，我找了个在线ps网站，打开图片发现这个图片恰好是每步一个像素点

https://ps.gaoding.com/#/

![image-20220126134730285](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220126134730285.png)

那这样的话我们只需要数一数或者想办法用python脚本做一些图像识别的工作再转换成flag就可以了，由于这里我想了想点不算多，20分钟可以搞定，于是我才用了手动数的方法，下面是手动数记录的过程hhh，我数了前5个然后转换了一下0和1的对应关系，刚好能对应上hgame，横是0竖是1，然后就每次数5个，把这个flag数出来了

![uTools_1643176015993](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/uTools_1643176015993.png)

写题解的时候又思考了一下，正确的解法应该是用Pillow这个Python的图像处理库，来做这个信息处理的工作，可惜写题解写到这里的时候已经18.26了马上8点要week2了，所以没来得及把这个题写个脚本处理一下试试。

### Easy RSA

这是一道比较简单的RSA算法的题目

首先查资料了解什么是rsa：

1. 随意选择两个大的质数p和q，p不等于q，计算N = pq.

2. 根据欧拉函数，求得r=φ(N)=φ(p)φ(q)=(p-1)(q-1)。

3. 选择一个小于r的整数e,是e与r互质。并求得e关于r的模反元素，命名为d。(求d令ed≡1(mod r))。(模反元素存在，当且仅当e与r互质）

4. 将p和q的记录销毁。

其中(N，e)是公钥，(N，d)是私钥。

然后我们看到题目的加密算法：将flag每个字符字符通过rsa加密并给出了pqe和c，那么我们要做到就很简单了，求出d然后解出m。

```python
import gmpy2
def make_key(p, q, e):
    n = p * q
    phi = (p-1) * (q-1)
    d = gmpy2.invert(e, phi)
    return d
m = [(12433, 149, 197, 104), (8147, 131, 167, 6633), (10687, 211, 197, 35594), (19681, 131, 211, 15710), (33577, 251, 211, 38798), (30241, 157, 251, 35973), (293, 211, 157, 31548), (26459, 179, 149, 4778), (27479, 149, 223, 32728), (9029, 223, 137, 20696), (4649, 149, 151, 13418), (11783, 223, 251, 14239), (13537, 179, 137, 11702), (3835, 167, 139, 20051), (30983, 149, 227, 23928), (17581, 157, 131, 5855), (35381, 223, 179, 37774), (2357, 151, 223, 1849), (22649, 211, 229, 7348), (1151, 179, 223, 17982), (8431, 251, 163, 30226), (38501, 193, 211, 30559), (14549, 211, 151, 21143), (24781, 239, 241, 45604), (8051, 179, 131, 7994), (863, 181, 131, 11493), (1117, 239, 157, 12579), (7561, 149, 199, 8960), (19813, 239, 229, 53463), (4943, 131, 157, 14606), (29077, 191, 181, 33446), (18583, 211, 163, 31800), (30643, 173, 191, 27293), (11617, 223, 251, 13448), (19051, 191, 151, 21676), (18367, 179, 157, 14139), (18861, 149, 191, 5139), (9581, 211, 193, 25595)]
print(len(m))
for i in range(38):
    e, p, q, c = m[i]
    d = make_key(p, q, e)
    n = p * q
    plaintext = (gmpy2.powmod(c, d, n))
    print(chr(plaintext), end='')
```

![image-20220127184506877](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127184506877.png)

### Matryoshka

题目就是已知了凯撒加密的key和维吉尼亚加密的key，意思就是密文flag被层层加密后变成了盲文，现在需要解密回去，解密顺序是这样的

Braille 盲文 - > Reverse 逆序密码 -> Morse 摩斯 -> hex 转16进制 -> Vigenère 维吉尼亚 -> base64 -> 栅栏:22 -> Caesar 凯撒:21 -> Flag

![image-20220124121251165](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220124121251165.png)

cyberchef不知道为什么解不出这层莫斯，然后换了个在线解密网站解出来了，如果不加这层逆序的话莫斯之后后面是解着解着就发现有问题的，也是根据hint 2个置换2个代换4个编码，已知有维吉尼亚和凯撒两个代换密码了，所以说的话如果莫斯直接解不出那么就看看能不能加一层代换密码，加了逆序后再莫斯就能够顺利解出来了

![image-20220124121545733](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220124121545733.png)

解出莫斯密码后因为字符中最大都没有超过F，猜测是16进制编码 去cyberchef里面转HEX

![image-20220124121649120](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220124121649120.png)

再解一层维吉尼亚后 这个显然是base64的形式所以说再解一层base64

![image-20220124121716256](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220124121716256.png)

然后栅栏的话我在cyberchef好像找不到，可能是我搜的不吧于另外找了个网站尝试了

![image-20220124121926158](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220124121926158.png)

一个一个试过去，因为已经知道最后一层是凯撒然后栅栏密码的key也不会超过这个原文长度，那么key从1开始解码之后去凯撒21再解一层看看是不是flag的形式就好，最后也是试出key=22时候能解密出flag

![image-20220124122755581](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220124122755581.png)

后来发现原来cyberchef是解的出morsecode的 是我分隔符没有选对 选forward slash就可以了

![image-20220124122639452](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220124122639452.png)





### English Novel

题目首先给出了410个分块加密的txt，其中每个txt是410字节，是能够一一对应的，那么整体的思路就是先根据加密算法写一个python的脚本找到加密的key，然后用key和加密后的flag.enc找到加密前的flag.enc

一开始没有注意到原文和密文并不是根据序号直接对应的所以说尝试解了几个后发现每个解出来的key都不一样

![uTools_1643097786581](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/uTools_1643097786581.png)

首先是寻找对应part部分 findPart.py

```python
import os

filePath=r"" # 文件夹路径
fileList=os.listdir(filePath)

for file in fileList:
	f=open(os.path.join(filePath,file))
	while True:
		line = f.readline()
		if not line:
			break
		line = line.strip('\n')
		key = 'ktl'
		if key in line:
			print(file)# txt文件内容
			print(line)

```

根据findPart.py 检索关键字

由于flag的开头一定是hgame 那么的话有了flag.enc我们就能知道key[0]-key[4]的值

比如part0 最前面5个字符是 read 加密后就会是 wmme 然后我们用 wmme去加密后的part里面检索，就能够找到part0对应了part175

解密部分 encrypt.py

```python
encryptData = "" # 未加密数据
originalData = "" # 加密后数据
key = [0 for i in range(410)]
encryptFlag = 'klsyf{W0_j0v_ca0z_\'Ks0ao-bln1qstxp_juqfqy\'?}'
def encrypt(originalData, encryptData,):
    for i in range(len(originalData)):
            key[i] = ord(encryptData[i]) - ord(originalData[i])
def enflag(key,encryptFlag):
    result = ""
    for i in range(len(encryptFlag)):
        if encryptFlag[i].isupper():
            result += chr((ord(encryptFlag[i]) - ord('A') - key[i]) % 26 + ord('A'))
        elif encryptFlag[i].islower():
            result += chr((ord(encryptFlag[i]) - ord('a') - key[i]) % 26 + ord('a'))
        else:
            result += encryptFlag[i]
    return result
encrypt(originalData,encryptData)
print(enflag(key,encryptFlag))

```

解密之后由于我们key只能在大小写字符处得到，而如果是空格引号等我们的key是默认为0的，所以说只用一组数据我们只能解一部分除非你找的这一组数据前面相当于flag长度的这么一串都是大小写字符但是这个肯定不太可能的，所以说我再随便找了2组数据。

part21对应326，part38对应pary280:

根据3个对应关系找到的flag 然后稍微拼凑一下就得到flag啦

```
# flag.enc klsyf{W0_j0v_ca0z_'Ks0ao-bln1qstxp_juqfqy'?}
# part 0 -> part 175 kgame{W0_y0u_kn0w_'Kn0wn-bla1nttxt_attack'?}
# part 21 -> part 326 hgame{W0_y0v_kn0w_'Kn0wn-pla1ntext_autaqk'?}
# part 38 -> part 280 kgame{D0_j0u_ka0w_'Kn0wo-pln1nttxt_attacy'?}
# hgame{D0_y0u_kn0w_'Kn0wn-pla1ntext_attack'?}
```

![uTools_1643099089206](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/uTools_1643099089206.png)

## IOT

### 饭卡的uno

这个题的话我是不太懂iot的，然后看到下载下来是个HEX文件，hex是16进制嘛所以说先解码成字符看一看

![image-20220127185120016](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127185120016.png)

然后我们好像就得到了flag，虽然说格式有点问题，但是flag一般是把字符改成数字一些，还是可以猜出原意，把一些乱七八糟的去掉后也是成功拿到flag，然后后来因为web misc crypto都ak了去看看能不能做几个我一点也不会的pwn和re，接触到了ida，使用ida反汇编后，我们就直接发现flag其实被记录在这里。

![image-20220127184801923](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220127184801923.png)

## RE

### easyasm

可执行文件是打不开的，是个DOS程序，然后题目也说了asm，asm指Assembly，所以说这题是需要看汇编代码

我们将可执行文件拖入IDA查看，首先是能够看到flag的一个形式

![uTools_1643189941375](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/uTools_1643189941375.png)

![uTools_1643191437159](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/uTools_1643191437159.png)

程序首先会将si和1Ch做比较，如果相等就会走右边提示right，如果不相等就会走左边，当左边的循环走完后如果没有进入right那就会进入wrong，这里可以得出这个for循环需要让si从00H一直++到1Ch 总共走28次，下面是对于程序块汇编代码的翻译

![image-20220126181203473](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220126181203473.png)

```asm
xor     ax, ax ax是累加器寄存器，ax和ax一定相等 结果是将ax清零
mov     al, [si] si是源变址寄存器，用于存放源操作数的偏移地址，将si地址上的值给al al是低8位寄存器 l表示low 是ax寄存器的低8位
shl     al, 1    
shl     al, 1    
shl     al, 1    
shl     al, 1    左移4位
push    ax       push ax是把ax里的值压入堆栈。即当前esp－4出的值变为ax的值，ax本身的值不变。
xor     ax, ax   重复前面的操作
mov     al, [si] 
shr     al, 1    
shr     al, 1	 
shr     al, 1	 
shr     al, 1 	 右移4位
pop     bx       pop bx是把当前esp的值赋给bx，并且esp＋4（bx的值改变，esp在pop之前指向的地方的值不变，即堆栈里的哪个值不会自动清零）此时bx = 08H
add     ax, bx   相当于把这个2位16进制数左右两位呼唤
xor     ax, 17h  和17H异或
add     si, 1	 si++
cmp     al, es:[si-1] 将al和以es为段首地址以si为偏移地址的值进行比较
jz      short loc_100DD
```

所以我们需要先和17H异或，然后把异或的值再左右位互换，转成ascii码表示的字符，就能够得到flag了啦

一开始还没想明白，忘了逆向是flag校验的一个过程，需要倒着算回去，正着算我寻思咋算也不太对，后来想明白后算了算前俩，检验发现前两个字符是 h,g，那么也说明我们这个思路是正确的

![image-20220126194950180](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220126194950180.png)

然后我尝试直接写个python脚本解决，发现位运算上我这个脚本就很奇怪，溢出的部分不会自动忽略到，然后因为想着快点拿到flag所以说先和17H异或，再把异或的结果人工把前后两位换了一下，最后用程序转ascii输出了，算是半自动吧（

![image-20220126200727195](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220126200727195.png)

### flagchecker

这是个apk逆向的题，我用了jadx-gui然后反编译了这个apk，并且找到了校验flag的类MainActivity。

从代码看flag应该是先以carol为key用RC4加密然后再base64加密后能够等同于这一串字符串，通过校验。

![uTools_1643173840445](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/uTools_1643173840445.png)

那么我们只需要把这个里面明文表示的字符串先base64解码，再RC4解码就可以得到flag了，这里使用了cyberchef这个在线解密网站。

**![image-20220126131107340](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220126131107340.png)**

## PWN

### test nc

只需要用用nc命令连接靶机，然后靶机直接给了我们shell，那这样的话cat flag我们就拿到flag啦

然后由于pwn一直被扫端口现在pwn的靶机连接前都需要进行身份验证了，我也是没有这个爆破这个sha256前4位的脚本没法复现只能文字描述一下啦。
