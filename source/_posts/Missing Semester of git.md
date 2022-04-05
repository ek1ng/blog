---
title: The Missing Semester of Your CS Education(git)
date: 2022-04-05 22:35:00
tags: [The Missing Semester of Your CS]
description: 计算机教育中缺失的一课 The Missing Semester of Your CS Education
---

# Git

## Git 的数据模型

​		Git 拥有一个经过精心设计的模型，这使其能够支持版本控制所需的所有特性，例如维护历史记录、支持分支和促进协作。

### 快照

​		在Git的术语里，文件被称作Blob对象（数据对象），也就是一组数据。目录则被称之为“树”，它将名字与 Blob 对象或树对象进行映射（使得目录中可以包含其他目录）。快照则是被追踪的最顶层的树。例如，一个树看起来可能是这样的：

```
<root> (tree)
|
+- foo (tree)
|  |
|  + bar.txt (blob, contents = "hello world")
|
+- baz.txt (blob, contents = "git is wonderful")
```

​		这个顶层的树包含了两个元素，一个名为 “foo” 的树（它本身包含了一个blob对象 “bar.txt”），以及一个 blob 对象 “baz.txt”。

### 历史记录建模：关联快照

​		版本控制系统和快照有什么关系呢？线性历史记录是一种最简单的模型，它包含了一组按照时间顺序线性排列的快照。不过处于种种原因，Git 并没有采用这样的模型。

​		在 Git 中，历史记录是一个由快照组成的有向无环图。注意，快照具有多个“父辈”而非一个，因为某个快照可能由多个父辈而来。例如，经过合并后的两条分支。

​		在 Git 中，这些快照被称为“提交”。通过可视化的方式来表示这些历史提交记录时，看起来差不多是这样的：

```
o <-- o <-- o <-- o
            ^  
             \
              --- o <-- o
```

​		上面是一个 ASCII 码构成的简图，其中的 `o` 表示一次提交（快照）。

​		箭头指向了当前提交的父辈（这是一种“在。。。之前”，而不是“在。。。之后”的关系）。

### 数据模型及其伪代码表示

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

### 对象和内存寻址

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

### 引用

至此commit就可以用SHA-1哈希值来标记了，但是哈希值并不好记，因此需要引用（references），引用是指向提交的指针。与对象不同的是，它是可变的（引用可以被更新，指向新的提交）。例如，`master` 引用通常会指向主分支的最新一次提交。这样，Git 就可以使用诸如 “master” 这样人类可读的名称来表示历史记录中某个特定的提交，而不需要在使用一长串十六进制字符了。

通常情况下，我们会想要知道“我们当前所在位置”，并将其标记下来。这样当我们创建新的提交的时候，我们就可以知道它的相对位置（如何设置它的“父辈”）。在 Git 中，我们当前的位置有一个特殊的索引，它就是 “HEAD”。

### 仓库

现在我们可以给出 Git 仓库的定义：`对象` 和 `引用`。

在硬盘上，Git 仅存储对象和引用：因为其数据模型仅包含这些东西。所有的 `git` 命令都对应着对提交树的操作，例如增加对象，增加或删除引用。

## 暂存区

暂存区和数据模型不相关，但是它是创捷提交接口的一部分。

我们先来理解下 Git 工作区、暂存区和版本库概念：

- **工作区：**就是你在电脑里能看到的目录。
- **暂存区：**英文叫 stage 或 index。一般存放在 **.git** 目录下的 index 文件（.git/index）中，所以我们把暂存区有时也叫作索引（index）。
- **版本库：**工作区有一个隐藏目录 **.git**，这个不算工作区，而是 Git 的版本库。

下面这个图画的非常清晰了

![image-20220405212445433](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220405212445433.png)

## Git常用操作

![image-20220405212728387](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220405212728387.png)

### 基础

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

### 分支和合并

- `git branch`: 显示分支

- `git branch <name>`: 创建分支

- `git checkout -b <name>`: 创建分支并切换到该分支
  - 相当于 `git branch <name>; git checkout <name>`
  
- `git merge <revision>`: 合并到当前分支

- `git mergetool`: 使用工具来处理合并冲突

- `git rebase`: 将一系列补丁变基（rebase）为新的基线

### 远端操作

- `git remote`: 列出远端
- `git remote add <name> <url>`: 添加一个远端
- `git push <remote> <local branch>:<remote branch>`: 将对象传送至远端并更新远端引用
- `git branch --set-upstream-to=<remote>/<remote branch>`: 创建本地和远端分支的关联关系
- `git fetch`: 从远端获取对象/索引
- `git pull`: 相当于 `git fetch; git merge`
- `git clone`: 从远端下载仓库

### 撤销

- `git commit --amend`: 编辑提交的内容或信息
- `git reset HEAD <file>`: 恢复暂存的文件
- `git checkout -- <file>`: 丢弃修改

### Git 高级操作

- `git config`: Git 是一个 [高度可定制的](https://git-scm.com/docs/git-config) 工具
- `git clone --depth=1`: 浅克隆（shallow clone），不包括完整的版本历史信息
- `git add -p`: 交互式暂存
- `git rebase -i`: 交互式变基
- `git blame`: 查看最后修改某行的人
- `git stash`: 暂时移除工作目录下的修改内容
- `git bisect`: 通过二分查找搜索历史记录
- `.gitignore`: [指定](https://git-scm.com/docs/gitignore) 故意不追踪的文件

## 课后练习

>1. 如果您之前从来没有用过 Git，推荐您阅读 [Pro Git](https://git-scm.com/book/en/v2) 的前几章，或者完成像 [Learn Git Branching](https://learngitbranching.js.org/)这样的教程。重点关注 Git 命令和数据模型相关内容；

我决定看一下Git Branching，之前就看过这个小游戏觉得不错，但是一直没做一下。下面记录一下自己还没搞明白的一些基础git 命令

### 如何切换当前指向的commit记录?

`git branch`是新建分支 `git checkout`是对HEAD指向的分支进行操作，比如说可以`git checkout <branchname>`切换分支，可以用`git checkout <hash> `来分离HEAD，HEAD会自动指向master，会表示成master* ，那比如说当前master指向C2，HEAD -> master -> C2，执行后会变成HEAD -> C2。HEAD就是我们当前指向的commit记录，那通过这种方式我们就可以切换HEAD到一个我们想更改的commit记录了。

### 如何撤销更改?

`git reset`本地仓库commit回滚

`git revert`远程仓库commit回滚，但是会生成新的commit记录，并不是消除撤销的记录。

### 如何合并分支？

`git merge <branchname>` 将HEAD指向的分支和<branchname>合并，生成一个新commit。

`git rebase <branchname>`将HEAD指向的分支的不同commit记录（也就是两个分支的有差异的commit记录）移动到<branchname>分支上，变成顺序关系.

上面这些可以解决90%问题了，剩下的话具体遇到问题再查吧，命令太多真有点记不住了。

看了看其他课程练习，就是对这个课程网站的git仓库查一查commit记录，其他都比较熟练了，查某行更改时谁的话用`git blame`就可以，其他感觉平时不太用的到，先把上面的记清楚吧，我现在也不是很熟练回滚和合并分支的操作，只会`git add . ` `git commit -m ""` `git psuh` XD



