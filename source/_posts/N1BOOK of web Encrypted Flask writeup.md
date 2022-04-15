---
title: N1BOOK of web Encrypted Flask writeup(unfinished)
date: 2022-04-13 22:02:00
updated: 2022-04-13 11:56:00
tags: [ctf,security,<<从0到1：CTFer成长之路>>]
description: <<从0到1：CTFer成长之路>>配套题目 web相关题目的做题记录 N1BOOK of web Encrypted Flask writeup
---

## 客户端session

题目主要是考客户端session的一种漏洞，所以我们先来说说客户端session是什么，这一部分内容主要参考了[P牛的文章](https://www.leavesongs.com/PENETRATION/client-session-security.html)。

### 什么是客户端session

在传统PHP开发中，`$_SESSION`变量的内容默认会被保存在服务端的一个文件中，通过一个叫“PHPSESSID”的Cookie来区分用户。这类session是“服务端session”，用户看到的只是session的名称（一个随机字符串），其内容保存在服务端。

然而，并不是所有语言都有默认的session存储机制，也不是任何情况下我们都可以向服务器写入文件。所以，很多Web框架都会另辟蹊径，比如Django默认将session存储在数据库中，而对于flask这里并不包含数据库操作的框架，就只能将session存储在cookie中。

因为cookie实际上是存储在客户端（浏览器）中的，所以称之为“客户端session”。

### 客户端session的加密过程

序列化session的主要过程：

1. json.dumps 将对象转换成json字符串，作为数据
2. 如果数据压缩后长度更短，则用zlib库进行压缩
3. 将数据用base64编码
4. 通过hmac算法计算数据的签名，将签名附在数据后，用“.”分割

第4步就解决了用户篡改session的问题，因为在不知道secret_key的情况下，是无法伪造签名的，注意到，在第4步中，flask仅仅对数据进行了签名。众所周知的是，签名的作用是防篡改，而无法防止被读取。而flask并没有提供加密操作，所以其session的全部内容都是可以在客户端读取的，这就可能造成一些安全问题。

就像我们做的这个题目中的session一样，flask生成了加密后的客户端session，我们可以看到加密后的内容。

![image-20220414115621070](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220414115621070.png)

