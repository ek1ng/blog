---
title: Rego在云原生安全场景的使用
date: 2023-04-19 11:36:00
updated: 2023-04-19 18:15:00
tags: ["Cloud Security"]
description: 主要学习了Trivy是如何使用Rego做iac配置检测的
category: Security
---

> 作者仅是云原生安全相关和opa相关生态的初学者，在此分享一些学习笔记和经验总结，以下是参考文章：
> 
> http://blog.newbmiao.com/2020/03/13/opa-quick-start.html
> 
> https://github.com/NewbMiao/opa-koans
> 
> https://moelove.info/2021/12/06/Open-Policy-Agent-OPA-%E5%85%A5%E9%97%A8%E5%AE%9E%E8%B7%B5/

## Rego和OPA

Rego 是一种声明式、功能强大的策略语言，由 Open Policy Agent（OPA）项目开发并维护。OPA 是一个开源、轻量级、云原生的安全和策略引擎，用于在微服务、云和 Kubernetes 中实现统一的访问控制和策略管理。

## OPA解决的问题

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/paN9tjO46QM3RbT.png)

可以看到，很多云原生生态相关的产品，都需要通过配置好的策略来进行控制，比如Dockerfile，K8s等等。而在不同的场景中，他们都需要和不同的开发者开发的应用做耦合，难以管理，OPA可以做到统一配置的策略，降低维护成本，方便服用的工作。

## 声明式策略语言Rego

 那么OPA是如何实现统一配置策略呢，就是通过将业务和策略分离，将策略统一发送给引擎，引擎通过规则来进行决策。

![image-20230419153130555](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230419153130555.png)

从图中可以看到，当请求进来时，OPA会通过Rego来“翻译”规则，并且根据Rego的结果，进行对应的决策，发送给Service。

### 语法

> 也许您只是想浏览一下文章大意，那么可以略过语法这一部分。
> 
> 如果想尝试一下，可以使用在线[运行网站](https://play.openpolicyagent.org/p/ZXkIlAEPCY)或者安装opa的命令行工具又或者是vsc，jetbrains的opa插件。

有几个比较抽象的概念，这里简单介绍一下，但是还是建议看[官方文档](https://www.openpolicyagent.org/docs/latest/policy-language/)，学语法二手资料实在看起来折磨...

定义变量

```
pi := 3.14159
rect := {"width": 2, "height": 4}
```

比较

```
rect == {"height": 4, "width": 2} //比较复合值
v if "hello" == "world" //表达式为真则v为真
```

`_`：如果变量在引用之外未使用，我们更愿意将它们替换为下划线 (_) 字符。

同名函数

>  **多个同名函数，满足一个就为true**

对于这样的输入，来写一个规则判断配置文件的后缀，只要满足一个，那么函数的返回就是true

```json
{
  "files": [
  {
    "type": "posix",
    "path": "/Users/newbmiao/Documents/1.yaml"
  },
  {
    "type": "posix",
    "path": "/Users/newbmiao/Documents/2.yaml"
  },
  {
    "type": "traditional-mac",
    "path": "Macintosh HD:Users:newbmiao:Documents:3.yml"
  },
  {
    "type": "traditional-mac",
    "path": "Macintosh HD:Users:newbmiao:Documents:3.json"
  }
  ]
}
```

```
is_config_file(str) {
    contains(str, ".yaml")
}

is_config_file(str) {
    contains(str, ".yml")
}

is_config_file(str) [
    contains(str, ".json")
]
```

剩下的虚拟文档，推导式等等就先没看了，简单了解了一下语法

## 实践

这部分主要是学习[Trivy](https://github.com/aquasecurity/trivy)这个开源的容器安全扫描器中对于rego的使用，主要是在iac的安全检查这一块的应用。相比于直接用正则匹配，用rego来写规则，对Dockerfile或者k8s的配置文件等进行检查，会更易于编写也能够更准确的匹配。

其中检测规则这部分主要在这个仓库https://github.com/aquasecurity/defsec/tree/master/rules

这里看一个简单的例子，按照官方的Dockerfile编写最佳实践，如果不解压tar`文件，最好用COPY取代ADD，那么如果我们想对Dockerfile做检测怎么办呢，Trivy的这个检测部分就是先将Dockerfile解析成JSON，然后根据传入的JSON,用写好的rego规则进行匹配。

https://github.com/aquasecurity/defsec/blob/master/rules/docker/policies/add_instead_of_copy.rego

```
# METADATA
# title: ADD instead of COPY
# description: You should use COPY instead of ADD unless you want to extract a tar file. Note that an ADD command will extract a tar file, which adds the risk of Zip-based vulnerabilities. Accordingly, it is advised to use a COPY command, which does not extract tar files.
# scope: package
# related_resources:
# - https://docs.docker.com/engine/reference/builder/#add
# schemas:
# - input: schema["dockerfile"]
# custom:
#   id: DS005
#   avd_id: AVD-DS-0005
#   severity: LOW
#   short_code: use-copy-over-add
#   recommended_action: Use COPY instead of ADD
#   input:
#     selector:
#     - type: dockerfile
package builtin.dockerfile.DS005

import data.lib.docker

get_add[output] {
    add := docker.add[_]
    args := concat(" ", add.Value)

    not contains(args, ".tar")
    output := {
        "args": args,
        "cmd": add,
    }
}

deny[res] {
    output := get_add[_]
    msg := sprintf("Consider using 'COPY %s' command instead of 'ADD %s'", [output.args, output.args])
    res := result.new(msg, output.cmd)
}
```

这个规则比较好懂，先导入了`data.lib.docker`这个包中的`add`，那么看一下他们抽出来的这个`lib`库做了什么。

https://github.com/aquasecurity/defsec/blob/master/rules/docker/lib/docker.rego

```
add[instruction] {
    instruction := input.Stages[_].Commands[_]
    instruction.Cmd == "add"
}
```

主要是做了封装，对传入的JSON根据Dockerfile中的具体命令做匹配，比如说`ADD`，`COPY`等等，做对应的封装。

`get_add`函数做的就是获取`Dockerfile`中`ADD`命令所对应的内容，检测是否包含`.tar`。

`deny`函数做的是调用`get_add`，如果有对应的`output`，那么就会触发`deny`，往`result`中添加结果。
