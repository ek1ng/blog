---
title: CSAPP lab0 实验环境搭建
date: 2022-03-28 11:55:00
updated: 2022-03-28 11:55:00
tags: [CSAPP,计算机组成原理]
description: CMU 15-213 lab0 实验环境搭建
archive: true
---

# CSAPP Lab0 实验环境搭建

这是csapp lab开始的第一步，就是搭建实验环境。

## linux环境

>
>
>参考:https://blog.csdn.net/aawoe/article/details/107104947

​		实验需要使用unix环境，所以如果是windows操作系统，那么可以用docker，vmware/vitrual box，wsl等方式搭建linux环境，我本人使用的是vmware，安装的linux发行版是manjaro，由于在做lab之前就已经配置好了虚拟机环境，因此就不记录搭建linux的环境过程了。

​		以及不要忘记安装c语言的编译器gcc，debug工具gdb。

## 实验文件下载

官网实验文件:http://csapp.cs.cmu.edu/3e/labs.html

github仓库实验文件:https://github.com/Hansimov/csapp/tree/master/_labs

需要注意官网实验文件的下载需要教师账户，所以说推荐从github仓库下载。

实验材料中包含四个文件，README，Writeup，Release Notes，Self-Study Handout，README和Writeup对自学者来说主要是帮助理解讲解内容，Self-Study Handout是实验内容，Release Notes是版本历史，所有实验都是给出这四个文件的，我认为很多时候英文课程带来的语言环境差异一定程度让实验更难，但是国内很多翻译版等等那些资料也并不咋样，尽可能看英文会比较好，我在我的github仓库中也会提供了实验文件供下载，从csapp的官网下和从github仓库下都一样，这些实验最近一次更新文件是19年，所以不用担心文件会不会有什么版本问题，接下来就可以开始愉快的做lab啦！