---
title: XSLeaks 侧信道攻击 (unfinished)
date: 2022-03-14 22:46:00
tags: [ctf,security]
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

这题在BUUOJ上有复现环境



## SUSCTF 2022 ez_note



## TQLCTF 2022 A More Secure Pastebin

