---
title: Mysql如何存储用户账号密码
date: 2022-04-11 14:34:00
updated: 2022-04-16 18:48:00
tags: [ctf,security]
description: 介绍Mysql不同版本如何存储Mysql用户账号密码
---

## 账号密码存在哪

这里基于了主机安全的情形来进行讨论，即在能够访问文件系统的情形下，如何在代码中通过读文件拿到`Mysql`的账号密码。因为如果是个人用户，找不到账号密码之类的情形，可以通过`Mysql`的`shell`很轻松的解决，

### mysql4.x-5.x

版本太老 懒得研究了

### mysql5.7

```
FROM mysql:5.7

ENV MYSQL_ROOT_PASSWORD=root

EXPOSE 3306/tcp
```

访问数据库，`user`表中的`User`字段表示用户名，`authentication_string`字段则是加密后的密码。

![image-20230413160949954](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230413160949954.png)

那么`user`表在`Mysql5`中，对应的文件存在`/var/lib/mysql/mysql/user.MYD`

![image-20230413161546184](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230413161546184.png)

### mysql8

```
FROM mysql:8.0

ENV MYSQL_ROOT_PASSWORD=root

EXPOSE 3306/tcp
```

同样在user表中，`user`表中的`User`字段表示用户名，`authentication_string`字段则是加密后的密码。

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

我没能找到Myisam引擎是具体如何在文件中存储user表的，所以我这里的思路就是，根据文本内容基于正则做匹配。

![image-20230414111459668](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230414111459668.png)

这里通过观察可以看到，对于除了Host为localhost，User为root的用户，并没有完整的`mysql_native_password`这串字符串，其他用户的用户名可以通过Host+1字节+任意长度用户名至下一个16进制为01的字符的方式匹配，密码可以通过mysql_native_password+2字节+41字节长度的密码的方式匹配。

接下来就可以简单编写一个正则，试试能不能匹配到用户名和密码。







## 如何加密

### mysql5

`mysql8`之前采用的密码认证插件为`mysql_native_password`

### mysql8

`mysql`8开始使用，增加了`caching_sha2_password`作为`mysql8`的密码认证插件。