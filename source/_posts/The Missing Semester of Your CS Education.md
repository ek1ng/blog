---
title: The Missing Semester of Your CS Education shell
date: 2022-03-31 16:57:00
tags: [The Missing Semester of Your CS]
description: The Missing Semester of Your CS Education shell
---

# The Missing Semester of Your CS Education 课程概览与 shell

主要是想起来自己vim还不太会用，所以说记得这个课程的vim教学不错，干脆就花一晚上时间看看整套课程，重点看一下vim的使用，我看的版本是https://missing-semester-cn.github.io/，是社区的中文翻译版的文档，这些工具大多我都已经能够熟练使用了，所以就没去看英文的视频感觉有点浪费时间。

shell确实是经常使用，但是又不那么会使用，学一学一些不太常用的shell命令可以比较高的提升自己的效率。

首先看课程前想起来自己用的windows的powershell实在是太丑，又不能总用虚拟机里manajro的shell，wsl的话倒是没装，所以shell这个工具对我这种windows用户来说，自带powershell确实不太好用，我选择了安装Powershell7.1 + oh my posh + 主题JanDeDobbeleer，现在看起来效果还算不错。

![image-20220331204556385](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220331204556385.png)

在折腾完Powershell后，shell的课程用的是bash，那我想了想可以用git bash，于是又给git bash也配置了一下，现在已经可以在cmd中打开并且有个看起来还不错的主题啦

![image-20220331205129041](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220331205129041.png)

更换完主题心情舒畅，接下来就开始shell命令的学习叭。

简单记录一些以前不太熟悉的

- shell 中的路径是一组被分割的目录，在 Linux 和 macOS 上使用 `/` 分割，而在Windows上是 `\`。
- 当前工作目录可以使用 `pwd` 命令来获取。
- ls -a参数可以列出带.的文件，-l参数可以更详细的列出文件信息
- `cat < hello.txt > hello2.txt` cat原来不只是打印内容 而是连接文件并且打印到输出设备，这样我们可以重定向cat的输出和输入，这条命令就可以把hello.txt的内容复制给hello2.txt
- 关于 shell，有件事我们必须要知道。`|`、`>`、和 `<` 是通过 shell 执行的，而不是被各个程序单独执行。 `echo` 等程序并不知道 `|` 的存在，它们只知道从自己的输入输出流中进行读写。

知识点大概就这些，接下来有个小的lab需要完成

前面的内容比较简单

![image-20220331212013121](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220331212013121.png)

> 接下来要求给semester这个文件一行一行地输入
>
> #!/bin/sh
> curl --head --silent https://missing.csail.mit.edu

做着做着感觉对于echo和cat有点懵逼，查了一下cat输出文件内容echo，echo则是输出字符串内容，应该说这俩都是接收输入然后输出在标准输出设备上，是接收的输入不同，所以我们如果我们要直接在命令行接收字符串的输入，需要使用echo，如果要我们想接收一个文件的输入，需要使用echo，至于输出我们可以用>来对标准输出重定向，到这个semester里面。

那么简单用的话我们直接用echo 把字符串内容输出到文件里面就可以啦

首先`#!/bin/sh`的写入有点棘手， `#` 在Bash中表示注释，而 `!` 即使被双引号（`"`）包裹也具有特殊的含义。 单引号（`'`）则不一样，此处利用这一点解决输入问题。

其次如果两次都用>写入，第二次写入会覆盖第一次写入的内容，这个应该叫覆盖写，用>>追加内容就可以了

![image-20220331213455170](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220331213455170.png)

接下来的要求:

>
>
>尝试执行这个文件。例如，将该脚本的路径（`./semester`）输入到您的shell中并回车。如果程序无法执行，请使用 ls 命令来获取信息并理解其不能执行的原因。

![image-20220331213546765](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220331213546765.png)

首先执行不了按照通俗理解就是没有root呗因为root了啥都能做，那具体的我们用ls -l查看权限后 文件对于文件所有者ek1ng 用户组users 其他人的权限分别为 rw- r-- r-- ，目前用户是ek1ng，只有读和写权限，没有运行权限x，如果是rwx就可以运行啦。那这里我们就需要使用chmod提升权限，来执行文件。

![image-20220331213941138](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220331213941138.png)

使用chmod 777 增加执行权限后，就可以运行啦。

>使用 `|` 和 `>` ，将 semester 文件输出的最后更改日期信息，写入主目录下的 `last-modified.txt` 的文件中

使用管道符|实现就可以

![image-20220331214206745](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220331214206745.png)

>写一段命令来从 /sys 中获取笔记本的电量信息，或者台式机 CPU 的温度。

不知道为什么在vmware里找不到，也许是我使用的不太对吧😢