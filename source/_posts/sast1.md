---
title: SAST工具入门（一）—— Codeql环境搭建与基础语法
date: 2023-03-21 20:10:00
updated: 2023-03-21 20:10:00
tags: ["security","sast"]
description: CodeQL的环境搭建与规则编写
category: Security
---

> 参考文章：https://www.freebuf.com/articles/web/283795.html

## 什么是CodeQL

简单来说，`CodeQL`就是一个静态分析（`SAST`）工具，可以在白盒场景通过编写`QL`制定的规则，自动化的扫描代码。

## 环境搭建

`CodeQL`分引擎和`SDK`两部分，引擎部分不开源，主要负责解析规则。`SDK`是开源的，包含很多漏洞规则，也可以自己写漏洞规则进行使用。

> 引擎部分：https://github.com/github/codeql-cli-binaries/releases

> SDK部分：https://github.com/github/codeql

引擎部分需要配置一下环境变量

![image-20230331162442920](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230331162442920.png)

`SDK`部分直接拉源代码就可以了

接下来拉一个项目，尝试一下`CodeQL`

这里我拉了这个[`Java`靶场](https://github.com/l4yn3/micro_service_seclab/)进行测试，拉下来后需要配一下数据库，确保项目可以正常运行。

接下来安装`vscode`的`codeql`插件，配置`codeql`所在的目录

![image-20230331162918241](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230331162918241.png)

在`java/ql/examples` 目录下创建`demo.ql`，内容为`select "Hello World`

，并且右键选择`CodeQL: Run Query`

![image-20230331163230009](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230331163230009.png)

到这里环境搭建的工作就完成了。

> 写起来比较简略，实际上还是踩了不少坑的，不过毕竟用Linux物理机，平常遇到奇奇怪怪的问题太多了，用搜索引擎结合一些文章就解决了

## 规则编写

> ql的语法和sql的语法有一些相似的地方

由于`CodeQL`实际的查询是对`AST`的查询，因此`QL`的类库是与`AST`对应的。

可以看一下`AST`的样子

![image-20230401112706932](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230401112706932.png)

### QL语法

```
from [datatype] var
where condition(var = something)
select var
```

example：

```
import java // 引入CodeQL类库
 
from int i // 表示所有int类型的数据
where i = 1 // 表示条件：当i等于1时
select i // 输出i
```

![image-20230401113356211](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230401113356211.png)

### 类库

`Method`:方法类，Method method表示获取当前项目中所有的方法

`MethodAccess`:方法调用类，MethodAccess call表示获取当前项目当中的所有方法调用

`Parameter`:参数类，Parameter表示获取当前项目当中所有的参数

example:

```
import java

from Method method // 表示所有方法
where method.hasName("getStudent") // 表示条件：方法名为getStudent
select method.getName(), method.getDeclaringType() // 输出方法的名称，和方法所属的类名
```

![image-20230401113424756](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230401113424756.png)

### 谓词

当`where`部分过长时，可以用谓词这个语法，把很长的查询语句封装成函数。

`predicate` 关键词用于声明谓词

`exists` 子查询，它根据内部的子查询返回true or false，来决定筛选出哪些数据。

利用上面这两个语法，我们可以把判断方法名称是否为`getStudent`的`where`部分，封装成函数。这个函数就被称为**谓词**。

```
import java

predicate isStudent(Method method){
	exists(|method.hasName("getStudent"))
} // |操作符返回查询结果的数量，大于0则true

from Method method
where isStudent(method)
select method.getName(), method.getDeclaringType().getName()
```

### Source、Sink

`SAST`的理念中通常会提到这个三元组`(source，sink，sanitizer)`

`source`是指漏洞污染链条的输入点。比如获取http请求的参数部分，就是非常明显的Source。

`sink`是指漏洞污染链条的执行点，比如SQL注入漏洞，最终执行SQL语句的函数就是sink(这个函数可能叫query或者exeSql，或者其它)。

`sanitizer`又叫净化函数，是指在整个的漏洞链条当中，如果存在一个方法阻断了整个传递链，那么这个方法就叫sanitizer。

#### 如何定义`source`

`source`，在我们这个java靶场中，具体来看就是后端接口的参数

```
    @RequestMapping(value = "/one")
    public List<Student> one(@RequestParam(value = "username") String username) {
        return indexLogic.getStudent(username);
    }
```

例如这段代码，`source`就是`username`参数

对于`source`的检测，我们可以用`CodeQL`的`SDK`提供的检测规则

```
override predicate isSource(DataFlow::Node src) { src instanceof RemoteFlowSource }
```

其中，`DataFlow::Node`表示一个数据流节点，可以是数据流源、汇或传输节点。`RemoteFlowSource`是一个表示远程数据流源的CodeQL类。

通俗的说，这里定义了一个叫`isSource`的谓词，来判断传入的这个节点是不是远程数据流源（`RemoteFlowSource`）。

#### 如何定义`sink`

这里我们以找sql注入的漏洞为例，`sink`就应该是`qurey`方法

```
override predicate isSink(DataFlow::Node sink) {
exists(Method method, MethodAccess call |
  method.hasName("query")
  and
  call.getMethod() = method and
  sink.asExpr() = call.getArgument(0)
)
}
```

这里我们声明谓词`isSink`，通过`exists`来判断名为`query`的方法，并且设置第一个参数为`sink`

### Data Flow

从`Source`向`Sink`的数据流是否能够走通决定了是否有可能存在漏洞，可以用`CodeQL`的语法`config.hasFlowPath(source, sink)`来判断。

## 尝试编写Demo检测SQL注入

前面分别讲了`Source`、`Sink`、`Data Flow`的定义方法，还需要一些语法将他们串起来，才是一个完成的demo。

`source`和`sink`的定义使用到的方法，需要继承自`TaintTracking::Configuration`类。

完整Demo：

```
/**
 * @id java/examples/demo
 * @name Sql-Injection
 * @description Sql-Injection
 * @kind path-problem
 * @problem.severity warning
 */

import java
import semmle.code.java.dataflow.FlowSources
import semmle.code.java.security.QueryInjection
import DataFlow::PathGraph


class VulConfig extends TaintTracking::Configuration {
  VulConfig() { this = "SqlInjectionConfig" }

  override predicate isSource(DataFlow::Node src) { src instanceof RemoteFlowSource }

  override predicate isSink(DataFlow::Node sink) {
    exists(Method method, MethodAccess call |
      method.hasName("query")
      and
      call.getMethod() = method and
      sink.asExpr() = call.getArgument(0)
    )
  }
}


from VulConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select source.getNode(), source, sink, "source"
```

![image-20230401135524256](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230401135524256.png)

非常的有意思，不过这里仍然存在问题，就是误报问题。

![image-20230401135912897](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230401135912897.png)

### 解决误报问题

这里也就是前面提到的三元组中，`sanitizer`的问题。

`CodeQL`默认规则中，没有对`List<Long>`这样的复合类型做判断，因此需要手动写一个`isSantizer`的谓词做判断，来解决误报的问题。

这段的意思是，如果当前`node`节点为基础类型、数字类型、泛型数字类型时，就切断数据流。

```
override predicate isSanitizer(DataFlow::Node node) {
    node.getType() instanceof PrimitiveType or
    node.getType() instanceof BoxedType or
    node.getType() instanceof NumberType or
    exists(ParameterizedType pt| node.getType() = pt and pt.getTypeArgument(0) instanceof NumberType )
  }
```

### 解决漏报问题

对于靶场中这段代码，没有捕获到。

```
    public List<Student> getStudentWithOptional(Optional<String> username) {
        String sqlWithOptional = "select * from students where username like '%" + username.get() + "%'";
        //String sql = "select * from students where username like ?";
        return jdbcTemplate.query(sqlWithOptional, ROW_MAPPER);
    }
```

这里是因为`CodeQL`的默认规则，没有对`Optional`这种类型做判断，这里可以选择手动添加对于`Optional`这种类型的检测。

```
predicate isTaintedString(Expr expSrc, Expr expDest) {
    exists(Method method, MethodAccess call, MethodAccess call1 | expSrc = call1.getArgument(0) and expDest=call and call.getMethod() = method and method.hasName("get") and method.getDeclaringType().toString() = "Optional<String>" and call1.getArgument(0).getType().toString() = "Optional<String>"  )
}
```

![image-20230401145345909](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230401145345909.png)



> 文章主要是本人的学习笔记，内容大多数都是对其他文章的参考，还是建议看一手文章进行学习
>
> 最后的ql文件 https://gist.github.com/ek1ng/f1c10a42a07b467a3989b422137b265a