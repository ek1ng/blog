---
title: js混淆与反混淆
date: 2023-02-05 11:36:00
updated: 2023-02-05 18:15:00
tags: [ctf,security]
description: js的混淆与反混淆

---

### 为什么要进行混淆

由于设计原因，前端的js代码是可以在浏览器访问到的，那么因为需要让代码不被分析和复制从而导致更多安全问题，所以我们要对js代码进行混淆。

### JS混淆和反混淆常见思路

在了解了js代码的执行过程后，我们来看如何对js进行混淆。可以想到比如我们想实现一个js混淆器我们该怎么做呢，要不就是用正则替换，要不就是在AST阶段生成混淆代码，用正则替换实现简单但是效果也比较差，现在js混淆大多数都是在不改变AST的情况下去生成混淆后的代码，保证代码的功能不变又能够做到混淆的目的。

#### 代码压缩

压缩js代码不用多说，就是去除空格，换行符等等，让代码变成一坨甚至一行。

#### 代码混淆

这里我们抛砖引玉，讲一些比较常见的混淆方式，实际上混淆的办法非常的多。

Javascript 提供了将字符串当做代码执行（evaluate）的能力，可以通过 Function 构造器、eval、setTimeout、setInterval 将字符串传递给 js 引擎进行解析执行。

- 访问成员变量的方法

js中可以通过window.eval()访问windows对象的eval方法，也可以用window['eval']来访问

- 变量名混淆（将变量名变成一些无意义的可以来较乱的字符串（16进制）降低代码的可读性）

```js
// let str = 'eval'
let _0xfg31e = 'eval'
```

- 字符串混淆（进行加密或者是编码，目的：确保代码里面，不可以使用搜索的方式来查到原始的字符串）

```js
// let str = 'eval'
let str = 'e'+'v'+'a'+'l'//拼接
```

- 进制/编码

```js
// let str = 'eval'
let str = '\u0065\u0076\u0061\u006c'//unicode编码
let str = 14..toString(15) + 31..toString(32) + 0xf1.toString(22)//利用toStirng()
```

- 利用数组进行拆分

```js
// console.log(new window.Date().getTime())  
var arr = ['log','Date','getTime']
console[arr[0]](new window[arr[1]]()[arr[2]]())
```

```js
14..toString(15) + 31..toString(32) + 0xf1.toString(22)
```

- 常量改算术表达式

```js
// var num = 1234
var num = 602216 ^ 603322 
```

- 算术表达式改函数调用表达式

```js
// var num = 602216 ^ 603322 
var a = function (s, h) {
    return s ^ h;
}(602216, 603322)
```

在全是特性的js中这种转换的方式非常的多，用几种就会让代码变得完全看不懂。

- 反调试

  - 禁止 debugger

  写个定时器死循环来禁止调试

  ```js
  function debug() {
      debugger;
      setTimeout(debug, 1);
  }
  debug();
  ```

  这个可以把调用debug()的部分注释掉，毕竟js在前端啥都能改

  - 清空控制台

  ```js
  function clear() {
      console.clear();
      setTimeout(clear, 10);
  }
  clear();
  ```

  没什么用，调试时可以直接查看变量，也可以用上面的办法绕过

  - 检测函数、对象属性修改

  攻击者在调试的时，经常会把防护的函数删除，或者把检测数据对象进行篡改。可以检测函数内容，在原型上设置禁止修改。

  ```js
  // eval函数
  function eval() {
      [native code]
  }
  
  //使用eval.toString进行内容匹配”[native code]”，可以轻易饶过
  window.eval = function(str){
          /*[native code]*/
          //[native code]
          console.log("[native code]");
      };
  
  //对eval.toString进行全匹配，通过重写toString就可以绕过
  window.eval = function(str){
          //....
      };
      window.eval.toString = function(){
          return `function eval() {
              [native code]
          }`
      };
  
  //检测eval.toString和eval的原型
  function hijacked(fun){
          return "prototype" in fun || fun.toString().replace(/\n|\s/g, "") != "function"+fun.name+"(){[nativecode]}";
      }
  ```

### 一些反混淆技巧

最重要的就是耐心，F12打断掉，然后用console.log之类的方法一步一步去看，因为不论怎么混淆并不改变代码本身的逻辑，大多数都是可以慢慢调还原出来的。

### F12的小技巧

第一个是这里有一个搜索功能可以对静态文件做全局的检索，在找一些特定功能块时会有用

![image-20230205171425181](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051714224.png)

第二个是替换功能，开启本地替换后，可以直接编辑源代码中的内容并且保存，文件会被存到替换文件里面，就可以随意的在前端做一些修改了。

![image-20230205171607007](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051716053.png)

### 常见的混淆/反混淆工具

### 亲手尝试反混淆

#### HGAME 2023 Week1 Classic Childhood Game

当然直接执行mota()就能出，但是我们来尝试一下通过调试反混淆这段代码，看看是什么逻辑。

https://www.json.cn/json/jshx.html 开方法变量重命名 字符串加密 重排字符串 Base64编码字符串 Unicode转义生成的混淆代码。

```js
function mota() {
  var a = ['\x59\x55\x64\x6b\x61\x47\x4a\x58\x56\x6a\x64\x61\x62\x46\x5a\x31\x59\x6d\x35\x73\x53\x31\x6c\x59\x57\x6d\x68\x6a\x4d\x6b\x35\x35\x59\x56\x68\x43\x4d\x45\x70\x72\x57\x6a\x46\x69\x62\x54\x55\x31\x56\x46\x52\x43\x4d\x46\x6c\x56\x59\x7a\x42\x69\x56\x31\x59\x35'];
  (function (b, e) {
    var f = function (g) {
      while (--g) {
        b['push'](b['shift']());
      }
    };
    f(++e);
  }(a, 0x198));
  var b = function (c, d) {
    c = c - 0x0;
    var e = a[c];
    if (b['CFrzVf'] === undefined) {
      (function () {
        var g;
        try {
          var i = Function('return\x20(function()\x20' + '{}.constructor(\x22return\x20this\x22)(\x20)' + ');');
          g = i();
        } catch (j) {
          g = window;
        }
        var h = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/=';
        g['atob'] || (g['atob'] = function (k) {
          var l = String(k)['replace'](/=+$/, '');
          var m = '';
          for (var n = 0x0, o, p, q = 0x0; p = l['charAt'](q++); ~p && (o = n % 0x4 ? o * 0x40 + p : p, n++ % 0x4) ? m += String['fromCharCode'](0xff & o >> (-0x2 * n & 0x6)) : 0x0) {
            p = h['indexOf'](p);
          }
          return m;
        });
      }());
      b['fqlkGn'] = function (g) {
        var h = atob(g);
        var j = [];
        for (var k = 0x0, l = h['length']; k < l; k++) {
          j += '%' + ('00' + h['charCodeAt'](k)['toString'](0x10))['slice'](-0x2);
        }
        return decodeURIComponent(j);
      };
      b['iBPtNo'] = {};
      b['CFrzVf'] = !![];
    }
    var f = b['iBPtNo'][c];
    if (f === undefined) {
      e = b['fqlkGn'](e);
      b['iBPtNo'][c] = e;
    } else {
      e = f;
    }
    return e;
  };
  alert(atob(b('\x30\x78\x30')));
}
```

首先定义了一个变量a，然后a是一段base64编码后的内容，并且又被unicode编码了，可以在控制台console.log这段内容，在动态调试的过程中我们也能够看到。

![image-20230205160610344](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051606373.png)

然后我们继续步进，可以看出这里其实是一个多次base64解码的过程![image-20230205160739247](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051607312.png)

![image-20230205161011881](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051610944.png)

这里e的值为`aGdhbWV7ZlVubnlKYXZhc2NyaXB0JkZ1bm55TTB0YUc0bWV9`

函数的返回值是e然后atob() base64解码一层出来就是flag了

#### 逆一下百度翻译的接口

> https://fanyi.baidu.com/

![image-20230205163106928](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051631005.png)

这里有一个叫做`langdetect`的接口来探测语言

![image-20230205163406389](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051634448.png)

然后有接口`https://fanyi.baidu.com/v2transapi`,是个POST请求数据都在表单里面。

![image-20230205163211795](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051632838.png)

这里利用了`sign`和`token`做了一些防止风控的策略，来看看`sign`和`token`是怎么生成的。

我们尝试多次翻译，发现`token`一直不变，然后直接拿`token`的值搜索，发现是在请求静态资源的时候，就会被硬编码进去。

![image-20230205163926748](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051639805.png)

`sign`的值在多次翻译的过程中发生了变化，那么我们来看看js是怎么生成`sign`的。

![image-20230205172055297](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051720347.png)

全局搜索源代码，发现index.dc84f2b3.js中有出现，其他地方看起来不太像，所以仔细看看这个文件，并且尝试打断点看看能不能断下来。

总共5个地方出现了，都打断点进行尝试，发现在21909行这个地方可以断下来

![image-20230205172321212](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051723281.png)

那么我们就来看看这个b(e)函数做了什么事情，将鼠标放在函数上方可以看到这个函数被引用的位置，我们可以发现传入的参数`e`是我们想要翻译的内容，那看来是根据要翻译的内容动态生成了一个`sign`用来签名。

我们跟进这个b(e)看一下

![image-20230205173224540](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051732618.png)

这个函数主题的逻辑就是根据我们传入的这个字符串来进行一些位运算，最终能够得到一个`sign`

找到这部分逻辑后我们可以把生成`sign`的部分代码从混淆的函数中抽离出来了

```js
var e = 'Hello World'

function b(t) {
            var o, i = t.match(/[\uD800-\uDBFF][\uDC00-\uDFFF]/g);
            if (null === i) {
                var a = t.length;
                a > 30 && (t = "".concat(t.substr(0, 10)).concat(t.substr(Math.floor(a / 2) - 5, 10)).concat(t.substr(-10, 10)))
            } else {
                for (var s = t.split(/[\uD800-\uDBFF][\uDC00-\uDFFF]/), c = 0, u = s.length, l = []; c < u; c++)
                    "" !== s[c] && l.push.apply(l, function(t) {
                        if (Array.isArray(t))
                            return e(t)
                    }(o = s[c].split("")) || function(t) {
                        if ("undefined" != typeof Symbol && null != t[Symbol.iterator] || null != t["@@iterator"])
                            return Array.from(t)
                    }(o) || function(t, n) {
                        if (t) {
                            if ("string" == typeof t)
                                return e(t, n);
                            var r = Object.prototype.toString.call(t).slice(8, -1);
                            return "Object" === r && t.constructor && (r = t.constructor.name),
                            "Map" === r || "Set" === r ? Array.from(t) : "Arguments" === r || /^(?:Ui|I)nt(?:8|16|32)(?:Clamped)?Array$/.test(r) ? e(t, n) : void 0
                        }
                    }(o) || function() {
                        throw new TypeError("Invalid attempt to spread non-iterable instance.\nIn order to be iterable, non-array objects must have a [Symbol.iterator]() method.")
                    }()),
                    c !== u - 1 && l.push(i[c]);
                var p = l.length;
                p > 30 && (t = l.slice(0, 10).join("") + l.slice(Math.floor(p / 2) - 5, Math.floor(p / 2) + 5).join("") + l.slice(-10).join(""))
            }
            for (var d = "".concat(String.fromCharCode(103)).concat(String.fromCharCode(116)).concat(String.fromCharCode(107)), h = (null !== r ? r : (r = window[d] || "") || "").split("."), f = Number(h[0]) || 0, m = Number(h[1]) || 0, g = [], y = 0, v = 0; v < t.length; v++) {
                var _ = t.charCodeAt(v);
                _ < 128 ? g[y++] = _ : (_ < 2048 ? g[y++] = _ >> 6 | 192 : (55296 == (64512 & _) && v + 1 < t.length && 56320 == (64512 & t.charCodeAt(v + 1)) ? (_ = 65536 + ((1023 & _) << 10) + (1023 & t.charCodeAt(++v)),
                g[y++] = _ >> 18 | 240,
                g[y++] = _ >> 12 & 63 | 128) : g[y++] = _ >> 12 | 224,
                g[y++] = _ >> 6 & 63 | 128),
                g[y++] = 63 & _ | 128)
            }
            for (var b = f, w = "".concat(String.fromCharCode(43)).concat(String.fromCharCode(45)).concat(String.fromCharCode(97)) + "".concat(String.fromCharCode(94)).concat(String.fromCharCode(43)).concat(String.fromCharCode(54)), k = "".concat(String.fromCharCode(43)).concat(String.fromCharCode(45)).concat(String.fromCharCode(51)) + "".concat(String.fromCharCode(94)).concat(String.fromCharCode(43)).concat(String.fromCharCode(98)) + "".concat(String.fromCharCode(43)).concat(String.fromCharCode(45)).concat(String.fromCharCode(102)), x = 0; x < g.length; x++)
                b = n(b += g[x], w);
            return b = n(b, k),
            (b ^= m) < 0 && (b = 2147483648 + (2147483647 & b)),
            "".concat((b %= 1e6).toString(), ".").concat(b ^ f)
        }
console.log(b(e))
```

本地运行发现缺少`r`

![image-20230205174014542](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051740581.png)

回到页面开始动调，看看r是什么

![image-20230205174918209](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051749251.png)

这里是访问了window对象的d属性，d的值为'gtk'，访问的结果是`320305.131321201`，那么所以我们本地的node环境是访问不到window[]的，那就直接把值硬编码进去。

![image-20230205175629597](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051756670.png)

又发现`n`不存在，那么我们继续动调

![image-20230205175732219](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051757279.png)

步进看看

![image-20230205175810968](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051758028.png)

给脚本补上n函数再试试，这个n函数看起来也是做了一些加密运算的工作，我们主要的目标是通过动态调试和一些反混淆的手段去看清楚整体的逻辑，而并不是像逆向一样去通过一个`sign`的结果看着这个加密方式我逆出来我要输入什么才是这个`sign`。

```js
var e = 'Hello World'

function b(t) {
    var o, i = t.match(/[\uD800-\uDBFF][\uDC00-\uDFFF]/g);
    if (null === i) {
        var a = t.length;
        a > 30 && (t = "".concat(t.substr(0, 10)).concat(t.substr(Math.floor(a / 2) - 5, 10)).concat(t.substr(-10, 10)))
    } else {
        for (var s = t.split(/[\uD800-\uDBFF][\uDC00-\uDFFF]/), c = 0, u = s.length, l = []; c < u; c++)
            "" !== s[c] && l.push.apply(l, function (t) {
                if (Array.isArray(t))
                    return e(t)
            }(o = s[c].split("")) || function (t) {
                if ("undefined" != typeof Symbol && null != t[Symbol.iterator] || null != t["@@iterator"])
                    return Array.from(t)
            }(o) || function (t, n) {
                if (t) {
                    if ("string" == typeof t)
                        return e(t, n);
                    var r = Object.prototype.toString.call(t).slice(8, -1);
                    return "Object" === r && t.constructor && (r = t.constructor.name),
                        "Map" === r || "Set" === r ? Array.from(t) : "Arguments" === r || /^(?:Ui|I)nt(?:8|16|32)(?:Clamped)?Array$/.test(r) ? e(t, n) : void 0
                }
            }(o) || function () {
                throw new TypeError("Invalid attempt to spread non-iterable instance.\nIn order to be iterable, non-array objects must have a [Symbol.iterator]() method.")
            }()),
                c !== u - 1 && l.push(i[c]);
        var p = l.length;
        p > 30 && (t = l.slice(0, 10).join("") + l.slice(Math.floor(p / 2) - 5, Math.floor(p / 2) + 5).join("") + l.slice(-10).join(""))
    }
    for (var d = "".concat(String.fromCharCode(103)).concat(String.fromCharCode(116)).concat(String.fromCharCode(107)), h = (r = "320305.131321201").split("."), f = Number(h[0]) || 0, m = Number(h[1]) || 0, g = [], y = 0, v = 0; v < t.length; v++) {
        var _ = t.charCodeAt(v);
        _ < 128 ? g[y++] = _ : (_ < 2048 ? g[y++] = _ >> 6 | 192 : (55296 == (64512 & _) && v + 1 < t.length && 56320 == (64512 & t.charCodeAt(v + 1)) ? (_ = 65536 + ((1023 & _) << 10) + (1023 & t.charCodeAt(++v)),
            g[y++] = _ >> 18 | 240,
            g[y++] = _ >> 12 & 63 | 128) : g[y++] = _ >> 12 | 224,
            g[y++] = _ >> 6 & 63 | 128),
            g[y++] = 63 & _ | 128)
    }
    for (var b = f, w = "".concat(String.fromCharCode(43)).concat(String.fromCharCode(45)).concat(String.fromCharCode(97)) + "".concat(String.fromCharCode(94)).concat(String.fromCharCode(43)).concat(String.fromCharCode(54)), k = "".concat(String.fromCharCode(43)).concat(String.fromCharCode(45)).concat(String.fromCharCode(51)) + "".concat(String.fromCharCode(94)).concat(String.fromCharCode(43)).concat(String.fromCharCode(98)) + "".concat(String.fromCharCode(43)).concat(String.fromCharCode(45)).concat(String.fromCharCode(102)), x = 0; x < g.length; x++)
        b = n(b += g[x], w);
    return b = n(b, k),
        (b ^= m) < 0 && (b = 2147483648 + (2147483647 & b)),
        "".concat((b %= 1e6).toString(), ".").concat(b ^ f)
}

function n(t, e) {
    for (var n = 0; n < e.length - 2; n += 3) {
        var r = e.charAt(n + 2);
        r = "a" <= r ? r.charCodeAt(0) - 87 : Number(r),
            r = "+" === e.charAt(n + 1) ? t >>> r : t << r,
            t = "+" === e.charAt(n) ? t + r & 4294967295 : t ^ r
    }
    return t
}
console.log(b(e))

```



![image-20230205181040097](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051810163.png)

![image-20230205181057371](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202302051810421.png)

到这里我们就完成啦

#### Defcon 2021 Quals 的赛题  threefactooorx 

由于赛题给出的附件（一个chrome浏览器扩展）在我这里chrome109上已经跑不起来了，我用的arch的包管理搜了一下没有什么办法回滚chrome的版本，解决办法应该还是有的但是不太想大费周章再去做了，看了看[p牛的wp](https://mp.weixin.qq.com/s/kzRHdMhWgcay2UnTgsACzA)这个题目的核心就是只要会调试和反混淆js,一步一步调试就知道在做什么了。

题目的本质是给了一个称为“3FA”的chrome扩展，这个插件是用于防止网络钓鱼的。插件中的js是混淆过的，需要装上这个扩展才能使用站点的功能，站点的功能是上传HTML后会有一个Bot访问到这个页面，并且发回来访问的截图。需要通过对这个混淆的js进行调试，发现这个js中有发送消息的函数，逆出来其中的逻辑之后，制作一个用于恶意的HTML页面，Bot（相当于一个也安装了3FA插件的真人）访问后，Bot的flag就会显示在页面上，然后题目设计了一个拍照Bot访问结果的并且回显在我们页面上面的功能，这里就相当于我们通过逆向这个chrome扩展，完成了对于访问者的攻击。
