---
title: 浅谈jsfuck编码
date: 2022-04-28 19:49:00
updated: 2022-04-28 22:46:00
tags: [javascript,security,ctf]
description: 基于隐式类型转换的jsfuck编码
---

>参考:
>
><https://zh.wikipedia.org/wiki/JSFuck>
>
><https://blog.csdn.net/qq_36539075/category_7591719.html>
>
><https://chinese.freecodecamp.org/news/javascript-implicit-type-conversion/>

## 前言

第一次看到jsfuck编码还是在hgame 2022的week1中一个[在线小游戏](https://ek1ng.com/HGAME%202022%20week1%20writeup.html#Tetris-plus)的ctf题目，只是在注释中发现了一段有很多[]的内容，搜索后发现是jsfuck编码，在控制台中执行就得到了flag，其实也不是很懂jsfuck的原理，只是单纯觉得比较神奇，今天又在一篇安全相关的文章中看到了[jsfuck cheat sheet](https://websec.readthedocs.io/zh/latest/language/javascript/jsfuck.html)于是就想深入了解一下jsfuck。

## 什么是jsfuck

先贴一段wiki对jsfuck的解释，**JSFuck**是一种[深奥](https://zh.wikipedia.org/wiki/深奥的编程语言)的 [JavaScript](https://zh.wikipedia.org/wiki/JavaScript) [编程](https://zh.wikipedia.org/wiki/编程)风格。以这种风格写成的[代码](https://zh.wikipedia.org/wiki/代码)中仅使用 `[`、`]`、`(`、`)`、`!` 和 `+` 六种[字符](https://zh.wikipedia.org/wiki/字符)。此编程风格的[名字](https://zh.wikipedia.org/wiki/名字)[派生](https://zh.wikipedia.org/wiki/衍生)自仅使用较少符号写代码的[Brainfuck](https://zh.wikipedia.org/wiki/Brainfuck)语言。与其他深奥的编程语言不同，以JSFuck风格写出的代码不需要另外的[编译器](https://zh.wikipedia.org/wiki/编译器)或[解释器](https://zh.wikipedia.org/wiki/解释器)来执行，无论[浏览器](https://zh.wikipedia.org/wiki/浏览器)或[JavaScript引擎](https://zh.wikipedia.org/wiki/JavaScript引擎)中的原生 JavaScript 解释器皆可直接运行。鉴于 JavaScript 是[弱类型语言](https://zh.wikipedia.org/wiki/強弱型別)，编写者可以用数量有限的字符重写 JavaScript 中的所有功能，且可以用这种方式执行任何类型的表达式。

## 为什么可以只用6种字符编码js

显然一段js代码只用6种字符去写，毫无疑问可以绕过很多关键词的过滤，但是在使用之前会有一个疑问就是为什么可以这样编码一段js代码。

![image-20220428220721675](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220428220721675.png)

其实这是由于js的隐式类型转换导致的，对于逻辑表达式`[]==![]`，js会返回true

先来说说对于运算符`==`的运算规则

- `NaN`和其他任何类型比较永远返回`false`（包括和他自己）`NaN == NaN // false`
- `Boolean` 和其他任何类型比较，Boolean 首先被转换为 Number 类型。`true == '2'  // false true -> 1`
- `String`和`Number`比较，先将`String`转换为`Number`类型。`123 == '123' // true, '123' -> 123`
- `null == undefined`比较结果是`true`，除此之外，`null`、`undefined`和其他任何结果的比较值都为`false`。
- `原始类型`和`引用类型`做比较时，引用类型会依照`ToPrimitive`规则转换为原始类型。

>⭐️`ToPrimitive`规则，是引用类型向原始类型转变的规则，它遵循先`valueOf`后`toString`的模式期望得到一个原始类型。如果还是没法得到一个原始类型，就会抛出 `TypeError`。

我们再来看看`[]==![]`，取非运算优先级最高，`[]`是空数组，对一个空数组取非得到`false`，那么`[]==![]` 会变成 `[]==false`，然后因为`Boolean` 和其他任何类型比较，Boolean 首先被转换为 Number 类型，`false -> 0`，变成`[]==0`，然后再因为`原始类型`和`引用类型`做比较时，引用类型会依照`ToPrimitive`规则转换为原始类型，`[].valueOf().toString()`的值为`''`，`String`和``Number`比较会把`String`转换成`Number`，也就是`'' -> 0`，最终变成`0==0`得到true。

看明白上面的例子后会发现，js的隐式类型转换使得我们可以组合上述的6个符号来替换一些我们希望得到的字符，符号等等内容，所以这也是为什么js代码可以用这样简单的6个符号用来表达，当然这么说其实也还是很难有信服力，我们接下来解析一下如何使用jsfuck编码`alert()`。

数字的话比较简单有1有0我们一直加想要多少就多少，然后我们肯定需要26个字符对吧，那字符的话其实是用截取的方式，`alert`的话我们可以用`true`和`false`截取就可以实现

```js

'a' == 'false'[1] == (false + '')[1] == (![]+[])[+!+[]]
'l' == 'false'[2] ==  (false + '')[2] == (![]+[])[!+[]+!+[]]
'e' == 'true'[0] == (true + '')[3] == (!![]+[])[!+[]+!+[]+!+[]]
'r' == 'true'[0] == (true + '')[1] == (!![]+[])[+!+[]] 
't' == 'true'[0] == (true + '')[0] == (!![]+[])[+[]]
'alert' == (![]+[])[+!+[]] + (![]+[])[!+[]+!+[]] + (!![]+[])[!+[]+!+[]+!+[]] + (!![]+[])[+!+[]] + (!![]+[])[+[]]
```

![image-20220428223328684](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220428223328684.png)

但是26个字母其他的又该怎么实现呢，我们这里看一下[jsfuck的官方仓库](https://github.com/aemkei/jsfuck/blob/master/jsfuck.js)中是如何获取指定字符的，那更多的字符就需要去看jsfuck的源码啦

```javascript
    'a':   '(false+"")[1]',
    'b':   '([]["entries"]()+"")[2]',
    'c':   '([]["flat"]+"")[3]',
    'd':   '(undefined+"")[2]',
    'e':   '(true+"")[3]',
    'f':   '(false+"")[0]',
    'g':   '(false+[0]+String)[20]',
    'h':   '(+(101))["to"+String["name"]](21)[1]',
    'i':   '([false]+undefined)[10]',
    'j':   '([]["entries"]()+"")[3]',
    'k':   '(+(20))["to"+String["name"]](21)',
    'l':   '(false+"")[2]',
    'm':   '(Number+"")[11]',
    'n':   '(undefined+"")[1]',
    'o':   '(true+[]["flat"])[10]',
    'p':   '(+(211))["to"+String["name"]](31)[1]',
    'q':   '("")["fontcolor"]([0]+false+")[20]',
    'r':   '(true+"")[1]',
    's':   '(false+"")[3]',
    't':   '(true+"")[0]',
    'u':   '(undefined+"")[0]',
    'v':   '(+(31))["to"+String["name"]](32)',
    'w':   '(+(32))["to"+String["name"]](33)',
    'x':   '(+(101))["to"+String["name"]](34)[1]',
    'y':   '(NaN+[Infinity])[10]',
    'z':   '(+(35))["to"+String["name"]](36)',

```

## 利用jsfuck绕过过滤

​  因为jsfuck只用6个字符编码，这样的话很多时候可以绕过一些对恶意代码的检测，看到渗透测试中大多数对jsfuck的运用就是绕XSS的过滤，比如[这篇文章](https://www.anquanke.com/post/id/188836)，文章中xss的payload结合jsfuck编码绕过对`alert`等一些危险函数的利用。ctf中我也没怎么看到过jsfuck绕过过滤的题目，只看到过一段js代码里面就直接是flag的题，因此也没有机会仔细研究一下，也许明年hgame可以出个有意思的题考考新生hhh。
