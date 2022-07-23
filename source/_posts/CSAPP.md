---
title: CSAPP lab
date: 2022-03-21 20:59:00
updated: 2022-03-21 20:59:00
tags: [CSAPP,计算机组成原理]
description: CMU 15-213 lab
pin: false
archive: true
---

# CSAPP学习过程
这篇文章主要记录CSAPP书和lab的学习过程，具体某个lab的踩坑过程会分别附单独链接，本文主要是记录漫长的学习过程以及方便想学但是尚未开始学习的同学参考，以下是github的lab代码仓库。
{% ghcard ek1ng/CSAPPlab %}

## Todo
- [x] Bilibili翻译课程 lecture 1-4
- [x] 搭建实验环境
- [x] Data lab
- [ ] Bilibili翻译课程 lecture 5-9
- [ ] Bomb lab
- [ ] Attack lab

## 前置材料
一本CSAPP
[CSAPP的bilibili翻译课程](https://www.bilibili.com/video/BV1iW411d7hd?from=search&seid=14100643096477102310&spm_id_from=333.337.0.0)
[实验材料](http://csapp.cs.cmu.edu/3e/labs.html)
[参考经验贴1](https://hansimov.gitbook.io/csapp/publish-info/about-authors) 
[参考经验贴2](https://www.zhihu.com/question/20402534)

## 我想做些什么
开设这个仓库是想记录我做CSAPP的lab的过程，也顺便将踩坑过程分享，帮助后人少走弯路，我预期跟着bilibili的翻译课程完成CSAPP的课本内容学习并且完成CSAPP的lab，我会在README中精简的记录我的学习过程，并且以章节为单位将详细的笔记发在我的博客上，当然以笔记形式的内容并不一定适合别人参考，参考的人们应该更需要一些精简的学习过程和汇集好的材料以及我具体的实验代码，当你发现其中某部分可能对你有用的时候，自然会去博客中看详细的学习过程，这应该是一个不错的分享方式，所以推荐结合博客和仓库使用。

## 学习过程（以Lab为单位总结）
简单查阅别的学习经验后，大多数人的分享都说看书再多遍也不如做lab学到的多，lab是课程的精髓，我已经粗略的学过编译原理，计算机组成原理和操作系统，所以我会比较快速的过一遍网课然后开始lab，目标3个月完成大多数的lab（也许有一些实在不感兴趣的lab会跳过）

## Timeline
{% timeline %}
{% timenode 2022-03-30 %}

完成[Datalab](https://ek1ng.com/2022/03/30/CSAPPlab1/)

{% endtimenode %}

{% timenode 2022-03-28 %}

完成[实验环境搭建](https://ek1ng.com/2022/03/28/CSAPPlab0/)

{% endtimenode %}

{% timenode 2022-03-27 %}

完成lecture04 floats，主要内容是浮点数，包括IEEE754的浮点数表示方法和设计原理，浮点数的运算，舍入方法，C语言对浮点数的设计，大概这些内容，到这里信息表示与处理这一章节就学完了，接下来会开始做data lab。

{% endtimenode %}

{% timenode 2022-03-25 %}

完成lecture03 Bits Bytes and integer，主要内容是整数运算和信息存储，包括机器字长，大小端，整数加减乘除运算与溢出等内容，感觉课堂习题的例子非常不错，对整数运算与溢出有了更深刻的了解。

{% endtimenode %}

{% timenode 2022-03-24 %}

完成lecture02 Bits Bytes and integer，主要内容是信息存储和整数表示，包括C语言的有无符号数转化，符号位扩展，截断等

{% endtimenode %}

{% timenode 2022-03-19 %}

完成lecture01 cource overview

{% endtimenode %}

{% timenode 2022-03-16 %}

决定开始学习并且简单的编写README

{% endtimenode %}

{% endtimeline %}



