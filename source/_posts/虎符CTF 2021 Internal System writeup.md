---
title: HFCTF 2021 Internal System writeup
date: 2022-03-09 22:11:00
tags: [ctf,security]
cover: https://w.wallhaven.cc/full/z8/wallhaven-z839rg.jpg
top_img: false
---
# 虎符CTF 2021 Internal System writeup

我在buuoj上复现了这个题目，顺便也在博客记录一下解题过程啦

## 题目描述

开发了一个公司内部的请求代理网站，好像有点问题，但来不及了还是先上线吧（─.─||）

**hint:** /source存在源码泄露；/proxy存在ssrf

## 做题过程

首先我们访问一下页面看看

![image-20220309211531121](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220309211531121.png)

发现是个登录窗口，如果我们直接登录提示不是admin，如果使用admin账户登录则重定向到登录页面

![image-20220309211937373](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220309211937373.png)

在注释中发现给出了源码，那我们审计一下js代码看看

首先我们肯定是要绕过登录，所以说我们先看登录部分的逻辑

```js
const salt = 'nooooooooodejssssssssss8_issssss_beeeeest'
const adminHash = sha256(sha256(salt + 'admin') + sha256(salt + 'admin'))

router.get('/login', (req, res, next) => {
  const {username, password} = req.query;

  if(!username || !password || username === password || username.length === password.length || username === 'admin') {
    res.render('login')
  } else {
    const hash = sha256(sha256(salt + username) + sha256(salt + password))

    req.session.admin = hash === adminHash

    res.redirect('/index')
  }
})

router.get('/index', (req, res, next) => {
  if(req.session.admin === undefined || req.session.admin === null) {
    res.redirect('/login')
  } else {
    res.render('index', {admin: req.session.admin, network: JSON.stringify(require('os').networkInterfaces())})
  }
})
```



密码和用户名不为空，密码不等于用户名并且用户名不能为明文admin就会进入else，我们当然可以轻松进入else，但是进入后它会提示你不是admin，因为他拿`sha256(sha256(salt + username) + sha256(salt + password))`和`sha256(sha256(salt + 'admin') + sha256(salt + 'admin'))`做比较，那他就是需要用户名和密码都是admin，所以说我们需要绕过用户名不能是明文admin并且密码和用户名相同的这两个限制。

**这里采用JS弱类型判断的特点，我们需要传入数组类型的username，这样。 JavaScript 的数组在使用加号拼接的时候最终还是会得到一个字符串（string），于是不会影响 sha256 的处理，但是却可以绕过username===admin的限制**

![image-20220309213115141](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220309213115141.png)

payload：

```url
http://cb1a412a-4a0a-457d-8695-88f035f3655c.node4.buuoj.cn:81/login
?username[]=admin
&password=admin
```

![image-20220309213328514](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220309213328514.png)



根据源码初步判断是SSRF漏洞（题目描述是参考别人wp的时候才找到的buu上是没有提供的）

我们来看看SSRF部分的代码

```js
const WAF_LISTS = [OTHER_WAF, SSRF_WAF, FLAG_WAF]

function SSRF_WAF(url) {
  const host = new UrlParse(url).hostname.replace(/\[|\]/g, '')

  return isIp(host) && IP.isPublic(host)
}

function FLAG_WAF(url) {
  const pathname = new UrlParse(url).pathname
  return !pathname.startsWith('/flag')
}

function OTHER_WAF(url) {
  return true;
}

router.get('/proxy', async(req, res, next) => {
  if(!req.session.admin) {
    return res.redirect('/index')
  }
  const url = decodeURI(req.query.url);

  console.log(url)

  const status = WAF_LISTS.map((waf)=>waf(url)).reduce((a,b)=>a&&b)

  if(!status) {
    res.render('base', {title: 'WAF', content: "Here is the waf..."})
  } else {
    try {
      const response = await axios.get(`http://127.0.0.1:${port}/search?url=${url}`)
      res.render('base', response.data)
    } catch(error) {
      res.render('base', error.message)
    }
  }
})
```

简单尝试了一下，就是输入url后submit，然后会跳转到proxy，search的话会提示你本地无法访问只能用proxy访问，那SSRF就是服务端请求伪造（Server Side Request Forgery, SSRF）指的是攻击者在未能取得服务器所有权限时，利用服务器漏洞以服务器的身份发送一条构造好的请求给服务器所在内网。 SSRF攻击通常针对外部网络无法直接访问的内部系统。我们要做的就是伪造请求绕过waf，去访问到flag。

首先/proxy接受一个url格式的参数，然后接收参数的点一般就是漏洞产生的地方，我们需要在这里伪造请求。

SSRF_WAF意思是host为公网ip，FLAG_WAF是访问目录不能以flag开头

首先因为127.0.0.1被拦截，我们需要访问0.0.0.0，这两个ip都表示本地服务器，然后Node.js的服务默认在3000端口启动，于是访问`http://0.0.0.0:3000`

![image-20220309214343149](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220309214343149.png)

然后这个时候我们就已经SSRF了，因为0.0.0.0:3000是内网的一个端口但是我们成功访问了，这个时候我们就从proxy代理到search，可以去访问search了，因为search要求我们从本地访问，那我们需要通过search这个路由看看能不能访问/flag

![image-20220309214942793](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220309214942793.png)

```js
router.post('/proxy', async(req, res, next) => {
  if(!req.session.admin) {
    return res.redirect('/index')
  }
  // test url
  // not implemented here
  const url = "https://postman-echo.com/post"
  await axios.post(`http://127.0.0.1:${port}/search?url=${url}`)
  res.render('base', "Something needs to be implemented")
})

```

这边直接访问search发现80端口拒绝了，80是http的端口，然后仔细看了看代码，我们需要传入一个代理访问的url

payload:`http://cb1a412a-4a0a-457d-8695-88f035f3655c.node4.buuoj.cn:81/proxy?url=http://0.0.0.0:3000/search?url=http://127.0.0.1:3000/flag`

![image-20220309215143627](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220309215143627.png)

这里我们就成功访问了flag文件，然后提示我们，netflix conductor server部署在了内网

我们首先需要找到这个服务被部署在了内网的哪个机器上，然后才能对这个服务进行攻击对吧，而且一开始登录进去的地方是给出了内网的网段，所以我们需要使用intruder扫一下看看。

