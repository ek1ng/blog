---
title: HGAME 2022 Final writeup (unfinished)
date: 2022-03-12 20:10:00
tags: [ctf,security]

---

# picoCTF writeup(unfinished)

picoCTF是CMU举办的新手赛，主要做了一下web题，感觉都是比较简单，有几题遇到了坑简单记录一下啦

## WEB

### SQLiLite

考点为sql注入而且会直接回显内容

![image-20220325165528317](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220325165528317.png)

不知道为什么这样就能成功登录，但是一些常见的万能密码不行...

![image-20220325165454307](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220325165454307.png)

### Secrets

用dirsearch扫一下发现目录/secret/，目录下有index.html和/hidden/file.css，但是这俩文件里面啥有用的也没有发现，卡住了

![image-20220325173435169](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220325173435169.png)

