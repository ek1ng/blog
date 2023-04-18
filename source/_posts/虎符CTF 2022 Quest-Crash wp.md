---
title: 虎符CTF 2022 Quest-Crash Writeup
date: 2022-03-21 17:00:00
updated: 2022-03-21 17:00:00
tags: [ctf,security]
category: CTF
---

# 虎符CTF 2022 Quest-Crash Writeup

加入协会后的第一场CTF，出了一个简单的misc，记录一下

题目给出了一个访问链接，仔细看了看网站的内容，大意是提供了一个有redis的存储的服务，并且给出系统信息和redis的配置，以及相关增删改查功能，然后让你crash这个服务。

getflag处提示redis-server is still running, pid: 7. ，Try harder!

题目给flag的要求是应该把**redis服务打挂**。

用burpsuite然后intruder发包是不好使的，这点流量估计打不挂redis，要另辟蹊径应该

感觉更有可能是KEYS * 命令查询的时候会导致宕机

抓包分析 请求KEYS*命令的时候是以POST请求发送{"query":"KEYS *"}的形式查询的，用户输入内容跟在*后面，这样的话可以构造查询语句

根据原理，**redis单线程的keys指令会导致线程阻塞一段时间，线上服务会停顿，直到指令执行完毕，服务才能恢复**。尝试了使用intruder增加2w条key，然后再用intruder一直发送KEYS*命令，redis还是很正常。

新思路，可能这个KEYS*不会模糊匹配，比如说我要给h?ello这样的数据然后再用KEYS*去匹配里面所有这样的数据，正在尝试中

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/asynccode)

这样会先把web服务打挂，而不是redis先挂，卡住了卡住了

**正确思路**，KEY*查询会导致redis锁，这个时候发起大量GET，然后这些GET要等KEY*查询结束后才会进来，这时候可以可以让redis压力突然增大，可能就会寄

设置50线程发送KEY*查询指令

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/asynccode)

设置100线程发送SET的指令，往redis中添加hxxxxello:hello的键值对

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/asynccode)

设置50线程发送GET指令查询

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/asynccode)

KEY*指令是单线程的，会导致redis锁，然后不停的发KEY*就会导致堵塞，排在队列后面的的SET和GET指令一下子涌入就会把redis击穿

按长度查看GETflag的指令，找到了返回flag的这次请求

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/asynccode)

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/asynccode)

```
HFCTF{f1abf147-e3c1-4cb9-a8cb-14ea1516db05}
```