---

title: SAST工具入门（二）—— Joern环境搭建与基础语法
date: 2023-03-22 19:49:00
updated: 2023-03-22 19:49:00
tags: ["security","sast"]
category: Security
---

> 参考文章：
>
> https://lorexxar.cn/2023/08/21/joern-and-cpg/
>
> https://lightless.me/archives/code-analysis-with-joern.html

## 环境搭建

> https://github.com/joernio/joern

环境配置很方便，看Readme用物理机安装或者Docker都可以。

需要java11+，java8会报错。

```
wget https://github.com/joernio/joern/releases/latest/download/joern-install.sh
chmod +x ./joern-install.sh
sudo ./joern-install.sh
```

## 简介

> 官方文档：https://docs.joern.io/

> Joern is developed with the goal of providing a useful tool for vulnerability discovery and research in static program analysis.

Joern的定位本身就是通过静态分析来辅助安全研究员挖洞的工具。

下面是官方给出的5个核心特点：

> - **Robust parsing.** Joern allows importing code even if a working build environment cannot be supplied or parts of the code are missing.

即便代码跑不起来，也能分析部分代码。像CodeQL是需要把代码编译成AST,再去AST数据库里面查询的，在这类项目上就有优势。

> - **Code Property Graphs.** Joern creates semantic code property graphs from the fuzzy parser output and stores them in an in-memory graph database. SCPGs are a language-agnostic intermediate representation of code designed for query-based code analysis.

Joern是基于CPG的，会将代码分析的结果创建成CPG存入图数据库。

> - **Taint Analysis.** Joern provides a taint-analysis engine that allows the propagation of attacker-controlled data in the code to be analyzed statically.

污点分析。

> - **Search Queries.** Joern offers a strongly-typed Scala-based extensible query language for code analysis based on Gremlin-Scala. This language can be used to manually formulate search queries for vulnerabilities as well as automatically infer them using machine learning techniques.

基于Scala的强类型可扩展查询语言。

> - **Extendable via CPG passes.** Code property graphs are multi-layered, offering information about code on different levels of abstraction. Joern comes with many default passes, but also allows users to add passes to include additional information in the graph, and extend the query language accordingly.

可扩展性，CPG是多层的，提供不同抽象级别的代码信息，也允许使用者在图中添加信息和扩展查询语言。

## 各种概念——什么是AST/CFG/PDG/CPG

> 需要一定的编译原理基础

各类SAST的工具基本都是先将代码分析成一种指定的数据结构（AST/CFG/PDG/CPG）并存入数据库，再通过设计特定的语法，来让使用者能够通过这些语法从数据库中查询出他们想要的函数调用关系，因此了解这些数据结构是很有必要的。

Joern作者的一篇文章[《Why Your Code Is A Graph》](https://blog.shiftleft.io/why-your-code-is-a-graph-f7b980eab740)讲解了为什么代码的关系可以用图表示，对这几种数据结构做了介绍。

### 抽象语法树 AST (Abstract Syntax Tree)

没有想到很好的概括性描述，维基百科：“AST是源代码语法结构的一种抽象表示”。

欧几里得算法（辗转相除法）的逻辑结构：

```
while b ≠ 0
  if a > b
    a := a − b
  else
    b := b − a
return a
```

用AST表示如下

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/0*0MsqUL3shyesZhCs.png)



### 控制流图 CFG (Control Flow Graph)

控制流是指程序中语句的顺序，而CFG 表示代码执行的顺序以及执行该代码段所需满足的条件。概念看起来有点抽象，其实就是下面这种图，看起来是比较直观的，其中节点表示程序的指令，边表示程序中的控制流。

![image-20231214203042060](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20231214203042060.png)

a表示的就是if-then-else，b表示的是while loop。

### 程序依赖图 PDG (Program Dependence Graph)

PDG 表示一段代码的数据和控制依赖性，其中有数据依赖边和控制依赖边的概念。

数据依赖边指定了一条数据的影响，控制依赖边制定谓词对其他节点的影响。

下面是一段示例代码和对应的PDG。

```
void example(){
  int x = 1;
  int y = 2;
  if (x > 1){
    int z = x + y;
    print(z);
  }
}
```

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/1*9w6vsnDi7mXyLG1vH5WcUQ.png)

其中D()表示数据依赖节点，C()表示控制依赖节点。

PDG可以直观的看到数据在代码中如何传输，和特定的数据能否传到特定的语句。

### 代码属性图 CPG (Code Property Graph)

AST/CFG/PDG其实都是在用一种“通用结构”解释代码，它们从不同角度表示代码。而CPG是这三者构成的一种图。途中用三种颜色表示了AST/CFG/PDG。

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/202308211430169.png)

## Joern的设计理念

> https://github.com/joernio/workshops/blob/master/2021-RSA/RSA-LAB2-R08.pdf
>
> https://blog.shiftleft.io/semantic-code-property-graphs-and-security-profiles-b3b5933517c1

## 基础语法

> 以java-sec-code为例来介绍

首先Joern通过控制台来从cpg中获取想要的信息

### 导入代码

> `importCode`

![image-20240213165014132](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20240213165014132.png)

### 小技巧

- `Tab`在控制台中可以补全关键字![image-20240213175635837](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20240213175635837.png)

- `toList`关键字可缩写为`l`

### 从cpg中查询信息

`cpg.metaData.l`/`cpg.metaData.toList`查询cpg的metaData

![image-20240213175705376](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20240213175705376.png)

`cpg.method.l`查询cpg的method

![image-20240213175558051](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20240213175558051.png)

获取所有method的name：`cpg.method.name.l`

获取所有method的某几个属性，例如name、lineNumber、code：`cpg.method.map(n=>List(n.name,n.lineNumber,n.code)).l`![image-20240213180233838](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20240213180233838.png)

获取指定method的某几个属性

`cpg.method.map(n=>List(n.name,n.lineNumber,n.code)).take(n).l`

![image-20240213180754165](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20240213180754165.png)
