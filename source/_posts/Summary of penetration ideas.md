---
title: 渗透基本思路总结
date: 2023-08-29 22:28:00
updated: 2023-08-29 22:28:00
tags: [pentest, security]
category: security
---
> 最近做了一阵子攻防相关的事，正好最近国护结束，做个总结，简单写一下渗透的基本思路（Check List）。
> 
> 不同的标题间内容并不完全独立，在实战中，比如先钓鱼获取到一台个人PC，但这台PC并不在办公网。而后通过收集个人PC的信息，能够登陆外网其他站点的后台，配合一个后台RCE进入办公网/生产网。这其中就有钓鱼，也有外网打点的部分。

## 资产收集

> 资产搜集通俗说就是“了解目标有什么东西”，讲究一个越全越好。

路径：

目标 -> 根域名 -> 子域名/IP (在哪里)-> 端口扫描，目录爆破，指纹（是什么）-> 版本、路由、未授权访问、弱密码、POC扫描（有什么风险）  

### 公司信息

首先拿到目标，需要对目标企业的股权结构等进行收集，明确哪些算是目标，例如收集全资/控股子公司、孙公司等等。

- 企查查
- 爱企查
- 天眼查
### 根域名
- ICP备案
### 子域名/IP

> 子域名和IP的访问形式是互补的，会有仅允许域名访问的站，也当然会有仅允许IP形式访问的站。

- Google Search
	- https://google.com/
- DNS解析记录 
	- https://www.dnsgrep.cn/
- whois IP反查
	- https://who.is/
- SSL证书
	- https://crt.sh/
- 在线查询平台
	- FOFA
	- ZoomEye
	- Hunter
	- Shodan
	- C99.nl
	- VirusTotal
	- PassiveTotal
- 子域名爆破
	- https://github.com/lijiejie/subDomainsBrute
	- https://github.com/projectdiscovery/subfinder
	- https://github.com/knownsec/ksubdomain
	- https://github.com/Mr-xn/subdomain_shell
- 缝合一把梭工具
	- https://github.com/shmilylty/OneForAll
	- https://github.com/projectdiscovery/subfinder
- 清洗结果
	- 泛解析
	- CDN（nslookup、子域名、多地ping、冷门DNS、DNS解析历史）

### 端口扫描

> gogo和fscan互补

- https://github.com/chainreactors/gogo
- https://github.com/shadow1ng/fscan
- [https://nmap.org](https://nmap.org/)
- https://github.com/lcvvvv/kscan

### 目录爆破

- https://github.com/epi052/feroxbuster
- https://github.com/maurosoria/dirsearch
- https://github.com/chainreactors/spray

###  ALL in ONE

> 工程化的资产收集工具，一把梭

- https://github.com/P1-Team/AlliN
- https://github.com/TophantTechnology/ARL
- https://github.com/yogeshojha/rengine
## 钓鱼

> 钓鱼本身比较有想象力，结合手头收集到的人员信息，准备时间长短，手头漏洞，演练情景可以有各式各样的方法。

### 话术

- 运维/开发
	- 安装补丁
	- 安装杀毒软件
- 客服
	- 咨询产品
- HR
	- 简历
- 销售
	- 采购材料
- 通用
	- 关于xxx的通知（考勤/放假/裁员/年终）

### 途径

- 邮箱
- IM（QQ/微信/钉钉）
- 公众号/抖音/微博/小红书/xxx

### C2

C2一般由免杀师傅制作，也可以选择一些开源方案（Sliver/Viper/MSF）。

### 利用

> 钓鱼上线的都是个人PC，可以收集浏览器/IM等信息

- [HackBrowserData](https://github.com/moonD4rk/HackBrowserData)
- [Browser-cookie-steal](https://github.com/DeEpinGh0st/Browser-cookie-steal)
- [BrowserGhost](https://github.com/QAX-A-Team/BrowserGhost/)

## 外网打点

- 常见脆弱资产
	- spring actuator
	- nacos
	- minio
	- 各种OA/CMS
	- redis
	- docker api 2375
	- ftp
	- mysql/postgres/mssql/..
- Web RCE思路
	- 有 nday/1day/0day
	- 没洞现挖
		- （任意）文件上传
		- 任意文件读
		- 任意代码执行、定时任务设置等功能
		-  反序列化
		- sql注入写Webshell
		- SSRF
- 后台登陆
	- 默认密码
	- 动态生成弱密码字典
	- 反混淆JS
	- 鉴权绕过
	- 前台SQL注入
	- 挂水坑

## 权限维持

- WebShell
	- 文件马
		- https://github.com/rebeyond/Behinder
		- https://github.com/BeichenDream/Godzilla
	- 内存马
		- Tomcat
		- WebSocket
		- Filter
		- Servlet
		- Listener
		- Spring Controller
		- Spring Interceptor
		- Valve
		- Agent
		- Timer
		- Jsp
- C2
	- Viper
	- Sliver
	- CS
	- MSF

## 提权

### Linux

gtfobins上汇总了Linux提权手法，很齐全。
https://gtfobins.github.io/

LinPEAS 不需要依赖 列举提权的可能性
https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS

### Windows

- 在线查询
	- http://blog.neargle.com/win-powerup-exp-index/
	- https://i.hacking8.com/tiquan
- 常用手法
	- 土豆家族
		- Rotten Potato
		- BadPotato
		- SweetPotato（Juicy+PrintSpoofer）
		- JuicyPotato
		- GodPotato
		- RoguePotato
		- ...
	- WIndows Kernel 溢出提权
	- 系统配置错误提权（文件权限、注册表、可信任服务路径）
	- 组策略
	- bypassUAC
	- 令牌窃取（可以用土豆系列）
	- 数据库提权（Mysql、Mssql）
## 近源渗透

靠实战积累，在这里不多介绍。

## 内网

- 信息收集（操作系统，网络，进程，文件）
- 非root/administrator 提权
- 个人PC 抓浏览器、微信记录 + 快速横机器防PC下线
- 低频率探测 同内网C段 22，80，8080等常用端口
- 如果完全打不动，那就扫描后快速横向

### 主机信息收集

#### Linux

> https://github.com/nixawk/pentest-wiki/blob/master/1.Information-Gathering/Linux/README.md

```
cat /proc/version
uname -a
cat /proc/sys/kernel/version
cat /etc/issue
ps -ef  
ps aux
top
cat /etc/profile
cat /etc/bashrc
cat ~/.bash_profile
cat ~/.bashrc
cat ~/.bash_logout
env
cat /etc/shadow
cat /etc/passwd
cat /etc/sudoers
find / -name *conf
find / -name *backup
ifconfig
route
lastlog
```
#### Windows

```
systeminfo
hostname
ipconfig /all
whomai /priv
whoami/all
quser or query user #获取在线用户
tasklist /svc  
tasklist /svc | find "TermService" + netstat -ano #获取远程端口
route print，arp -a #查看路由表
netstat -ano #查看开放端口列表
dir c:\programdata\ #分析安装杀软
smbclient -L ip    #本机共享列表/访问权限
wmic process brief #查看进程信息
wmic product get name,version #查询已安装软件
wmic startup get command ,caption #查看启动程序
wmic qfe get Caption,Description,HotFixID,InstalledOn #列出已安装的补丁
wmic useraccount get /all #获取域内用户详细信息
REG query HKLM\SYSTEM\CurrentControlSet\Control\Terminal" "Server\WinStations\RDP-Tcp /v PortNumber #获取远程端口
net user /domain #用户列表
net localgroup administrators
net config workstation #查看当前登陆域和用户信息
net user \\ip\c$  #本机访问的域共享/访问权限
net view /domain #查询域
net view /domain:域名 #查询域内所有计算机
netdom query pdc #查询所有域用户列表
net group /domain #域内所有用户组
net group "domain computers" /domain #域成员计算机列表
net accounts /domain #获取域密码信息
net localgroup administrators #查询本地管理员组的用户
nltest /domain_trusts #获取域信任信息
nltest /DCLIST:域名 #查看域控主机名
nslookup -type=SRV \_ladp._tcp #查看域控制器组
```
### 域渗透
#### 获取域控
- SYSVOL
- MS14-068 域提权
- SPN扫描
- 黄金票据
- 白眼票据
- NTLM relay
- Kerberos 委派
- 地址解析协议
- zerologon CVE
- CVE-2021-42278和CVE-2021-42287
#### 无用户/低权限用户

##### 提权

见提权-Windows部分
##### 获取管理员用户账密
- 获取当前用户密码（mimikatz）
- 绕LSA读密码
- Token窃取
- 查看本地存储密码（LaZagne）
- 卷影拷贝提取 ntds.dit
- dpapi解密
- MS14-068（域提权）
##### 获取其他用户账密

无域用户密码的情况
- 密码喷洒 （用少量密码同时爆多个用户）
- ASREP-Roasting Attack（离线爆破，要求不使用Kerberos预认证）
有域用户账密的情况
- Kerberoasting Attack（获取Hash）
- 爆破SMB共享账密
- 爆破DNS
- Bloodhound 评估域环境问题
## 云上攻防

在演练项目中其实用到的并不多，虽然有各种各样花里胡哨的Docker/K8S攻防技巧，但会遇到的场景不多，这里按照场景来说。

### OSS AKSK

比较容易找到，先用官方CLI确认权限范围内可以控制的内容，然后就是接管权限内的内容。

### ECS、RDS

> https://wiki.teamssix.com/CloudService/EC2/

SSRF 获取Metadata 来获取云控制台权限。
### Docker/K8s

> https://ek1ng.com/cloudsecurity.html

通常打起来都比较麻烦，遇到了不得不打可以看看之前分析写的文章。



