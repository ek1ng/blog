---
title: The Missing Semester of Your CS Education(Command-line Environment)
date: 2022-04-04 14:00:00
tags: [The Missing Semester of Your CS]
description: 计算机教育中缺失的一课 The Missing Semester of Your CS Education
---

# Command-line Environment

> 学习如何同时执行多个不同的进程并追踪它们的状态、如何停止或暂停某个进程以及如何使进程在后台运行，学习一些能够改善您的 shell 及其他工具的工作流的方法，这主要是通过定义别名或基于配置文件对其进行配置来实现的。

主要就是讲使用命令行查看当前机器的进程和命令行环境的配置等内容。

## 任务控制

众所周知，`<C-c>`可以停止命令行命令的执行。

### 结束进程

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

### 暂停和后台执行进程

信号可以让进程做其他的事情，而不仅仅是终止它们。例如，`SIGSTOP` 会让进程暂停( `Ctrl-Z` )，我们可以使用 [`fg`](https://www.man7.org/linux/man-pages/man1/fg.1p.html) 或 [`bg`](http://man7.org/linux/man-pages/man1/bg.1p.html) 命令恢复暂停的工作。它们分别表示在前台继续或在后台继续，[`jobs`](http://man7.org/linux/man-pages/man1/jobs.1p.html) 命令会列出当前终端会话中尚未完成的全部任务。

![image-20220404120813276](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220404120813276.png)

后台的进程仍然是您的终端进程的子进程，一旦您关闭终端（会发送另外一个信号`SIGHUP`），这些后台的进程也会终止。为了防止这种情况发生，您可以使用 [`nohup`](https://www.man7.org/linux/man-pages/man1/nohup.1.html) (一个用来忽略 `SIGHUP` 的封装) 来运行程序。

比如我最近整了个qq机器人挂在协会的服务器上，那如果我需要让qq机器人在ssh连接断开的情况下继续运行，要么使用screen挂起一个终端，要么就用nohup让终端的关闭也不会影响qq机器人这个后台进程。可以使用百分号 + 任务编号（`jobs` 会打印任务编号）来选取该任务。

命令中的 `&` 后缀可以让命令在直接在后台运行，这使得您可以直接在 shell 中继续做其他操作。

下面的命令行交互过程演示了上面的一些知识，比如说用nohup挂起的当前终端的子进程2，因为用了nohup所以说`SIGHUP`这个信号就没法kill这个进程，当然如果直接kill这个进程还是可以的。

![image-20220404121246236](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220404121246236.png)

## 终端多路复用

当您在使用命令行时，您通常会希望同时执行多个任务。举例来说，您可以想要同时运行您的编辑器，并在终端的另外一侧执行程序。尽管再打开一个新的终端窗口也能达到目的，使用终端多路复用器则是一种更好的办法。

感觉目前没什么需求，有一定的学习成本然后学了后又不用容易忘，简单知道一下有tmux这么个工具吧

## 别名

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

## 配置文件（Dotfiles）

很多程序的配置都是通过纯文本格式的被称作*点文件*的配置文件来完成的（之所以称为点文件，是因为它们的文件名以 `.` 开头，例如 `~/.vimrc`。也正因为此，它们默认是隐藏文件，`ls`并不会显示它们）。

实际上，很多程序都要求您在 shell 的配置文件中包含一行类似 `export PATH="$PATH:/path/to/program/bin"` 的命令，这样才能确保这些程序能够被 shell 找到。

还有一些其他的工具也可以通过*点文件*进行配置：

- `bash` - `~/.bashrc`, `~/.bash_profile`
- `git` - `~/.gitconfig`
- `vim` - `~/.vimrc` 和 `~/.vim` 目录
- `ssh` - `~/.ssh/config`
- `tmux` - `~/.tmux.conf`

## 远端设备（ssh）

说到ssh就不得不推荐一下termius辣，在协会学长的推荐下用了这个ssh客户端，真不戳。

通过如下命令，您可以使用 `ssh` 连接到其他服务器：

```bash
ssh foo@bar.mit.edu
```

`ssh` 的一个经常被忽视的特性是它可以直接远程执行命令。 `ssh foobar@server ls` 可以直接在用foobar的命令下执行 `ls` 命令。 想要配合管道来使用也可以， `ssh foobar@server ls | grep PATTERN` 会在本地查询远端 `ls` 的输出而 `ls | ssh foobar@server grep PATTERN` 会在远端对本地 `ls` 输出的结果进行查询。

关于ssh远程执行命令这一点，在数据整理的内容中也有对应的运用。

### SSH 密钥

基于密钥的验证机制使用了密码学中的公钥，我们只需要向服务器证明客户端持有对应的私钥，而不需要公开其私钥。这样您就可以避免每次登录都输入密码的麻烦了秘密就可以登录。不过，私钥(通常是 `~/.ssh/id_rsa` 或者 `~/.ssh/id_ed25519`) 等效于您的密码，所以一定要好好保存它。

#### 密钥生成

使用 [`ssh-keygen`](http://man7.org/linux/man-pages/man1/ssh-keygen.1.html) 命令可以生成一对密钥：

```
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519
```

##### 基于密钥的认证机制

`ssh` 会查询 `.ssh/authorized_keys` 来确认那些用户可以被允许登录

### 通过 SSH 复制文件

- `ssh+tee`, 最简单的方法是执行 `ssh` 命令，然后通过这样的方法利用标准输入实现 `cat localfile | ssh remote_server tee serverfile`。回忆一下，[`tee`](https://www.man7.org/linux/man-pages/man1/tee.1.html) 命令会将标准输出写入到一个文件；
- [`scp`](https://www.man7.org/linux/man-pages/man1/scp.1.html) ：当需要拷贝大量的文件或目录时，使用`scp` 命令则更加方便，因为它可以方便的遍历相关路径。语法如下：`scp path/to/local_file remote_host:path/to/remote_file`；
- [`rsync`](https://www.man7.org/linux/man-pages/man1/rsync.1.html) 对 `scp` 进行了改进，它可以检测本地和远端的文件以防止重复拷贝。它还可以提供一些诸如符号连接、权限管理等精心打磨的功能。甚至还可以基于 `--partial`标记实现断点续传。`rsync` 的语法和`scp`类似；

#### 利用ssh实现监听远程设备的端口

#### 本地端口转发

本地端口转发，即远端设备上的服务监听一个端口，而您希望在本地设备上的一个端口建立连接并转发到远程端口上。

例如，我们在远端服务器上运行 Jupyter notebook 并监听 `8888` 端口。 然后，建立从本地端口 `9999` 的转发，使用 `ssh -L 9999:localhost:8888 foobar@remote_server` 。这样只需要访问本地的 `localhost:9999` 即可。

![localport](https://i.stack.imgur.com/a28N8.png%C2%A0)

#### 远程端口转发

感觉用处不大

![localport](https://i.stack.imgur.com/4iK3b.png )

## 课后练习

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
