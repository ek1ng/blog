---
title: StarCTF 2022 web writeup (unfinished)
date: 2022-04-19 22:28:00
updated: 2022-04-19 22:28:00
tags: [ctf, security]
description: 在协会参加的第一场XCTF分站赛，0输出，复盘题目学习学习
---

# \*CTF 2022 writeup

这是在协会参加的第一场 XCTF 分站赛，虽然协会的师傅们把 web ak 了但是我是没有能够帮上什么忙，赛后也是补一下题目然后复盘一下。6 星的师傅们也都在 github 仓库开源了他们的题目，也是非常感谢师傅们愿意开源题目以便于复现赛题。

## oh-my-grafana

官方给的 docker 好像有点问题，还没能够在本地搭建起复现环境，有点晚了明天继续看看

## **oh-my-notepro**

## **oh-my-lotto & revenge**

## TODAY

这是一道社工题目，也算是第一次遇到吧，长个见识。

> “I’m anninefour. I love machine learning and data science.
> Flag is in my pocket!”

根据`I love machine learning and data science.`，kaggle 的介绍`Kaggle: Your Machine Learning and Data Science Community`，在 kaggle 上搜索 anninefour 可以搜到用户

![image-20220419201301902](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220419201301902.png)

发现简介部分@1liujing

在推特上搜索 1liujing 找到对应用户

![image-20220419201447061](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220419201447061.png)

![img](https://pbs.twimg.com/media/FQH5aGcaIAM450H?format=jpg&name=900x900)

去 google map 搜一下，从图中 x 夫果品生鲜超市和花山 xx 可以搜到花山名苑，在评论区可以发现 flag

![image-20220419202143964](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220419202143964.png)

> \*CTF{aGFwcHlsb2NrZG93bg==}
