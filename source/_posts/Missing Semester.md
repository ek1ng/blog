---
title: The Missing Semester of Your CS Education
date: 2022-04-01 13:48:00
updated: 2022-04-07 13:48:00
tags: [The Missing Semester of Your CS]
description: 计算机教育中缺失的一课
---
主要是想起来自己vim还不太会用，所以说记得这个课程的vim教学不错，干脆就花时间看看整套课程，重点看一下vim的使用，我看的版本是社区的[中文翻译版](https://missing-semester-cn.github.io/)的文档，这些工具大多我都已经能够熟练使用了，所以就没去看英文的视频感觉有点浪费时间。

## shell

首先的话shell在这个课程的第一课和第二课都讲，但是因为内容一样，所以说就并在一起写了。

### 课程概览与 shell

#### 课程内容

shell确实是经常使用，但是又不那么会使用，学一学一些不太常用的shell命令可以比较高的提升自己的效率。

看课程前想起来自己用的windows的powershell实在是太丑，又不能总用虚拟机里manajro的shell，wsl的话倒是没装，所以shell这个工具对我这种windows用户来说，自带powershell确实不太好用，我选择了安装Powershell7.1 + oh my posh + 主题JanDeDobbeleer，现在看起来效果还算不错。

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

#### 课后练习

前面的内容比较简单

![image-20220331212013121](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220331212013121.png)

> 接下来要求给semester这个文件一行一行地输入
>
> #!/bin/sh
> curl --head --silent <https://missing.csail.mit.edu>

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

### Shell 工具和脚本

#### 课程内容

##### 变量

挺神奇的，`foo = bar` （使用空格隔开）是不能正确工作的，因为解释器会调用程序`foo` 并将 `=` 和 `bar`作为参数。 在shell脚本中使用空格会起到分割参数的作用，有时候可能会造成混淆，请务必多加检查。

Bash中的字符串通过`'` 和 `"`分隔符来定义，但是它们的含义并不相同。以`'`定义的字符串为原义字符串，其中的变量不会被转义，而 `"`定义的字符串会将变量值进行替换。

![image-20220401110937373](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220401110937373.png)

bash使用了很多特殊的变量来表示参数、错误代码和相关变量。

- `$0` - 脚本名
- `$1` 到 `$9` - 脚本的参数。 `$1` 是第一个参数，依此类推。
- `$@` - 所有参数
- `$#` - 参数个数
- `$?` - 前一个命令的返回值
- `$$` - 当前脚本的进程识别码
- `!!` - 完整的上一条命令，包括参数。常见应用：当你因为权限不足执行命令失败时，可以使用 `sudo !!`再尝试一次。
- `$_` - 上一条命令的最后一个参数。如果你正在使用的是交互式 shell，你可以通过按下 `Esc` 之后键入 . 来获取这个值。

命令通常使用 `STDOUT`来返回输出值，使用`STDERR` 来返回错误及错误码，便于脚本以更加友好的方式报告错误。返回值0表示正常执行，其他所有非0的返回值都表示有错误发生。

同一行的多个命令可以用` ; `分隔。程序 `true` 的返回码永远是`0`，`false` 的返回码永远是`1`。

![image-20220401111531962](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220401111531962.png)

##### 命令替换

通过 `$( CMD )` 这样的方式来执行`CMD` 这个命令时，它的输出结果会替换掉 `$( CMD )` 。

##### 进程替换

 `<( CMD )` 会执行 `CMD` 并将结果输出到一个临时文件中，并将 `<( CMD )` 替换成临时文件名。

##### 运行脚本

```bash
#!/bin/bash

echo "Starting program at $(date)" # date会被替换成日期和时间

echo "Running program $0 with $# arguments with pid $$"

for file in "$@"; do
    grep foobar "$file" > /dev/null 2> /dev/null
    # 如果模式没有找到，则grep退出状态为 1
    # 我们将标准输出流和标准错误流重定向到Null，因为我们并不关心这些信息
    if [[ $? -ne 0 ]]; then
        echo "File $file does not have any foobar, adding one"
        echo "# foobar" >> "$file"
    fi
done
```

##### 通配（globbing）

###### 通配符

当你想要利用通配符进行匹配时，你可以分别使用 `?` 和 `*` 来匹配一个或任意个字符。

![image-20220401112506936](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220401112506936.png)

###### 花括号`{}`

 你有一系列的指令，其中包含一段公共子串时，可以用花括号来自动展开这些命令。可以用来批量移动或转换文件。

```bash

convert image.{png,jpg}
# 会展开为
convert image.png image.jpg

cp /path/to/project/{foo,bar,baz}.sh /newpath
# 会展开为
cp /path/to/project/foo.sh /path/to/project/bar.sh /path/to/project/baz.sh /newpath

# 也可以结合通配使用
mv *{.py,.sh} folder
# 会移动所有 *.py 和 *.sh 文件

mkdir foo bar

# 下面命令会创建foo/a, foo/b, ... foo/h, bar/a, bar/b, ... bar/h这些文件
touch {foo,bar}/{a..h}
touch foo/x bar/y
```

##### shebang

对于如下代码

```python
#!/usr/local/bin/python
import sys
for arg in reversed(sys.argv[1:]):
    print(arg)
```

内核知道去用 python 解释器而不是 shell 命令来运行这段脚本，是因为脚本的开头第一行的 [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix))

##### shell工具

> 重要的是你要知道有些问题使用合适的工具就会迎刃而解，而具体选择哪个工具则不是那么重要。

###### find

找文件，也可以用FD

###### grep

找文件内容

##### 查找 shell 命令

history 可以使用ctrl + R 进行搜索 也可以使用 | grep来找想要的历史命令

#### 课后练习

>阅读 man ls ，然后使用ls 命令进行如下操作：
>
>- 所有文件（包括隐藏文件） ：`-a`
>- 文件打印以人类可以理解的格式输出 (例如，使用454M 而不是 454279954) : `-h`
>- 文件以最近访问顺序排序：`-t`
>- 以彩色文本显示输出结果`--color=auto`

![image-20220401114840212](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220401114840212.png)

>
>
>编写两个bash函数 marco 和 polo 执行下面的操作。 每当你执行 marco 时，当前的工作目录应当以某种形式保存，当执行 polo 时，无论现在处在什么目录下，都应当 cd 回到当时执行 marco 的目录。 为了方便debug，你可以把代码写在单独的文件 marco.sh 中，并通过 source marco.sh命令，（重新）加载函数。通过source 来加载函数，随后可以在 bash 中直接使用。

```bash
 #!/bin/bash
 marco(){
     echo "$(pwd)" > $HOME/marco_history.log
     echo "save pwd $(pwd)"
 }
 polo(){
     cd "$(cat "$HOME/marco_history.log")"
 }
```

>
>
>假设您有一个命令，它很少出错。因此为了在出错时能够对其进行调试，需要花费大量的时间重现错误并捕获输出。 编写一段bash脚本，运行如下的脚本直到它出错，将它的标准输出和标准错误流记录到文件，并在最后输出所有内容。 加分项：报告脚本在失败前共运行了多少次。

debug.sh

```bash
 count=1

 while true
 do
     ./buggy.sh 2> out.log
     if [[ $? -ne 0 ]]; then
         echo "failed after $count times"
         cat out.log
         break
     fi
     ((count++))

 done
```

buggy.sh

```bash
 #!/usr/bin/env bash

 n=$(( RANDOM % 100 ))

 if [[ n -eq 42 ]]; then
     echo "Something went wrong"
     >&2 echo "The error was using magic numbers"
     exit 1
 fi

 echo "Everything went according to plan"
```

![image-20220401133122164](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220401133122164.png)

>
>
>您的任务是编写一个命令，它可以递归地查找文件夹中所有的HTML文件，并将它们压缩成zip文件。注意，即使文件名中包含空格，您的命令也应该能够正确执行（提示：查看 xargs的参数-d）

tip:有些命令，例如tar 则需要从参数接受输入。这里我们可以使用[xargs](https://man7.org/linux/man-pages/man1/xargs.1.html) 命令，它可以使用标准输入中的内容作为参数。

先创建一些html文件

```bash
 mkdir html_root
  cd html_root
  touch {1..10}.html
  mkdir html
  cd html
  touch xxxx.html
```

![image-20220401134201779](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220401134201779.png)

`find html_root -name "*.html" | xargs -d '\n' tar -cvzf html.zip`![image-20220401134612641](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220401134612641.png)

>
>
>(进阶) 编写一个命令或脚本递归的查找文件夹中最近使用的文件。更通用的做法，你可以按照最近的使用时间列出文件吗？

`find . -type f -print0 | xargs -0 ls -lt | head -1`

当文件数量较多时，上面的解答会得出错误结果，解决办法是增加 `-mmin`条件，先将最近修改的文件进行初步筛选再交给ls进行排序显示 `find . -type f -mmin -60 -print0 | xargs -0 ls -lt | head -10`

![image-20220401134803463](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220401134803463.png)

## vim

首先vim的学习对我来说确实是比较刚需，随着linux的使用越来越多，我目前就只会进入插入模式，然后用dd删除，:wq或者:q退出，其他都不咋会用，很有必要研究一下vim的使用

### vim的设计

Vim 避免了使用鼠标，因为那样太慢了；Vim 甚至避免用 上下左右键因为那样需要太多的手指移动。

#### 操作模式

Vim的设计以大多数时间都花在阅读、浏览和进行少量编辑改动为基础，因此它具有多种操作模式：

- *正常模式*：在文件中四处移动光标进行修改
- *插入模式*：插入文本
- *替换模式*：替换文本
- *可视化（一般，行，块）模式*：选中文本块
- *命令模式*：用于执行命令

在不同的操作模式下，键盘敲击的含义也不同。在默认设置下，Vim会在左下角显示当前的模式。 Vim启动时的默认模式是正常模式。通常你会把大部分 时间花在正常模式和插入模式。

你可以按下 `<ESC>` （退出键） 从任何其他模式返回正常模式。 在正常模式，键入 `i` 进入插入 模式， `R` 进入替换模式， `v` 进入可视（一般）模式， `V` 进入可视（行）模式， `Ctrl-V`进入可视（块）模式， `:` 进入命令模式。

#### 编程思想

Vim 最重要的设计思想是 Vim 的界面本身是一个程序语言。键入操作 （以及他们的助记名） 本身是命令， 这些命令可以组合使用。 这使得移动和编辑更加高效，特别是一旦形成肌肉记忆。

### 如何使用

#### 插入文本

按i进入插入模式后编辑文本

#### 缓存， 标签页， 窗口

Vim 会维护一系列打开的文件，称为“缓存”。一个 Vim 会话包含一系列标签页，每个标签页包含 一系列窗口（分隔面板）。每个窗口显示一个缓存。跟网页浏览器等其他你熟悉的程序不一样的是， 缓存和窗口不是一一对应的关系；窗口只是视角。一个缓存可以在_多个_窗口打开，甚至在同一 个标签页内的多个窗口打开。这个功能其实很好用，比如在查看同一个文件的不同部分的时候。`vim -o file1 file2`可以打开多个窗口，`:split file2` 新建一个窗口,`:vsplit file2`新建垂直分割窗口

#### 命令行模式

- `:q`退出
- `:w`保存
- `:wq`保存退出
- `:e filename`打开要编辑的文件
- `ls`显示打开的缓存
- `help name`打开name的帮助文档

#### 如何移动光标

多数时候你会在正常模式下，使用移动命令在缓存中导航。在 Vim 里面移动也被称为 “名词”， 因为它们指向文字块。

- 基本移动: `hjkl` （左， 下， 上， 右）(感觉上下左右就够用)

- 词： `w` （下一个词）， `b` （词初）， `e` （词尾）

- 行： `0` （行初）， `^` （第一个非空格字符）， `$` （行尾）

- 屏幕： `H` （屏幕首行）， `M` （屏幕中间）， `L` （屏幕底部）

- 翻页： `Ctrl-u` （上翻）， `Ctrl-d` （下翻）

- 文件： `gg` （文件头）， `G` （文件尾）

- 行数： `:{行数}<CR>` 或者 `{行数}G` ({行数}为行数)

- 杂项： `%` （找到配对，比如括号或者 /**/ 之类的注释对）

- 查找： `f{字符}`,  `t{字符}`，`F{字符}`，`T{字符}`
  - 查找/到 向前/向后 在本行的{字符}
  - `,` / `;` 用于导航匹配

- 搜索: `/{正则表达式}`, `n` / `N` 用于导航匹配

#### 选择

在可视化模式:

- 可视化：`v`
- 可视化行： `V`
- 可视化块：`Ctrl+v`

可以用`hjkl` 移动命令来选中，这样的话就可以选中一大段删除，之前一直在正常模式`dd`删除效率·1很低

#### 编辑

所有你需要用鼠标做的事， 你现在都可以用键盘：采用编辑命令和移动命令的组合来完成。 这就是 Vim 的界面开始看起来像一个程序语言的时候。Vim 的编辑命令也被称为 “动词”， 因为动词可以施动于名词。`

- `i`进入插入模式
  - 但是对于操纵/编辑文本，不单想用退格键完成
  
- `O` / `o` 在之上/之下插入行

- d{移动命令}删除 {移动命令}

  - 例如， `dw` 删除词, `d$` 删除到行尾, `d0` 删除到行头。

- c{移动命令}改变 {移动命令}

  - 例如， `cw` 改变词
  - 比如 `d{移动命令}` 再 `i`

- `x` 删除字符（等同于 `dl`）

- `s` 替换字符（等同于 `xi`）

- 可视化模式 + 操作

  - 选中文字, `d` 删除 或者 `c` 改变

- `u` 撤销, `<C-r>` 重做

- `y` 复制 / “yank” （其他一些命令比如 `d` 也会复制）

- `p` 粘贴

- 更多值得学习的: 比如 `~` 改变字符的大小写

#### 执行指定操作若干次

你可以用一个计数来结合“名词”和“动词”，这会执行指定操作若干次。

- `3w` 向前移动三个词
- `5j` 向下移动5行
- `7dw` 删除7个词

剩下的确实不太看得进去了，感觉上面看完已经可以基本使用了，剩下的内容需要在使用vim的过程中不断使用搜索引擎，然后寻找更好的解决当前问题的方法来提升自己。

### 课后练习

>完成vimtutor(vim自带的教程，在命令行输入vim即可)

>
>
>在使用中学习，而不是在记忆中学习

vimtutor主要是vim自带的一个教程，在实践中可以更好的学习vim

下面这个还是比较受用的，就是操作符 + 操作对象 操作对象也可以单独使用，比如w就是从光标移动到下一个单词初始

![image-20220402215530244](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220402215530244.png)

终于做完了，后面感觉不太用得上就先没看，我在linux下编辑代码也比较习惯能用鼠标的vscode这些，所以说大括号如何匹配啊这种写代码的操作就没怎么尝试了。

>1. 下载我们的[vimrc](https://missing-semester-cn.github.io/2020/files/vimrc)，然后把它保存到 `~/.vimrc`。 通读这个注释详细的文件 （用 Vim!）， 然后观察 Vim 在这个新的设置下看起来和使用起来有哪些细微的区别。

![image-20220402222555567](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220402222555567.png)

首先左边有个帮忙统计行数的插件，然后有代码高亮，禁止使用了四个方向键并且有对应错误提示

，也没怎么仔细看，大概这样，英文注释有点看不进去。

>安装和配置一个插件： `ctrlp.vim`.

>自定义 CtrlP： 添加 [configuration](https://github.com/ctrlpvim/ctrlp.vim/blob/master/readme.md#basic-options) 到你的 ~/.vimrc 来用按 Ctrl-P 打开 CtrlP

折磨 目前不打算换vim编辑代码，好烦，这些就不做了

>进一步自定义你的 ~/.vimrc 和安装更多插件。 安装插件最简单的方法是使用 Vim 的包管理器，即使用 vim-plug 安装插件

插件也不知道为啥装不上，吐了

总的来说还是收获不少，至少看完后可以有一定效率的用vim解决大多数问题，效率目前肯定没有用鼠标操控的文本编辑器效率高，但是还是算是会用了，反正越用越熟练嘛效率自然就上去了。


## Data Wrangling

这一节的主要内容是数据处理，将某种格式存储的数据转换成另外一种格式。

### 用来整理的数据以及相关的应用场景

日志处理通常是一个比较典型的使用场景，因为我们经常需要在日志中查找某些信息，这种情况下通读日志是不现实的。

研究一下系统日志，用ssh连接我自己的服务器(39.108.253.105)，看看哪些用户曾经尝试过登录我们的服务器：

```bash
ssh -l root 39.108.253.105 journalctl
```

![image-20220403211253725](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220403211253725.png)

能够得到特别多的信息，那这时候想得到有用的信息就需要过滤

```bash
ssh -l root 39.108.253.105 journalctl | grep sshd
```

sshd是ssh服务进程的名字，会发现还是很多，我们来改进一下：

```bash
ssh -l root 39.108.253.105 'journalctl | grep sshd | grep "Disconnected from"' | less
```

多出来的引号是什么作用呢？这么说吧，我们的日志是一个非常大的文件，把这么大的文件流直接传输到我们本地的电脑上再进行过滤是对流量的一种浪费。因此我们采取另外一种方式，我们先在远端机器上过滤文本内容，然后再将结果传输到本机。 `less` 为我们创建来一个文件分页器，使我们可以通过翻页的方式浏览较长的文本。

![image-20220403212301621](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220403212301621.png)

为了进一步节省流量，我们甚至可以将当前过滤出的日志保存到文件中，这样后续就不需要再次通过网络访问该文件了：

```bash
ssh -l root 39.108.253.105 'journalctl | grep sshd | grep "Disconnected from"' > ssh.log
less ssh.log
```

报错permission denied，不知道为啥...

我们先研究一下 `sed` 这个非常强大的工具。`sed` 是一个基于文本编辑器`ed`构建的”流编辑器” 。在 `sed` 中，您基本上是利用一些简短的命令来修改文件，而不是直接操作文件的内容（尽管您也可以选择这样做）。相关的命令行非常多，但是最常用的是 `s`，即*替换*命令，例如我们可以这样写：

```bash
ssh -l root 39.108.253.105 journalctl | grep sshd | grep "Disconnected from"| sed 's/.*Disconnected from //'
```

### 正则表达式

 `/.*Disconnected from /`，正则表达式通常以/开始和结束。

- `.` 除换行符之外的”任意单个字符”
- `*` 匹配前面字符零次或多次
- `+` 匹配前面字符一次或多次
- `[abc]` 匹配 `a`, `b` 和 `c` 中的任意一个
- `(RX1|RX2)` 匹配`RX1` 或 `RX2`
- `^` 行首
- `$` 行尾
- `{num1,num2}`匹配num1-num2个前面字符

回过头我们再看`/.*Disconnected from /`，我们会发现这个正则表达式可以匹配任何以若干任意字符开头，并接着包含”Disconnected from “的字符串。这也正式我们所希望的。`sed` 还可以非常方便的做一些事情，例如打印匹配后的内容，一次调用中进行多次替换搜索等。

想要匹配用户名后面的文本，尤其是当这里的用户名可以包含空格时，这个问题变得非常棘手！这里我们需要做的是匹配*一整行*：

```bash
 | sed -E 's/.*Disconnected from (invalid |authenticating )?user .* [^ ]+ port [0-9]+( \[preauth\])?$//'
```

开始的部分和以前是一样的，随后，我们匹配两种类型的“user”（在日志中基于两种前缀区分）。再然后我们匹配属于用户名的所有字符。接着，再匹配任意一个单词（`[^ ]+` 会匹配任意非空且不包含空格的序列）。紧接着后面匹配单“port”和它后面的一串数字，以及可能存在的后缀`[preauth]`，最后再匹配行尾。

问题还没有完全解决，日志的内容全部被替换成了空字符串，整个日志的内容因此都被删除了。我们实际上希望能够将用户名*保留*下来。对此，我们可以使用“捕获组（capture groups）”来完成。被圆括号内的正则表达式匹配到的文本，都会被存入一系列以编号区分的捕获组中。捕获组的内容可以在替换字符串时使用（有些正则表达式的引擎甚至支持替换表达式本身），例如`\1`、 `\2`、`\3`等等，因此可以使用如下命令：

```
 | sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/'
```

看完这些非常头疼，正则表达式好复杂

#### 回到数据整理

`sed` 还可以做很多各种各样有趣的事情，例如文本注入：(使用 `i` 命令)，打印特定的行 (使用 `p`命令)，基于索引选择特定行等等。详情请见`man sed`

好难....确实没啥动力看下面的内容了，感觉光看有点难学明白，大概懂个意思过两天也忘了

### 课后练习

>1. 学习一下这篇简短的 [交互式正则表达式教程](https://regexone.com/).

感觉交互式教程还不错，在运用中对正则的规则有了一些印象

`\d`匹配数字，`\s`匹配空格，`\w`匹配字母，`^`表示行首，`$`表示行末，`()`表示一个组，

内容其实还挺不错的，不过有点摸了今天，就这样叭，后面的就不做了，确实感觉有点烦

## Command-line Environment

> 学习如何同时执行多个不同的进程并追踪它们的状态、如何停止或暂停某个进程以及如何使进程在后台运行，学习一些能够改善您的 shell 及其他工具的工作流的方法，这主要是通过定义别名或基于配置文件对其进行配置来实现的。

主要就是讲使用命令行查看当前机器的进程和命令行环境的配置等内容。

### 任务控制

众所周知，`<C-c>`可以停止命令行命令的执行。

#### 结束进程

shell 会使用 UNIX 提供的信号机制执行进程间通信。当一个进程接收到信号时，它会停止执行、处理该信号并基于信号传递的信息来改变其执行。就这一点而言，信号是一种软件中断。

当我们输入 `Ctrl-C` 时，shell 会发送一个`SIGINT` 信号到进程。

下面是一个捕获`SIGINT`信号并且忽略它的代码，停止此程序需要`SIGQUIT`，输入`Ctrl-\`就可以。

```python
#!/usr/bin/env python
import signal, time

def handler(signum, time):
    print("\nI got a SIGINT, but I am not stopping")

signal.signal(signal.SIGINT, handler)
i = 0
while True:
    time.sleep(.1)
    print("\r{}".format(i), end="")
    i += 1
```

很奇怪的是我好像不能用ctrl + \发送`sigquit`信号，然后我又去git bash里面试了试，发现也停不下来，难道是我键盘有问题么

![image-20220404114543576](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220404114543576.png)

![image-20220404114406107](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220404114406107.png)

而且我可以正常的打出^\

![image-20220404114602259](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220404114602259.png)

搞不懂问题出在哪，应该也不是操作系统的差异导致的，因为bash不是模拟的linux的命令行么

![image-20220404114929876](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220404114929876.png)

虚拟机里面的manjaro可以正常停下来，我认为问题的根本应该是gitbash模拟的linux环境不够真实吧...因为shell本质是个和操作系统内核交互的应用程序，然后windows应该是不能用`^\`发送`SIGQUIT`信号的，应该是只能`^C`发送`SIGINT`信号，所以powershell和git bash都没有能够停下进程，但是manjaro里面的zsh成功停下来了。

上面主要是讲了`SIGINT`和`SIGNQUIT`命令。`SIGTERM` 则是一个更加通用的、也更加优雅地退出信号。为了发出这个信号我们需要使用 [`kill`](https://www.man7.org/linux/man-pages/man1/kill.1.html) 命令, 它的语法是： `kill -TERM <PID>`。

#### 暂停和后台执行进程

信号可以让进程做其他的事情，而不仅仅是终止它们。例如，`SIGSTOP` 会让进程暂停( `Ctrl-Z` )，我们可以使用 [`fg`](https://www.man7.org/linux/man-pages/man1/fg.1p.html) 或 [`bg`](http://man7.org/linux/man-pages/man1/bg.1p.html) 命令恢复暂停的工作。它们分别表示在前台继续或在后台继续，[`jobs`](http://man7.org/linux/man-pages/man1/jobs.1p.html) 命令会列出当前终端会话中尚未完成的全部任务。

![image-20220404120813276](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220404120813276.png)

后台的进程仍然是您的终端进程的子进程，一旦您关闭终端（会发送另外一个信号`SIGHUP`），这些后台的进程也会终止。为了防止这种情况发生，您可以使用 [`nohup`](https://www.man7.org/linux/man-pages/man1/nohup.1.html) (一个用来忽略 `SIGHUP` 的封装) 来运行程序。

比如我最近整了个qq机器人挂在协会的服务器上，那如果我需要让qq机器人在ssh连接断开的情况下继续运行，要么使用screen挂起一个终端，要么就用nohup让终端的关闭也不会影响qq机器人这个后台进程。可以使用百分号 + 任务编号（`jobs` 会打印任务编号）来选取该任务。

命令中的 `&` 后缀可以让命令在直接在后台运行，这使得您可以直接在 shell 中继续做其他操作。

下面的命令行交互过程演示了上面的一些知识，比如说用nohup挂起的当前终端的子进程2，因为用了nohup所以说`SIGHUP`这个信号就没法kill这个进程，当然如果直接kill这个进程还是可以的。

![image-20220404121246236](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220404121246236.png)

### 终端多路复用

当您在使用命令行时，您通常会希望同时执行多个任务。举例来说，您可以想要同时运行您的编辑器，并在终端的另外一侧执行程序。尽管再打开一个新的终端窗口也能达到目的，使用终端多路复用器则是一种更好的办法。

感觉目前没什么需求，有一定的学习成本然后学了后又不用容易忘，简单知道一下有tmux这么个工具吧

### 别名

为了避免重复输入一长串包括许多选项的命令，shell支持设置别名

```bash
alias alias_name="command_to_alias arg1 arg2"
```

```bash
# 创建常用命令的缩写
alias ll="ls -lh"

# 能够少输入很多
alias gs="git status"
alias gc="git commit"
alias v="vim"

# 手误打错命令也没关系
alias sl=ls

# 重新定义一些命令行的默认行为
alias mv="mv -i"           # -i prompts before overwrite
alias mkdir="mkdir -p"     # -p make parent dirs as needed
alias df="df -h"           # -h prints human readable format

# 别名可以组合使用
alias la="ls -A"
alias lla="la -l"

# 在忽略某个别名
\ls
# 或者禁用别名
unalias la

# 获取别名的定义
alias ll
# 会打印 ll='ls -lh'
```

在默认情况下 shell 并不会保存别名。为了让别名持续生效，您需要将配置放进 shell 的启动文件里，像是`.bashrc` 或 `.zshrc`

### 配置文件（Dotfiles）

很多程序的配置都是通过纯文本格式的被称作*点文件*的配置文件来完成的（之所以称为点文件，是因为它们的文件名以 `.` 开头，例如 `~/.vimrc`。也正因为此，它们默认是隐藏文件，`ls`并不会显示它们）。

实际上，很多程序都要求您在 shell 的配置文件中包含一行类似 `export PATH="$PATH:/path/to/program/bin"` 的命令，这样才能确保这些程序能够被 shell 找到。

还有一些其他的工具也可以通过*点文件*进行配置：

- `bash` - `~/.bashrc`, `~/.bash_profile`
- `git` - `~/.gitconfig`
- `vim` - `~/.vimrc` 和 `~/.vim` 目录
- `ssh` - `~/.ssh/config`
- `tmux` - `~/.tmux.conf`

### 远端设备（ssh）

说到ssh就不得不推荐一下termius辣，在协会学长的推荐下用了这个ssh客户端，真不戳。

通过如下命令，您可以使用 `ssh` 连接到其他服务器：

```bash
ssh foo@bar.mit.edu
```

`ssh` 的一个经常被忽视的特性是它可以直接远程执行命令。 `ssh foobar@server ls` 可以直接在用foobar的命令下执行 `ls` 命令。 想要配合管道来使用也可以， `ssh foobar@server ls | grep PATTERN` 会在本地查询远端 `ls` 的输出而 `ls | ssh foobar@server grep PATTERN` 会在远端对本地 `ls` 输出的结果进行查询。

关于ssh远程执行命令这一点，在数据整理的内容中也有对应的运用。

#### SSH 密钥

基于密钥的验证机制使用了密码学中的公钥，我们只需要向服务器证明客户端持有对应的私钥，而不需要公开其私钥。这样您就可以避免每次登录都输入密码的麻烦了秘密就可以登录。不过，私钥(通常是 `~/.ssh/id_rsa` 或者 `~/.ssh/id_ed25519`) 等效于您的密码，所以一定要好好保存它。

##### 密钥生成

使用 [`ssh-keygen`](http://man7.org/linux/man-pages/man1/ssh-keygen.1.html) 命令可以生成一对密钥：

```
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519
```

###### 基于密钥的认证机制

`ssh` 会查询 `.ssh/authorized_keys` 来确认那些用户可以被允许登录

#### 通过 SSH 复制文件

- `ssh+tee`, 最简单的方法是执行 `ssh` 命令，然后通过这样的方法利用标准输入实现 `cat localfile | ssh remote_server tee serverfile`。回忆一下，[`tee`](https://www.man7.org/linux/man-pages/man1/tee.1.html) 命令会将标准输出写入到一个文件；
- [`scp`](https://www.man7.org/linux/man-pages/man1/scp.1.html) ：当需要拷贝大量的文件或目录时，使用`scp` 命令则更加方便，因为它可以方便的遍历相关路径。语法如下：`scp path/to/local_file remote_host:path/to/remote_file`；
- [`rsync`](https://www.man7.org/linux/man-pages/man1/rsync.1.html) 对 `scp` 进行了改进，它可以检测本地和远端的文件以防止重复拷贝。它还可以提供一些诸如符号连接、权限管理等精心打磨的功能。甚至还可以基于 `--partial`标记实现断点续传。`rsync` 的语法和`scp`类似；

##### 利用ssh实现监听远程设备的端口

##### 本地端口转发

本地端口转发，即远端设备上的服务监听一个端口，而您希望在本地设备上的一个端口建立连接并转发到远程端口上。

例如，我们在远端服务器上运行 Jupyter notebook 并监听 `8888` 端口。 然后，建立从本地端口 `9999` 的转发，使用 `ssh -L 9999:localhost:8888 foobar@remote_server` 。这样只需要访问本地的 `localhost:9999` 即可。

![localport](https://i.stack.imgur.com/a28N8.png%C2%A0)

##### 远程端口转发

感觉用处不大

![localport](https://i.stack.imgur.com/4iK3b.png )

### 课后练习

>我们可以使用类似 ps aux | grep 这样的命令来获取任务的 pid ，然后您可以基于pid 来结束这些进程。但我们其实有更好的方法来做这件事。在终端中执行 sleep 10000 这个任务。然后用 Ctrl-Z 将其切换到后台并使用 bg来继续允许它。现在，使用 [pgrep](https://www.man7.org/linux/man-pages/man1/pgrep.1.html) 来查找 pid 并使用 [pkill](http://man7.org/linux/man-pages/man1/pgrep.1.html) 结束进程而不需要手动输入pid。(提示：: 使用 -af 标记)。

pgrep相当于更方便的过滤出你想要的进程pid

![image-20220404130745438](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220404130745438.png)

>如果您希望某个进程结束后再开始另外一个进程， 应该如何实现呢？ 在这个练习中，我们使用 sleep 60 & 作为先执行的程序。一种方法是使用 wait 命令。尝试启动这个休眠命令，然后待其结束后再执行 ls 命令。

```sh
sleep 60 &
pgrep sleep | wait; ls
```

![image-20220404134741612](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220404134741612.png)

>但是，如果我们在不同的 bash 会话中进行操作，则上述方法就不起作用了。因为 `wait` 只能对子进程起作用。之前我们没有提过的一个特性是，`kill` 命令成功退出时其状态码为 0 ，其他状态则是非0。`kill -0` 则不会发送信号，但是会在进程不存在时返回一个不为0的状态码。请编写一个 bash 函数 `pidwait` ，它接受一个 pid 作为输入参数，然后一直等待直到该进程结束。您需要使用 `sleep` 来避免浪费 CPU 性能。

```bash
pidwait()
{
   while kill -0 $1 #循环直到进程结束
   do
   sleep 1
   done
   ls
}
```

![image-20220404135526970](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220404135526970.png)

终端多路复用目前没啥需求就先不做了，ssh已经比较熟练了平时用的多所以也不做了，配置文件有点懒得整，先到这里吧，发现了这个课程的中文文档的两个问题，顺便去给仓库提了俩pr哈哈哈

## Git

### Git 的数据模型

​  Git 拥有一个经过精心设计的模型，这使其能够支持版本控制所需的所有特性，例如维护历史记录、支持分支和促进协作。

#### 快照

​  在Git的术语里，文件被称作Blob对象（数据对象），也就是一组数据。目录则被称之为“树”，它将名字与 Blob 对象或树对象进行映射（使得目录中可以包含其他目录）。快照则是被追踪的最顶层的树。例如，一个树看起来可能是这样的：

```
<root> (tree)
|
+- foo (tree)
|  |
|  + bar.txt (blob, contents = "hello world")
|
+- baz.txt (blob, contents = "git is wonderful")
```

​  这个顶层的树包含了两个元素，一个名为 “foo” 的树（它本身包含了一个blob对象 “bar.txt”），以及一个 blob 对象 “baz.txt”。

#### 历史记录建模：关联快照

​  版本控制系统和快照有什么关系呢？线性历史记录是一种最简单的模型，它包含了一组按照时间顺序线性排列的快照。不过处于种种原因，Git 并没有采用这样的模型。

​  在 Git 中，历史记录是一个由快照组成的有向无环图。注意，快照具有多个“父辈”而非一个，因为某个快照可能由多个父辈而来。例如，经过合并后的两条分支。

​  在 Git 中，这些快照被称为“提交”。通过可视化的方式来表示这些历史提交记录时，看起来差不多是这样的：

```
o <-- o <-- o <-- o
            ^  
             \
              --- o <-- o
```

​  上面是一个 ASCII 码构成的简图，其中的 `o` 表示一次提交（快照）。

​  箭头指向了当前提交的父辈（这是一种“在。。。之前”，而不是“在。。。之后”的关系）。

#### 数据模型及其伪代码表示

```
// 文件就是一组数据
type blob = array<byte>

// 一个包含文件和目录的目录
type tree = map<string, tree | blob>

// 每个提交都包含一个父辈，元数据和顶层树
type commit = struct {
    parent: array<commit>
    author: string
    message: string
    snapshot: tree
}
```

#### 对象和内存寻址

Git 中的对象可以是 blob、树或提交：

```
type object = blob | tree | commit
```

Git 在储存数据时，所有的对象都会基于它们的 [SHA-1 哈希](https://en.wikipedia.org/wiki/SHA-1) 进行寻址。哈希值非常关键。

```
objects = map<string, object>

def store(object):
    id = sha1(object)
    objects[id] = object

def load(id):
    return objects[id]
```

Blobs、树和提交都一样，它们都是对象。当它们引用其他对象时，它们并没有真正的在硬盘上保存这些对象，而是仅仅保存了它们的哈希值作为引用。

```
<root> (tree)
|
+- foo (tree)
|  |
|  + bar.txt (blob, contents = "hello world")
|
+- baz.txt (blob, contents = "git is wonderful")
```

例子中的树看上去是这样的：

```
100644 blob 4448adbf7ecd394f42ae135bbeed9676e894af85    baz.txt
040000 tree c68d233a33c5c06e0340e4c224f0afca87c8ce87    foo
```

树本身会包含一些指向其他内容的指针，例如 `baz.txt` (blob) 和 `foo` (树)。如果我们用 `git cat-file -p 4448adbf7ecd394f42ae135bbeed9676e894af85`，即通过哈希值查看 baz.txt 的内容，会得到以下信息：

```
git is wonderful
```

#### 引用

至此commit就可以用SHA-1哈希值来标记了，但是哈希值并不好记，因此需要引用（references），引用是指向提交的指针。与对象不同的是，它是可变的（引用可以被更新，指向新的提交）。例如，`master` 引用通常会指向主分支的最新一次提交。这样，Git 就可以使用诸如 “master” 这样人类可读的名称来表示历史记录中某个特定的提交，而不需要在使用一长串十六进制字符了。

通常情况下，我们会想要知道“我们当前所在位置”，并将其标记下来。这样当我们创建新的提交的时候，我们就可以知道它的相对位置（如何设置它的“父辈”）。在 Git 中，我们当前的位置有一个特殊的索引，它就是 “HEAD”。

#### 仓库

现在我们可以给出 Git 仓库的定义：`对象` 和 `引用`。

在硬盘上，Git 仅存储对象和引用：因为其数据模型仅包含这些东西。所有的 `git` 命令都对应着对提交树的操作，例如增加对象，增加或删除引用。

### 暂存区

暂存区和数据模型不相关，但是它是创捷提交接口的一部分。

我们先来理解下 Git 工作区、暂存区和版本库概念：

- **工作区：**就是你在电脑里能看到的目录。
- **暂存区：**英文叫 stage 或 index。一般存放在 **.git** 目录下的 index 文件（.git/index）中，所以我们把暂存区有时也叫作索引（index）。
- **版本库：**工作区有一个隐藏目录 **.git**，这个不算工作区，而是 Git 的版本库。

下面这个图画的非常清晰了

![image-20220405212445433](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220405212445433.png)

### Git常用操作

![image-20220405212728387](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220405212728387.png)

#### 基础

- `git help <command>`: 获取 git 命令的帮助信息
- `git init`: 创建一个新的 git 仓库，其数据会存放在一个名为 `.git` 的目录下
- `git status`: 显示当前的仓库状态
- `git add <filename>`: 添加文件到暂存区
- `git commit`: 创建一个新的提交
  - 如何编写 [良好的提交信息](https://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html)!
  - 为何要 [编写良好的提交信息](https://chris.beams.io/posts/git-commit/)
- `git log`: 显示历史日志
- `git log --all --graph --decorate`: 可视化历史记录（有向无环图）
- `git diff <filename>`: 显示与暂存区文件的差异
- `git diff <revision> <filename>`: 显示某个文件两个版本之间的差异
- `git checkout <revision>`: 更新 HEAD 和目前的分支

#### 分支和合并

- `git branch`: 显示分支

- `git branch <name>`: 创建分支

- `git checkout -b <name>`: 创建分支并切换到该分支
  - 相当于 `git branch <name>; git checkout <name>`
  
- `git merge <revision>`: 合并到当前分支

- `git mergetool`: 使用工具来处理合并冲突

- `git rebase`: 将一系列补丁变基（rebase）为新的基线

#### 远端操作

- `git remote`: 列出远端
- `git remote add <name> <url>`: 添加一个远端
- `git push <remote> <local branch>:<remote branch>`: 将对象传送至远端并更新远端引用
- `git branch --set-upstream-to=<remote>/<remote branch>`: 创建本地和远端分支的关联关系
- `git fetch`: 从远端获取对象/索引
- `git pull`: 相当于 `git fetch; git merge`
- `git clone`: 从远端下载仓库

#### 撤销

- `git commit --amend`: 编辑提交的内容或信息
- `git reset HEAD <file>`: 恢复暂存的文件
- `git checkout -- <file>`: 丢弃修改

#### Git 高级操作

- `git config`: Git 是一个 [高度可定制的](https://git-scm.com/docs/git-config) 工具
- `git clone --depth=1`: 浅克隆（shallow clone），不包括完整的版本历史信息
- `git add -p`: 交互式暂存
- `git rebase -i`: 交互式变基
- `git blame`: 查看最后修改某行的人
- `git stash`: 暂时移除工作目录下的修改内容
- `git bisect`: 通过二分查找搜索历史记录
- `.gitignore`: [指定](https://git-scm.com/docs/gitignore) 故意不追踪的文件

### 课后练习

>1. 如果您之前从来没有用过 Git，推荐您阅读 [Pro Git](https://git-scm.com/book/en/v2) 的前几章，或者完成像 [Learn Git Branching](https://learngitbranching.js.org/)这样的教程。重点关注 Git 命令和数据模型相关内容；

我决定看一下Git Branching，之前就看过这个小游戏觉得不错，但是一直没做一下。下面记录一下自己还没搞明白的一些基础git 命令

#### 如何切换当前指向的commit记录?

`git branch`是新建分支 `git checkout`是对HEAD指向的分支进行操作，比如说可以`git checkout <branchname>`切换分支，可以用`git checkout <hash>`来分离HEAD，HEAD会自动指向master，会表示成master* ，那比如说当前master指向C2，HEAD -> master -> C2，执行后会变成HEAD -> C2。HEAD就是我们当前指向的commit记录，那通过这种方式我们就可以切换HEAD到一个我们想更改的commit记录了。

#### 如何撤销更改?

`git reset`本地仓库commit回滚

`git revert`远程仓库commit回滚，但是会生成新的commit记录，并不是消除撤销的记录。

#### 如何合并分支？

`git merge <branchname>` 将HEAD指向的分支和<branchname>合并，生成一个新commit。

`git rebase <branchname>`将HEAD指向的分支的不同commit记录（也就是两个分支的有差异的commit记录）移动到<branchname>分支上，变成顺序关系.

上面这些可以解决90%问题了，剩下的话具体遇到问题再查吧，命令太多真有点记不住了。

看了看其他课程练习，就是对这个课程网站的git仓库查一查commit记录，其他都比较熟练了，查某行更改时谁的话用`git blame`就可以，其他感觉平时不太用的到，先把上面的记清楚吧，我现在也不是很熟练回滚和合并分支的操作，只会`git add .` `git commit -m ""` `git psuh` XD

## 调试及性能分析

### 调试代码

#### 打印调试法与日志

要么加打印语句，要么用日志。

日志的优势:

- 您可以将日志写入文件、socket 或者甚至是发送到远端服务器而不仅仅是标准输出；
- 日志可以支持严重等级（例如 INFO, DEBUG, WARN, ERROR等)，这使您可以根据需要过滤日志；
- 对于新发现的问题，很可能您的日志中已经包含了可以帮助您定位问题的足够的信息.

对日志着色可以让日志可读性更好，下面是一个可以在终端打印颜色的bash脚本

```bash
#!/usr/bin/env bash
for R in $(seq 0 20 255); do
    for G in $(seq 0 20 255); do
        for B in $(seq 0 20 255); do
            printf "\e[38;2;${R};${G};${B}m█\e[0m";
        done
    done
done
```

![image-20220406114949259](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406114949259.png)

#### 第三方日志系统

如果您正在构建大型软件系统，您很可能会使用到一些依赖，有些依赖会作为程序单独运行。如 Web 服务器、数据库或消息代理都是此类常见的第三方依赖。

和这些系统交互的时候，阅读它们的日志是非常必要的，因为仅靠客户端侧的错误信息可能并不足以定位问题，大多数的程序都会将日志保存在您的系统中的某个地方。对于 UNIX 系统来说，程序的日志通常存放在 `/var/log`。例如， [NGINX](https://www.nginx.com/) web 服务器就将其日志存放于`/var/log/nginx`。

目前，系统开始使用 **system log**，您所有的日志都会保存在这里。大多数（但不是全部的）Linux 系统都会使用 `systemd`，这是一个系统守护进程，它会控制您系统中的很多东西，例如哪些服务应该启动并运行。`systemd` 会将日志以某种特殊格式存放于`/var/log/journal`，您可以使用 [`journalctl`](http://man7.org/linux/man-pages/man1/journalctl.1.html) 命令显示这些消息。对于大多数的 UNIX 系统，您也可以使用[`dmesg`](http://man7.org/linux/man-pages/man1/dmesg.1.html) 命令来读取内核的日志。

不仅如此，大多数的编程语言都支持向系统日志中写日志。

![image-20220406115724039](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406115724039.png)

#### 调试器

当通过打印已经不能满足您的调试需求时，您应该使用调试器。

调试器是一种可以允许我们和正在执行的程序进行交互的程序，它可以做到：

- 当到达某一行时将程序暂停；
- 一次一条指令地逐步执行程序；
- 程序崩溃后查看变量的值；
- 满足特定条件时暂停程序；

```python
def bubble_sort(arr):
    n = len(arr)
    for i in range(n):
        for j in range(n):
            if arr[j] > arr[j+1]:
                arr[j] = arr[j+1]
                arr[j+1] = arr[j]
    return arr

print(bubble_sort([4, 2, 1, 8, 7, 6]))
```

Python 的调试器是[`pdb`](https://docs.python.org/3/library/pdb.html)，下面对`pdb` 支持的命令进行简单的介绍：

- **l**(ist) - 显示当前行附近的11行或继续执行之前的显示；
- **s**(tep) - 执行当前行，并在第一个可能的地方停止，可以进入函数；
- **n**(ext) - 继续执行直到当前函数的下一条语句或者 return 语句；
- **b**(reak) - 设置断点（基于传入的参数）；
- **p**(rint) - 在当前上下文对表达式求值并打印结果。还有一个命令是**pp** ，它使用 [`pprint`](https://docs.python.org/3/library/pprint.html) 打印；
- **r**(eturn) - 继续执行直到当前函数运行完，返回结果；
- **c**(ontinue) - 执行到下一断点或者结束
- **q**(uit) - 退出调试器。

注意，因为 Python 是一种解释型语言，所以我们可以通过 `pdb` shell 执行命令。 [`ipdb`](https://pypi.org/project/ipdb/) 是一种增强型的 `pdb` ，它使用[`IPython`](https://ipython.org/) 作为 REPL并开启了 tab 补全、语法高亮、更好的回溯和更好的内省，同时还保留了`pdb` 模块相同的接口。

接下来我们尝试使用pdb来调试这段冒泡排序的python代码。

首先进入ipdb调试

![image-20220406123045415](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406123045415.png)

pdb shell中调用step 也就是输入s，然后不停回车就可以逐步调试

![image-20220406123336453](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406123336453.png)

发现数组越界后，打印j的值看一看

![image-20220406123444118](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406123444118.png)

发现j的值是5，那么j+1的值是6，然后由于j的范围是range(n)，也就是0-5，当j为5时候arr[j+1]数组越界导致报错，所以j的范围应该更改为range(n-1)，quit出pdb，更改j的范围后，我们运行一下程序看看。

![image-20220406123755593](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406123755593.png)

现在程序不报错了，但是显然冒泡排序的目的并没有达成，现在我们再次进入pdb打断点调试一下。

我们现在排序的地方出了问题，明明传入的值是4，2，1，8，7，6，但是输出都变成了1和6，所以问题出在交换的地方，那我们就给第6行打个断点，然后继续执行代码到断点处。

![image-20220406124330746](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406124330746.png)

看一下当前变量的值

![image-20220406124402604](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406124402604.png)

执行一轮循环后停在断点处，然后再看一下数组的值

![image-20220406124523660](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406124523660.png)

所以问题找到了，通过上面的操作可以简单熟悉一下pdb工具

#### Specialized Tools

即使您需要调试的程序是一个二进制的黑盒程序，仍然有一些工具可以帮助到您。当您的程序需要执行一些只有操作系统内核才能完成的操作时，它需要使用 [系统调用](https://en.wikipedia.org/wiki/System_call)。有一些命令可以帮助您追踪您的程序执行的系统调用。在 Linux 中可以使用[`strace`](http://man7.org/linux/man-pages/man1/strace.1.html) ，下面的例子展现来如何使用 `strace` 或 `dtruss` 来显示`ls` 执行时，对[`stat`](http://man7.org/linux/man-pages/man2/stat.2.html) 系统调用进行追踪对结果。

![image-20220406143402443](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406143402443.png)

#### 静态分析

有些问题是您不需要执行代码就能发现的。例如，仔细观察一段代码，您就能发现某个循环变量覆盖了某个已经存在的变量或函数名；或是有个变量在被读取之前并没有被定义。 这种情况下 [静态分析](https://en.wikipedia.org/wiki/Static_program_analysis) 工具就可以帮我们找到问题。静态分析会将程序的源码作为输入然后基于编码规则对其进行分析并对代码的正确性进行推理。

对于风格检查和代码格式化，还有以下一些工具可以作为补充：用于 Python 的 [`black`](https://github.com/psf/black)、用于 Go 语言的 `gofmt`、用于 Rust 的 `rustfmt` 或是用于 JavaScript, HTML 和 CSS 的 [`prettier`](https://prettier.io/) 。这些工具可以自动格式化您的代码，这样代码风格就可以与常见的风格保持一致。 尽管您可能并不想对代码进行风格控制，标准的代码风格有助于方便别人阅读您的代码，也可以方便您阅读它的代码。

### 性能分析

鉴于 [过早的优化是万恶之源](http://wiki.c2.com/?PrematureOptimization)，您需要学习性能分析和监控工具，它们会帮助您找到程序中最耗时、最耗资源的部分，这样您就可以有针对性的进行性能优化。

#### 计时

和调试代码类似，大多数情况下我们只需要打印两处代码之间的时间即可发现问题，但是CPU同时在处理多个进程，这个时间代表的代码运行的时间并不一定准确。

用time 查看一个http请求的资源消耗情况

![image-20220406144441783](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220406144441783.png)

#### 性能分析工具（profilers）

这章看的比较混，因为感觉目前不太用得上

##### CPU

 CPU 性能分析工具有两种： 追踪分析器（*tracing*）及采样分析器（*sampling*）。追踪分析器 会记录程序的每一次函数调用，而采样分析器则只会周期性的监测（通常为每毫秒）您的程序并记录程序堆栈。

大多数的编程语言都有一些基于命令行的分析器，我们可以使用它们来分析代码，它们通常可以集成在 IDE 中。

##### 内存

像 C 或者 C++ 这样的语言，内存泄漏会导致您的程序在使用完内存后不去释放它。为了应对内存类的 Bug，我们可以使用类似 [Valgrind](https://valgrind.org/) 这样的工具来检查内存泄漏问题。

对于 Python 这类具有垃圾回收机制的语言，内存分析器也是很有用的，因为对于某个对象来说，只要有指针还指向它，那它就不会被回收。

#### 资源监控

- **通用监控** - 最流行的工具要数 [`htop`](https://htop.dev/),了，它是 [`top`](http://man7.org/linux/man-pages/man1/top.1.html)的改进版。`htop` 可以显示当前运行进程的多种统计信息。`htop` 有很多选项和快捷键，常见的有：`<F6>` 进程排序、 `t` 显示树状结构和 `h` 打开或折叠线程。 还可以留意一下 [`glances`](https://nicolargo.github.io/glances/) ，它的实现类似但是用户界面更好。如果需要合并测量全部的进程， [`dstat`](http://dag.wiee.rs/home-made/dstat/) 是也是一个非常好用的工具，它可以实时地计算不同子系统资源的度量数据，例如 I/O、网络、 CPU 利用率、上下文切换等等；
- **I/O 操作** - [`iotop`](http://man7.org/linux/man-pages/man8/iotop.8.html) 可以显示实时 I/O 占用信息而且可以非常方便地检查某个进程是否正在执行大量的磁盘读写操作；
- **磁盘使用** - [`df`](http://man7.org/linux/man-pages/man1/df.1.html) 可以显示每个分区的信息，而 [`du`](http://man7.org/linux/man-pages/man1/du.1.html) 则可以显示当前目录下每个文件的磁盘使用情况（ **d**isk **u**sage）。`-h` 选项可以使命令以对人类（**h**uman）更加友好的格式显示数据；[`ncdu`](https://dev.yorhel.nl/ncdu)是一个交互性更好的 `du` ，它可以让您在不同目录下导航、删除文件和文件夹；
- **内存使用** - [`free`](http://man7.org/linux/man-pages/man1/free.1.html) 可以显示系统当前空闲的内存。内存也可以使用 `htop` 这样的工具来显示；
- **打开文件** - [`lsof`](http://man7.org/linux/man-pages/man8/lsof.8.html) 可以列出被进程打开的文件信息。 当我们需要查看某个文件是被哪个进程打开的时候，这个命令非常有用；
- **网络连接和配置** - [`ss`](http://man7.org/linux/man-pages/man8/ss.8.html) 能帮助我们监控网络包的收发情况以及网络接口的显示信息。`ss` 常见的一个使用场景是找到端口被进程占用的信息。如果要显示路由、网络设备和接口信息，您可以使用 [`ip`](http://man7.org/linux/man-pages/man8/ip.8.html) 命令。注意，`netstat` 和 `ifconfig` 这两个命令已经被前面那些工具所代替了。
- **网络使用** - [`nethogs`](https://github.com/raboof/nethogs) 和 [`iftop`](http://www.ex-parrot.com/pdw/iftop/) 是非常好的用于对网络占用进行监控的交互式命令行工具。

课后练习就不看了，感觉这章的东西对我来说，主要就是会用调试工具，性能分析不太用得上
