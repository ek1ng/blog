---
title: Mysql是如何存储用户账号密码
date: 2023-05-06 14:34:00
updated: 2023-05-06 18:48:00
tags: [security]
description: 介绍Mysql不同版本如何存储Mysql用户账号密码
---

> 研究这个问题主要是基于主机安全的一个需求场景，即在能够访问主机文件系统的情形下，如何在代码中通过读文件拿到`Mysql`的账号密码，并且做对应的安全检测，例如检测是否存在弱密码。

## 账号密码存在哪

首先，mysql的用户密码是存储在一个叫做`mysql`的数据库的`user`数据表中的，这是一张系统表。

### mysql5.7

```
FROM mysql:5.7

ENV MYSQL_ROOT_PASSWORD=root

EXPOSE 3306/tcp
```

在mysql5.7中，我们访问数据库，`user`表中的`User`字段表示用户名，`authentication_string`字段则是加密后的密码。

![image-20230413160949954](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230413160949954.png)

那么`user`表在`Mysql5.7`中，对应的文件存在`/var/lib/mysql/mysql/user.MYD`

![image-20230413161546184](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230413161546184.png)

### mysql8

```
FROM mysql:8.0

ENV MYSQL_ROOT_PASSWORD=root

EXPOSE 3306/tcp
```

同样对于mysql8，在user表中，`user`表中的`User`字段表示用户名，`authentication_string`字段则是加密后的密码。

![image-20230413164432147](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230413164432147.png)

`user`表在`Mysql8`中，对应的文件存在`/var/lib/mysql/mysql.ibd`

## 解析用户名和加密后的密码

### mysql5.7

这是mysql这个库中user表的主要内容

| Host      | User          | plugin                | authentication_string                     |
| --------- | ------------- | --------------------- | ----------------------------------------- |
| localhost | root          | mysql_native_password | *81F5E21E35407D884A6CD4A731AEBFB6AF209E1B |
| localhost | mysql.session | mysql_native_password | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE |
| localhost | mysql.sys     | mysql_native_password | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE |
| %         | root          | mysql_native_password | *81F5E21E35407D884A6CD4A731AEBFB6AF209E1B |
| localhost | root          | mysql_native_password | *94BDCEBE19083CE2A1F959FD02F964C7AF4CFC29 |

mysql5.7的**系统表**使用MyIASM存储，因此主要就是要解决MyIASM的文件格式如何去解析的问题，这里呢我发现了仓库`https://github.com/feix/mysqld_user_myd`，这是一个用`python`写的用于从`user.MYD`文件中解获取`host`,`name`,`password`的脚本，这里我就稍加改动，写成了一版`go`的。具体的实现可以参看这个PR：`https://github.com/chaitin/veinmind-tools/pull/234`。

### mysql8

对于Mysql8，系统表的存储引擎改成了InnoDB，因此对应的文件格式也就变了，而我主要的需求是完善某项目的弱密码扫描插件，其中Mysql8的InnoDB解析的部分已经写好了，因此就没有去操心了。

## 如何加密

### mysql5

`mysql8`之前采用的密码认证插件为`mysql_native_password`，具体呢是用`SHA1`算法对密码进行加密。

下面是`SHA1`的一些细节

1. 获取客户端提交的明文密码。
2. 从一个加密好的常量字符串（也称 salt）中获取一个字符的编码值，插入到明文密码中。
3. 重复进行第 2 步，直到插入足够多的编码值，构成一个字符串。
4. 对这个字符串进行 SHA1 哈希运算，得到最终的密码摘要。

`SHA1`一般各个语言都有实现好的标准库/第三方库，可以直接调用来实现。

### mysql8

`mysql`8开始使用，默认的密码插件改为了`caching_sha2_password`。采用了`SHA-256`的算法对密码进行哈希，并且与服务器商城的随机插盐值进行组合，因此更加的安全。在这种情况下，从本地文件系统里，通过读文件解析出来的哈希值，由于不清楚插盐值，比较难写一个加密算法，来验证提取出来的哈希后的密码，原文是否可能是弱密码，因此暂时搁置了。
