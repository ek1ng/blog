---
title: The Missing Semester of Your CS Education(shell)
date: 2022-04-01 13:48:00
tags: [The Missing Semester of Your CS]
description: 计算机教育中缺失的一课 The Missing Semester of Your CS Education
---

# shell

主要是想起来自己vim还不太会用，所以说记得这个课程的vim教学不错，干脆就花一晚上时间看看整套课程，重点看一下vim的使用，我看的版本是社区的[中文翻译版](https://missing-semester-cn.github.io/)的文档，这些工具大多我都已经能够熟练使用了，所以就没去看英文的视频感觉有点浪费时间。

首先的话shell在这个课程的第一课和第二课都讲，但是因为内容一样，所以说就并在一起写了。

## 课程概览与 shell

### 课程内容

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

### 习题

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

## Shell 工具和脚本

### 课程内容

#### 变量

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

#### 命令替换

通过 `$( CMD )` 这样的方式来执行`CMD` 这个命令时，它的输出结果会替换掉 `$( CMD )` 。

#### 进程替换

 `<( CMD )` 会执行 `CMD` 并将结果输出到一个临时文件中，并将 `<( CMD )` 替换成临时文件名。

#### 运行脚本

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

#### 通配（globbing）

##### 通配符

当你想要利用通配符进行匹配时，你可以分别使用 `?` 和 `*` 来匹配一个或任意个字符。

![image-20220401112506936](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220401112506936.png)

##### 花括号`{}`

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

#### shebang

对于如下代码

```python
#!/usr/local/bin/python
import sys
for arg in reversed(sys.argv[1:]):
    print(arg)
```

内核知道去用 python 解释器而不是 shell 命令来运行这段脚本，是因为脚本的开头第一行的 [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix))

#### shell工具

> 重要的是你要知道有些问题使用合适的工具就会迎刃而解，而具体选择哪个工具则不是那么重要。

##### find 

找文件，也可以用FD

##### grep

找文件内容

##### 查找 shell 命令

history 可以使用ctrl + R 进行搜索 也可以使用 | grep来找想要的历史命令

### 习题

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

当文件数量较多时，上面的解答会得出错误结果，解决办法是增加 `-mmin `条件，先将最近修改的文件进行初步筛选再交给ls进行排序显示 `find . -type f -mmin -60 -print0 | xargs -0 ls -lt | head -10`

![image-20220401134803463](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220401134803463.png)