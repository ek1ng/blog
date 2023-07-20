---
title: 云原生安全分享会材料
date: 2023-06-28 19:20:00
updated: 2023-06-28 20:45:00
tags: ["Cloud Security"]
category: Security
---

> 这是一篇用于给协会小学弟们分享的文章，粗略从各个角度讲了一讲，有任何问题都欢迎联系我交流，email：ek1ng@qq.com。

## 基础知识🧀

在开始之前，你需要能够基本掌握Docker和Kubernetes的使用。

基本使用推荐看官方文档，配合一些教程动手尝试。

> https://www.docker.com/

Docker 能区分镜像/容器，能基本使用命令，能写Dockerfile，粗略了解原理即可。

> https://kubernetes.io/zh-cn/
>
> https://github.com/guangzhengli/k8s-tutorials

K8s 可以看官方文档或者上面这个教程，用minikube（一个本地单机器部署的模拟k8s的项目）搭一个环境尝试一下。

## 相关资料🤯

> k8s靶场：
>
> https://github.com/madhuakula/kubernetes-goat
>
> 云原生安全红队工具CDK：
>
> https://github.com/cdk-team/CDK
>
> 云原生安全安全检查项目： trivy：静态安全风险检查
>
> https://github.com/aquasecurity/trivy
>
> Veinmind-tools: 和trivy差不多定位的竞品
>
> https://github.com/chaitin/veinmind-tools
>
> Elkeid：静态+动态入侵检测/日志审计
>
> https://github.com/bytedance/Elkeid 一些教程：
>
> https://github.com/neargle/my-re0-k8s-security
>
> https://wiki.teamssix.com/cloudnative/
>
> 相关资料库：
>
> https://github.com/4ndersonLin/awesome-cloud-security
>
> 入门后非常推荐学习的 国外的云安全研究团队博客 Wiz：
>
> https://www.wiz.io/blog
>
> 比如他们发表的这篇阿里云 云数据库的漏洞，值得学习https://www.wiz.io/blog/brokensesame-accidental-write-permissions-to-private-registry-allowed-potential-r#analyticdb-for-postgresql
>
> 可以看这篇我大概翻译了一下写的中文
>
> https://ek1ng.com/BrokenSesame.html
>
> 腾讯云鼎实验室公开的一份资料：
>
> https://ek1ng.oss-cn-hangzhou.aliyuncs.com/%E4%BA%91%E4%B8%8A%E5%AE%89%E5%85%A8%E6%94%BB%E9%98%B2%E5%AE%9E%E6%88%98%E6%89%8B%E5%86%8C.pdf

## Docker🐳

> 基本的使用相信都会了，这里讲一些隔离的原理，帮助更好的理解什么时候会有漏洞，漏洞是怎么产生的，如何避免。 
>
> https://juejin.cn/post/7096115284875935781

### 原理

首先从理念上，相较于传统vm有一个独立的操作系统，docker本质还是一个运行在宿主机上的进程，容器共享宿主机的OS。

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/asynccode)

#### Linux Namespace（命名空间）

> 一个宿主机可以运行多个容器，那么多个容器的IP和端口怎么分配，主机名怎么分配，每个容器都有root用户，怎么解决用户重名问题，这些都是通过namespace解决的。

Namespace（命名空间）是Linux 内核的一个特性，是对全局系统资源封装隔离。 

Docker 主要有以下6种资源的隔离

1. Pid Namespace（进程隔离）
2. Net Namespace（网络隔离）
3. User Namespace（用户隔离）
4. IPC Namespace（进程间通信隔离）
5. MNT Namespace（磁盘挂载点和文件系统隔离）
6. UTS Namespace（主机名隔离）

```JavaScript
/proc/<pid>/ns
```

#### Linux Cgroup（控制组）

> Cgroup 是 control group的缩写，用来实现CPU、内存、硬盘等方面的隔离。

```JavaScript
/proc/<pid>/cgroup
```

#### Linux Capabilities

> capabilities是Linux用来划分进程的权限的，通常我们会通过查看CAP有哪些，来看我们在当前的场景有什么权限做什么事情，如果有一些比较高的权限，就很容易能打破某些隔离机制，完成利用。

```JavaScript
/proc/self/status |grep CapEff -> CapEff
capsh --decode=<CapEff>
```

### 漏洞

> 正如上面原理所说，Docker本质就是这么一项基于Linux namespace和cgroup的虚拟化技术，所以问题大多数都是由于各种资源的虚拟化隔离出了问题导致的，下面利用手法基本覆盖百分之80以上情景。

#### 文件隔离

> 文件隔离是最简单利用的，技巧非常多，大多数的出发点都是，容器挂载了宿主机的危险目录，或者两个容器共享某个目录，导致逃逸到宿主机，逃逸到另一个容器等风险。

##### 共享`/`

写定时任务，ssh key，动态链接库，覆盖可执行程序，写环境变量等等，方法很多。

##### 共享`/proc`

> https://wiki.teamssix.com/cloudnative/docker/docker-procfs-escape.html
>
> 从2.6.19内核版本开始，Linux支持在/proc/sys/kernel/core_pattern中使用新语法。如果该文件中的首个字符是管道符|，那么该行的剩余内容将被当作用户空间程序或脚本解释并执行。

通过共享的`/proc`目录，写宿主机的`/proc/sys/kernel/core_pattern`

##### 共享`docker.sock`

> 在Docker in docker的场景中会出现共享docker.sock

可以在容器内通过docker.sock，调用到宿主机的docker命令，创建一个特权容器然后挂载宿主机目录然后逃逸即可。

##### 漏洞实例

> 一个通过文件隔离和Pid隔离被打破导致的容器逃逸的例子。

例如Wiz的这篇文章中，https://www.wiz.io/blog/brokensesame-accidental-write-permissions-to-private-registry-allowed-potential-r。

阿里云AnalyticDB for PostgreSQL的云数据库服务，作者提权到容器内root权限后，阿里云有一个开启`SSL`加密的功能，是通过创建一个新容器，并且往数据库容器里面写文件实现的。

由于PID没有隔离，可以通过`/proc/xxx/root`访问到新容器的文件系统。

又发现新容器和云数据库容器共享相同的`/home/adbpgadmin`目录，通过写`/home/adbpgadmin/.ssh/config`的`LocalCommand`字段，完成了从云数据库容器，逃逸到这个 点击 开启`SSL`加密的 按钮后，创建出来的新容器，来做下一步利用。

#### 进程隔离

> 启动容器时用 -pid=host 会共用pid namespace，打破进程隔离，但是还需要有注入进程的权限，才能够加以利用。

##### cap_sys_ptrace && –pid=host

> https://zone.huoxian.cn/d/1071-docker

打破进程隔离，并且有`cap_sys_ptrace`权限（允许注入进程）的情况下，可以完成逃逸。

#### Cgroup隔离

> 通过挂载宿主机cgroup，打破Cgroup隔离导致容器逃逸的手法
>
> https://blog.trailofbits.com/2019/07/19/understanding-docker-container-escapes/

##### cap_sys_admin

cap_sys_admin是一种高权限的capabilities，常见利用手法是挂载cgroup，来实现逃逸，具体分下面两种经典的利用手法，详细就看上面文章吧。

- 劫持`release_agent`，利用cgroup的notify-on-release机制触发shellcode实现逃逸
- 重写cgroup的devices.allow实现逃逸

##### lxcfs

> kubernetes的场景

当`POD`挂载了`lxcfs`目录包含`cgroup`目录的情形，且有`cgroup`目录写权限的情况

#### 内核问题

##### cap_sys_module

cap_sys_module 权限允许加载内核模块，如果在容器里加载一个恶意的内核模块，将直接导致逃逸。

##### 共享宿主机内核

由于docker是共享宿主机内核版本的，所以如果宿主机内核版本过低，可以通过打内核洞提权到宿主机root权限，经典的就是dirty cow，dirty pipes。

> https://wiki.teamssix.com/CloudNative/Docker/CVE-2022-0847-dirty-pipe.html

```JavaScript
uname -r //查看内核版本
```

#### Docker api 2375 未授权访问

> 比较少见的情形
>
> https://www.cnblogs.com/-mo-/p/11529387.html

Docker remote api本身用于通过请求api来执行docker命令，如果开放对外访问，那么和挂载`docker.sock`的情形类似，相当于可以调用宿主机的docker命令，创建特权容器挂载目录写文件，就能利用。

#### Docker CVE

> 以上都是开发者对docker这项技术使用不当导致的，docker当然本身也出过一些CVE，导致在某些情况能够完成利用。

比较经典的CVE可以看一下这俩 都是有poc的 漏洞本身是pwn web手会用exp即可。

- CVE-2019-5736 docker-runc 逃逸
- CVE-2020–15257 container-shim 逃逸

## K8s☸️

### 原理

推荐尝试亲手用minikube/k3s/k8s搭建环境，通过一些基本操作了解k8s常见的资源类型（Pod，Deployment，Service，Namespace，cronjob，Secret），组件（etcd，kube-apiserer，kube-controller-manager，kube-scheduler），相关的一些生态产品（K9s，kubersphere）

### 漏洞

#### 组件接口未授权访问

> 各种组件接口的未授权访问

API Server 8080 6443

Etcd 2379,2389

Kubelet 10250,20248

Dashboard 8001

#### 访问权限控制（RBAC）

> RBAC（基于角色的访问控制）是目前kubernetes的推荐的访问权限控制机制。
>
> https://github.com/cdk-team/CDK/wiki/Exploit:-k8s-get-sa-token

`Role`：特定namespace下的角色，namespace级别的权限控制。

`ClusterRole`：集群角色，集群级别的权限控制。

`RoleBinding`：绑定`Role`的权限。

`ClusterRoleBinding`：绑定`ClusterRoleBinding`的权限。

如果当前Pod有创建Pod权限，可以创建一个挂载service-account token为admin的Pod，提权到`Cluster Admin`。

#### 网络隔离

Kubernetes 默认 Pod 之间网络连通，因此如果拿下一个 Pod 的权限，可以通过传统的内网横向思路，来进行信息搜集，并且可能在 Pod 之间横向移动。

#### Kubernetes CVE

- CVE-2018-1002105

## 攻击方视角：渗透、挖洞

### 收集信息

> 收集信息非常重要，不论是到了哪一步，都应该清楚的知道现在在什么环境中，自己有什么权限，从而去考虑自己可以做哪些事情。

#### 判断自己在什么环境

> 物理机，KVM，dockerd，containerd等等

```JavaScript
cat /proc/1/cgroup
systemd-detect-virt // 非常好用
```

#### 判断自己有什么权限

##### cdk

> cdk是一个非常不错的工具，文档也有，可以看cdk的代码学习一些场景场景的利用手法。
>
> https://github.com/cdk-team/CDK/wiki/

```JavaScript
cdk evalute
```

##### 手动收集

环境变量

> 看有没有敏感信息

```
env
```

内核

```
uname -r
```

###### 定时任务

```
cd /var/spool/cron
crontab -l
```

###### 进程信息

> 如果pid = 1是init 一般是docker
>
> 也看看有哪些服务啥的

```
ps -ef
```

###### 文件隔离

> 看一下挂载了哪些目录

```
cat /proc/<pid>/mounts
```

网络隔离

> Nmap fscan之类都行 和传统渗透一样

###### Cgroup

```
cat /proc/<pid>/cgroup
```

###### Capabilites

`/proc/self/status |grep CapEff` -> CapEff

```
capsh --decode=<CapEff>
```

## 防守方视角：检测攻击行为和后渗透行为

### 传统主机安全场景

云原生场景只是把服务部署到`kubernetes`这些上，对于传统主机安全领域的安全问题仍然存在，比如说资产有web洞，服务有弱密码，被写后门写webshell写内存马的检测等等这些问题其实在防守方视角，不论是上云了还是没上，都存在。

### 云原生场景的差异

第一，上云了之后，即便打下web服务也不能摸到宿主机，多了docker/k8s等等这一层；第二，上云了之后，有更多的云场景的利用手法，比如说对于一个k8s的环境，安全检测的产品除了传统的主机安全问题，还需要考虑比如说有没有可能有deamon-set后门，有没有可能有docker逃逸风险，等等这些。具体的可以看看一下几个项目的源码

trivy：`https://github.com/aquasecurity/trivy`

trivy是一个静态的云原生安全场景风险检测的项目，18k的star，有各种各样的**静态**安全风险检测功能。

Elkeid：`https://github.com/bytedance/Elkeid`

字节开源的云原生安全场景的主机安全防护平台，有`agent`，可以做**动态**的检测。

静态的工具，就是不需要额外配置，可以在本地或者接入`CI/CD`的流程中，用来扫描比如说`Dockerfile`，`Kubernetes`的配置文件（IaC），来或者扫描镜像/容器的情况，来看有没有一些安全风险。

而动态的工具，一般会有一个`Agent`，这个`Agent`会需要部署到你的机器上，这个`Agent`会实时的上报信息，然后通过写好的安全策略引擎来做检测，看当前有没有发生入侵行为等等。

像动态检测的工具，由于有`Agent`的存在，可以做到比如说实时检测是否有反弹shell的进程，静态的就没法做这个事情，因为反弹shell可能就一小会的时间，是需要实时`hook`的情况才有价值，因此我认为纯静态检测的工具可能更多的意义就是放在`CI/CD`之类的环节，来做一些安全预警的工作，而有部署`agent`的`hids`产品，就有更多的事件源来支持安全策略引擎，做更多的事情。