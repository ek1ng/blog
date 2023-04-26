---
title: Kubenetes 入门学习
date: 2023-04-25 11:36:00
updated: 2023-04-26 18:15:00
tags: [Cloud Security]
description: k8s的一些学习笔记
category: Operations
---

仅为学习笔记，建议参考如下文档

> https://kubernetes.io/zh-cn/docs/home/
>
> https://github.com/guangzhengli/k8s-tutorials
>
> https://minikube.sigs.k8s.io/docs/

# 基础概念

## K8s组件

![Components of Kubernetes](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/components-of-kubernetes.svg)

### Control Plane Components

控制平面组件主要为集群做全局决策，比如资源调度，以及检测和响应集群事件。以下是控制平面的组件。

#### kube-apiserver

负责公开kubernetes api，处理接受请求。api server是K8s控制平面的前端。

#### etcd

一致且高可用的键值存储，是K8s所有集群数据的后台数据库。

#### kube-scheduler

负责监视新创建的，未指定运行节点的Pods，并选择节点来让Pod在上面运行。

#### kube-controller-manager

负责运行控制器进程。

控制器包括：

1. 节点控制器（Node Controller）：负责在节点出现故障时进行通知和响应
2. 任务控制器（Job Controller）：监测代表一次性任务的 Job 对象，然后创建 Pods 来运行这些任务直至完成
3. 端点分片控制器（EndpointSlice controller）：填充端点分片（EndpointSlice）对象（以提供 Service 和 Pod 之间的链接）。
4. 服务账号控制器（ServiceAccount controller）：为新的命名空间创建默认的服务账号（ServiceAccount）。

控制器会被编译到同一个可执行文件，并且在同一个进程中运行。

#### cloud-controller-manager

仅运行特定于云平台的控制器，将你的集群连接到云提供商的API，并且将与该云平台交互的组件和与集群交互的组件分离开来。因此如果在自己的环境中运行k8s，集群不需要有cloud-controller-manager。

### Node Components

节点组件在每个Node上运行，负责维护运行的Pod并提供K8s运行环境。

#### kubelet

kubelet保证了containers都运行在Pod中。它接受一组通过各类机制提供的PodSpecs，确保描述的容器处于运行状态且健康。kubelet不会管理不是由K8s创建的容器。

#### kube-proxy

kube-proxy是集群每个Node上运行的网络代理，实现K8s Service概念的一部分。维护节点上的一些网络规则，这些网络规则会允许从集群内部或外部的网络会话与Pod进行网络通信。

#### Container Runtime

容器运行时是负责运行容器的软件。

### Addons

插件使用K8s资源实现集群级别功能。

#### DNS

插件不是必需组件，但是几乎所有K8s集群都有集群DNS。集群DNS是一个DNS服务器，和其他DNS服务器一起工作，为K8s服务提供DNS记录。

#### DashBorad

基于Web的用户界面。

#### Container Resource Monitoring

容器资源监控将关于容器的一些常见的时间序列度量值保存到一个集中的数据库中，并提供浏览这些数据的界面。

#### Cluster-level Logging

集群层面日志机制将容器的日志数据保存到一个集中的日志存储中，并提供搜索和浏览接口。

# 实践

## Environment

minikube可以在本地模拟一个单节点的K8s集群，是一个专门用于学习K8s的工具。

我先搭建了一个minikube的环境。

![image-20230425174117020](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230425174117020.png)

## Pod

在生产环境的部署中，从一开始的单机部署多个服务，存在资源难以分配的问题；到单机运行多个虚拟机，但虚拟机性能损耗比较大大；再到容器化部署，容器本身是一个轻量虚拟机，性能损耗小，并且资源隔离可以按需分配，统一环境，解决环境差异导致的问题，多种好处都让容器化成为了更加现代的部署选择。

![img](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/kwmxgxwp.svg)

那么如果我们仅需要用一台机器运行单个/多个容器来提供支持，那其实Docker就完全够用；如果仅是三四台机器，我们也可以给每台机器单独配置运行环境，并且配置负载均衡，但是当机器变得更多的时候，就需要K8s这样的工具，来对更多的机器以及容器进行轻便且稳定的管理。

而K8s中有一个Pod的概念，**Pod**就是K8s中创建和管理的最小的可部署计算单元，一个Pod可以有多个Container。

![image-20230425183318472](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230425183318472.png)

![image-20230425183437073](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230425183437073.png)

Container的本质是进程，而Pod是管理这一组进程的资源。

接下来尝试创建一个Pod！

```
# nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx-container
      image: nginx
```

`kind`表示创建资源类型，`metadata.name`表示要创建的`pod`名称，`spec.containers`表示要运行的容器名称和镜像名称。

```
kubectl apply -f nginx.yaml //create nginx Pod
pod/nginx-pod created

kubectl get pods //check pod status
nginx-pod   1/1     Running   0             47s

kubectl port-forward nginx-pod 4000:80 // 映射 nginx 端口
Forwarding from 127.0.0.1:4000 -> 80
Forwarding from [::1]:4000 -> 80

kubectl exec -it nginx-pod /bin/bash // 进入容器

echo "hello kubernetes by nginx!" > /usr/share/nginx/html/index.html //配置nginx首页内容
```

最后我们就可以成功访问到nginx服务，完成一个简单的Pod部署。

![image-20230425183007591](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230425183007591.png)

## Deployment

上面我们是手动对Pod进行了管理，但是在生产环境中，一般不会直接管理Pod。像是对于版本升级、扩容等操作，不可能给每一个Pod都手动的去升级，所以这里就需要K8s来自动化的管理，这里就需要K8s的另一个资源**Deployment**。

```
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
        - image: guangzhengli/hellok8s:v1
          name: hellok8s-container
```

kind 表示 资源类型为 deployment

metadata.name表示要创建的deployment名称

spec中，replicas表示部署pod副本数量，selector表示deployment资源和pod资源关联的方式，这里表示deployment会管理(selector)所有lables为hellok8s的pod。

template的内容是用来定义pod资源的，和前面我们直接定义一个pod一样，区别是要加上metadata.labels表示被deployment管理，而metadata.name是会被deployment自动创建的。

接下来创建一个deployment来感受一下如何用k8s手动管理。

```
kubectl apply -f deployment.yaml
deployment.apps/hellok8s-deployment created

kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
hellok8s-deployment   0/1     1            0           15s

kubectl get pods
NAME                                   READY   STATUS    RESTARTS      AGE
hellok8s-deployment-594f97b78d-7mpj2   1/1     Running   0             51s
nginx-pod                              1/1     Running   0             17h

kubectl delete pod hellok8s-deployment-594f97b78d-7mpj2
pod "hellok8s-deployment-594f97b78d-7mpj2" deleted

kubectl get pods   
NAME                                   READY   STATUS    RESTARTS   AGE
hellok8s-deployment-594f97b78d-ntb24   1/1     Running   0          6s
nginx-pod                              1/1     Running   0          17h
```

删除一个pod后，deployment会自动创建一个新的pod，这其实就表面和之前手动管理的pod不同，deployment会根据配置文件的定义来进行维护，而我们要做的就是维护好deployment.yaml。

### 扩容

将replicas改成3，来研究一下扩容。

```
root@OPS-4152:/home/ek1ng# kubectl apply -f deployment.yaml 
deployment.apps/hellok8s-deployment configured
root@OPS-4152:/home/ek1ng# kubectl get pods   
NAME                                   READY   STATUS    RESTARTS   AGE
hellok8s-deployment-594f97b78d-688b8   1/1     Running   0          5s
hellok8s-deployment-594f97b78d-8zlmx   1/1     Running   0          5s
hellok8s-deployment-594f97b78d-ntb24   1/1     Running   0          5m29s
nginx-pod                              1/1     Running   0          17h
```

### 版本更新

构建一个v2版本镜像，研究一下版本升级。

这里我直接用了原文作者的公开仓库，简单来说就是本地build了一个新的tag，然后修改一下deployment.yaml中对应的Image的tag为新版本的tag。

这里就是将image的v1换成了v2这个tag。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
        - image: guangzhengli/hellok8s:v2
          name: hellok8s-container
```

```
root@OPS-4152:/home/ek1ng# kubectl apply -f deployment.yaml 
deployment.apps/hellok8s-deployment configured

root@OPS-4152:/home/ek1ng# kubectl get pods
NAME                                   READY   STATUS              RESTARTS   AGE
hellok8s-deployment-594f97b78d-688b8   1/1     Running             0          11m
hellok8s-deployment-594f97b78d-8zlmx   1/1     Running             0          11m
hellok8s-deployment-594f97b78d-ntb24   1/1     Running             0          17m
hellok8s-deployment-67bd64b56f-m9x9p   0/1     ContainerCreating   0          5s
nginx-pod                              1/1     Running             0          17h

root@OPS-4152:/home/ek1ng# kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
hellok8s-deployment-67bd64b56f-lx4cr   1/1     Running   0          11s
hellok8s-deployment-67bd64b56f-m9x9p   1/1     Running   0          19s
hellok8s-deployment-67bd64b56f-w76dt   1/1     Running   0          9s
nginx-pod                              1/1     Running   0          17h

root@OPS-4152:/home/ek1ng# kubectl port-forward hellok8s-deployment-67bd64b56f-m9x9p 3000:3000

root@OPS-4152:~# curl localhost:3000
[v2] Hello, Kubernetes!
```

可以看到完成了版本更新。

### 滚动更新

那么这里又有一个问题，Pod是怎么进行更新的，在更新的时候服务可用吗？

答案是会同时进行更新，所以会有短时间不可用，需要等待所有Pod都升级完成才能提供服务，那么如果我们不希望服务不可用，就需要滚动更新，在v2版本的Pod没有可用之前，不删除v1版本的Pod。

这里呢可以配置spec.strategy,type，可以配置成`RollingUpdate`（滚动更新）或`Recreate`（重新创建）。

对于配置成`RollingUpdate`，还有属性`maxSurge`（最大峰值，能创建超出期望的Pod数量上限），`maxUnavailable`（最大不可用，更新过程中不可用的Pod数上线）

那么我们来尝试一下滚动更新，先回滚到v1版本。

```
root@OPS-4152:/home/ek1ng# kubectl describe pod hellok8s-deployment-594f97b78d-zcwvq | grep image
  Normal  Pulled     84s   kubelet            Container image "guangzhengli/hellok8s:v1" already present on machine
```

确认是v1后尝试滚动更新。

![rollingupdate](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6775616e677a68656e676c692f50696355524c406d61737465722f755069632f726f6c6c696e677570646174652e706e67)

### 存活探针（livenessProb）

存活探针用来做服务存活的检测，自动重启。

这里也是直接用了`guangzhengli/hellok8s:liveness`这个镜像，代码如下。

```
package main

import (
	"fmt"
	"io"
	"net/http"
	"time"
)

func hello(w http.ResponseWriter, r *http.Request) {
	io.WriteString(w, "[v2] Hello, Kubernetes!")
}

func main() {
	started := time.Now()
	http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		duration := time.Since(started)
		if duration.Seconds() > 15 {
			w.WriteHeader(500)
			w.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
		} else {
			w.WriteHeader(200)
			w.Write([]byte("ok"))
		}
	})

	http.HandleFunc("/", hello)
	http.ListenAndServe(":3000", nil)
}
```

`/healthz` 接口会在启动成功的 15s 内正常返回 200 状态码，在 15s 后，会一直返回 500 的状态码。

修改deployment，使用存活探针`livenessProbe`，用get请求来判断存活，设定每3秒探测一次，执行第一次探测前等待三秒。如果请求发现500会认为容器不是健康的，会重启容器。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s-deployment
spec:
  strategy:
     rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  replicas: 3
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
        - image: guangzhengli/hellok8s:liveness
          name: hellok8s-container
          livenessProbe:
            httpGet:
              path: /healthz
              port: 3000
            initialDelaySeconds: 3
            periodSeconds: 3
```

会发现能够自动重启。

```
root@OPS-4152:/home/ek1ng# kubectl apply -f deployment.yaml 
deployment.apps/hellok8s-deployment configured

root@OPS-4152:/home/ek1ng# kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
hellok8s-deployment-6d66979d6-fbjrs   1/1     Running   0          15s
hellok8s-deployment-6d66979d6-jcxng   1/1     Running   0          21s
hellok8s-deployment-6d66979d6-jg9kl   1/1     Running   0          22s
nginx-pod                             1/1     Running   0          21h

root@OPS-4152:/home/ek1ng# kubectl get pods
NAME                                  READY   STATUS    RESTARTS     AGE
hellok8s-deployment-6d66979d6-fbjrs   1/1     Running   1 (2s ago)   26s
hellok8s-deployment-6d66979d6-jcxng   1/1     Running   1 (5s ago)   32s
hellok8s-deployment-6d66979d6-jg9kl   1/1     Running   1 (5s ago)   33s
nginx-pod                             1/1     Running   0            21h
```

### 就绪探针（readinessProb）

就绪探针用来检测容器是否做好准备，来做负载均衡。

先把版本回滚到v2。

```
kubectl rollout undo deployment hellok8s-deployment --to-revision=2
```

使用镜像`guangzhengli/hellok8s:bad`，这个镜像提供的Web服务接口`/healthz`是会直接返回500的。

- `initialDelaySeconds`：容器启动后要等待多少秒后才启动存活和就绪探测器， 默认是 0 秒，最小值是 0。
- `periodSeconds`：执行探测的时间间隔（单位是秒）。默认是 10 秒。最小值是 1。
- `timeoutSeconds`：探测的超时后等待多少秒。默认值是 1 秒。最小值是 1。
- `successThreshold`：探测器在失败后，被视为成功的最小连续成功数。默认值是 1。 存活和启动探测的这个值必须是 1。最小值是 1。
- `failureThreshold`：当探测失败时，Kubernetes 的重试次数。 对存活探测而言，放弃就意味着重新启动容器。 对就绪探测而言，放弃意味着 Pod 会被打上未就绪的标签。默认值是 3。最小值是 1。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s-deployment
spec:
  strategy:
     rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  replicas: 3
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
        - image: guangzhengli/hellok8s:bad
          name: hellok8s-container
          readinessProbe:
            httpGet:
              path: /healthz
              port: 3000
            initialDelaySeconds: 1
            successThreshold: 5
```

有两个Pod一直处于没有Ready的状态，因此设置了最小不可用服务数量`maxUnavailable=1`，超出的最大可用pod数量`maxSurge=1`，因此Ready的Pod数量有3-1个，总共有3+1个Pod。

```
root@OPS-4152:/home/ek1ng# kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
hellok8s-deployment-595dff4d4d-pnsdj   0/1     Running   0          13s
hellok8s-deployment-595dff4d4d-tg2lq   0/1     Running   0          13s
hellok8s-deployment-67bd64b56f-jdgc6   1/1     Running   0          7m28s
hellok8s-deployment-67bd64b56f-t2c2c   1/1     Running   0          7m30s
```

## Service

K8s提供了一种叫做Service的资源，用以实现网络**负载均衡**以及 Pod 的高可用性。

Service的三种类型：

1. ClusterIP：Service **默认类型**，为 Service 分配 Cluster IP 作为虚拟 IP 地址，用于在集群内部访问 Service。
2. NodePort：将 Service 暴露在每个 Node 上的某个端口上，可以通过 访问 `NodeIP:NodePort` 访问 Service。NodePort 类型同时也会创建 Cluster IP，允许从集群内部访问 Service。
3. LoadBalancer：通过云服务提供商或硬件负载均衡器（LB）自动为 Service 分配一个外部的负载均衡器，同时也会自动创建 NodePort 和 Cluster IP。

### ClusterIP

![image-20230426172110742](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230426172110742.png)

CLusterIP是默认类型，尝试一下搭建一个图中的Service。

使用镜像`guangzhengli/hellok8s:v3`，对应代码如下

```
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
)

func hello(w http.ResponseWriter, r *http.Request) {
	host, _ := os.Hostname()
	io.WriteString(w, fmt.Sprintf("[v3] Hello, Kubernetes!, From host: %s", host))
}

func main() {
	http.HandleFunc("/", hello)
	http.ListenAndServe(":3000", nil)
}
```

更新v3版本的deployment。

接下来我们来定义一个Service，将端口映射出来。让集群中运行的其他应用程序访问我们的Pod时，使用这种类型的Service。

```
# service-hellok8s-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: service-hellok8s-clusterip
spec:
  type: ClusterIP
  selector:
    app: hellok8s
  ports:
  - port: 3000
    targetPort: 3000
```

```
root@OPS-4152:/home/ek1ng# kubectl apply -f service-hellok8s-clusterip.yaml
service/service-hellok8s-clusterip created

root@OPS-4152:/home/ek1ng# kubectl get endpoints
NAME                         ENDPOINTS                                                    AGE
kubernetes                   192.168.49.2:8443                                            64d
service-hellok8s-clusterip   10.244.120.100:3000,10.244.120.101:3000,10.244.120.99:3000   5s

root@OPS-4152:/home/ek1ng# kubectl get pod -o wide
NAME                                  READY   STATUS    RESTARTS   AGE   IP               NODE       NOMINATED NODE   READINESS GATES
hellok8s-deployment-d5ff4cf8b-74ntx   1/1     Running   0          15m   10.244.120.99    minikube   <none>           <none>
hellok8s-deployment-d5ff4cf8b-kxsll   1/1     Running   0          15m   10.244.120.100   minikube   <none>           <none>
hellok8s-deployment-d5ff4cf8b-vrh5s   1/1     Running   0          14m   10.244.120.101   minikube   <none>           <none>
nginx-pod                             1/1     Running   0          22h   10.244.120.77    minikube   <none>           <none>

root@OPS-4152:/home/ek1ng# kubectl get service
NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes                   ClusterIP   10.96.0.1       <none>        443/TCP    64d
service-hellok8s-clusterip   ClusterIP   10.110.81.182   <none>        3000/TCP   45s

root@OPS-4152:/home/ek1ng# kubectl exec -it nginx-pod /bin/bash

root@nginx-pod:/# curl 10.110.81.182:3000
[v3] Hello, Kubernetes!, From host: hellok8s-deployment-d5ff4cf8b-kxsll

root@nginx-pod:/# curl 10.110.81.182:3000
[v3] Hello, Kubernetes!, From host: hellok8s-deployment-d5ff4cf8b-vrh5s
```

上面的Service将流量负责转发向lable为hellok8s的Pod。

我们用nginx-pod这个pod,去访问用我们起的这个Service，会发现多次请求，返回的`hostname`并不一样，这说明Service接收请求后，转发给了不同的Pod，并且可以自动做负载均衡。

### NodePort

一个K8s集群往有多个Node,可以通过Node的ip和NodePort暴露服务。如果现在集群有两台Node都运行`hellok8s:v3` ，我们创建一个`NodePort`类型的Service，将`hellok8s:v3`的3000端口映射到`Node`的30000端口，就可以通过`http://node-ip:30000`访问服务。

![image-20230426172337533](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230426172337533.png)

当然，图中说的模型是有多个Node的，不过minikube只能模拟一个Node。所以我们这里也就只用了一个Node，内部有多个Pod。

来亲手尝试NodePort类型的Service。

```
# service-hellok8s-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: service-hellok8s-nodeport
spec:
  type: NodePort
  selector:
    app: hellok8s
  ports:
  - port: 3000
    nodePort: 30000
```

```
root@OPS-4152:/home/ek1ng# kubectl apply -f service-hellok8s-nodeport.yaml
service/service-hellok8s-nodeport created

root@OPS-4152:/home/ek1ng# kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
hellok8s-deployment-d5ff4cf8b-74ntx   1/1     Running   0          50m
hellok8s-deployment-d5ff4cf8b-kxsll   1/1     Running   0          50m
hellok8s-deployment-d5ff4cf8b-vrh5s   1/1     Running   0          50m
nginx-pod                             1/1     Running   0          23h

root@OPS-4152:/home/ek1ng# minikube ip
192.168.49.2

root@OPS-4152:/home/ek1ng# curl http://192.168.49.2:30000
[v3] Hello, Kubernetes!, From host: hellok8s-deployment-d5ff4cf8b-kxsll

root@OPS-4152:/home/ek1ng# curl http://192.168.49.2:30000
[v3] Hello, Kubernetes!, From host: hellok8s-deployment-d5ff4cf8b-vrh5s
```

从集群外请求，可以做负载均衡，收到来自不同Pod的响应。

### LoadBalancer

LoadBalancer是使用云提供商的负载均衡器向外暴露服务。

![service-loadbalancer-fix-name](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6775616e677a68656e676c692f50696355524c406d61737465722f755069632f736572766963652d6c6f616462616c616e6365722d6669782d6e616d652e706e67)

## Ingress

ingress可以简单理解为服务的网关Gateway，是所有流量的入口，经过配置的路由规则，将流量重定向到后端的服务。

![ingress](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6775616e677a68656e676c692f50696355524c406d61737465722f755069632f696e67726573732e706e67)

## Namespace

在开发中，需要不同环境来做开发和测试，例如`dev`,`test`,`prod`，为了给不同环境区分资源，让不同环境的资源独立，互相不影响，K8s提供了Namespace来帮助隔离资源。

Namespace将同一集群中的资源划分为相互隔离的组。同一Namespace中资源名称唯一。Namespace的作用域仅限带有Namespace的对象，比如说Deployment，Service。

之前使用的都是默认的namespace，为`default`。

```
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  
---

apiVersion: v1
kind: Namespace
metadata:
  name: test
```

定义了`dev`和`test`两个namespace。

```
root@OPS-4152:/home/ek1ng# kubectl apply -f namespace.yaml    
namespace/dev created
namespace/test created

root@OPS-4152:/home/ek1ng# kubectl get namespaces
NAME              STATUS   AGE
default           Active   65d
dev               Active   12s
kube-node-lease   Active   65d
kube-public       Active   65d
kube-system       Active   65d
role              Active   6d2h
test              Active   12s
```

如果说要在某个namespace创建资源，只需要跟上`-n <namespace>`

```
kubectl apply -f deployment.yaml -n dev

kubectl get pods -n dev
```

## Configmap

当使用namespace区别资源时，在代码中就不能写一样的数据库地址，因此可以用Configmap来将配置数据和应用程序的代码区分开。

下面对之前的代码进行修改，从环境变量中获取`DB_URL`来作为数据库地址。这里使用的镜像名是`hellok8s:v4`，代码如下

```
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
)

func hello(w http.ResponseWriter, r *http.Request) {
	host, _ := os.Hostname()
	dbURL := os.Getenv("DB_URL")
	io.WriteString(w, fmt.Sprintf("[v4] Hello, Kubernetes! From host: %s, Get Database Connect URL: %s", host, dbURL))
}

func main() {
	http.HandleFunc("/", hello)
	http.ListenAndServe(":3000", nil)
}
```

接下来就是给两个namespace创建configmap

```
# hellok8s-config-dev.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hellok8s-config
data:
  DB_URL: "http://DB_ADDRESS_DEV"
```

```
# hellok8s-config-test.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hellok8s-config
data:
  DB_URL: "http://DB_ADDRESS_TEST"
```

```
root@OPS-4152:/home/ek1ng# kubectl apply -f hellok8s-config-dev.yaml -n dev
configmap/hellok8s-config created
root@OPS-4152:/home/ek1ng# kubectl apply -f hellok8s-config-test.yaml -n test 
configmap/hellok8s-config created
```

接着给两个namespace分别创建相同的Pod

```
# hellok8s.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hellok8s-pod
spec:
  containers:
    - name: hellok8s-container
      image: guangzhengli/hellok8s:v4
      env:
        - name: DB_URL
          valueFrom:
            configMapKeyRef:
              name: hellok8s-config
              key: DB_URL
```

```
root@OPS-4152:/home/ek1ng# kubectl apply -f hellok8s.yaml -n dev    
pod/hellok8s-pod created

root@OPS-4152:/home/ek1ng# kubectl apply -f hellok8s.yaml -n test
pod/hellok8s-pod created

root@OPS-4152:/home/ek1ng# kubectl port-forward hellok8s-pod 3000:3000 -n dev

root@OPS-4152:~# curl http://localhost:3000
[v4] Hello, Kubernetes! From host: hellok8s-pod, Get Database Connect URL: http://DB_ADDRESS_DEV

root@OPS-4152:/home/ek1ng# kubectl port-forward hellok8s-pod 3000:3000 -n test

root@OPS-4152:~# curl http://localhost:3000
[v4] Hello, Kubernetes! From host: hellok8s-pod, Get Database Connect URL: http://DB_ADDRESS_TEST
```

## Secret

当配置信息需要存一些加密信息时，configmap就无法满足，例如对于数据库密码，总不能明文存在configmap里，因此就可以用Secret来存储。用起来和configmap类似，这里就不尝试了。

## Job

如果需要执行一次性任务，就需要用到Job这类资源。

> Job 会创建一个或者多个 Pod，并将继续重试 Pod 的执行，直到指定数量的 Pod 成功终止。 随着 Pod 成功结束，Job 跟踪记录成功完成的 Pod 个数。 当数量达到指定的成功个数阈值时，任务（即 Job）结束。 删除 Job 的操作会清除所创建的全部 Pod。 挂起 Job 的操作会删除 Job 的所有活跃 Pod，直到 Job 被再次恢复执行。
>
> 一种简单的使用场景下，你会创建一个 Job 对象以便以一种可靠的方式运行某 Pod 直到完成。 当第一个 Pod 失败或者被删除（比如因为节点硬件失效或者重启）时，Job 对象会启动一个新的 Pod。

尝试创建一个Job

```
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  parallelism: 3
  completions: 5
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: echo
          image: busybox
          command:
            - "/bin/sh"
          args:
            - "-c"
            - "for i in 9 8 7 6 5 4 3 2 1 ; do echo $i ; done"
```

```
root@OPS-4152:/home/ek1ng# kubectl apply -f hello-job.yaml
job.batch/hello-job created
root@OPS-4152:/home/ek1ng# kubectl get jobs  
NAME        COMPLETIONS   DURATION   AGE
hello-job   0/5           3s         3s
root@OPS-4152:/home/ek1ng# kubectl get pods    
NAME                                  READY   STATUS              RESTARTS   AGE
hello-job-98x9t                       0/1     ContainerCreating   0          6s
hello-job-jhx77                       0/1     ContainerCreating   0          6s
hello-job-n5ftg                       0/1     ContainerCreating   0          6s
hellok8s-deployment-d5ff4cf8b-74ntx   1/1     Running             0          93m
hellok8s-deployment-d5ff4cf8b-kxsll   1/1     Running             0          93m
hellok8s-deployment-d5ff4cf8b-vrh5s   1/1     Running             0          93m
nginx-pod                             1/1     Running             0          23h
root@OPS-4152:/home/ek1ng# kubectl logs -f hello-job-98x9t
9
8
7
6
5
4
3
2
1

```

## CronJob

CronJob 可以理解为定时任务，创建基于 Cron 时间调度的 [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)。

> CronJob 用于执行周期性的动作，例如备份、报告生成等。 这些任务中的每一个都应该配置为周期性重复的（例如：每天/每周/每月一次）； 你可以定义任务开始执行的时间间隔。

类似Job，不做实践了。

## Helm

那么当然，前面零散的创建了很多yaml，分别创建了各自各样的资源，而如果我们向完整部署一套上面的服务，一个一个的kubectl apply过去当然不太现实，所以就可以用到`Helm`。

Helm是一个帮助管理K8s应用的应用。我们可以自己打包一个heml chart，来进行一整套K8s服务的部署。

## Dashboard

dashboard指基于Web的用户节目，可以选择Kubuneters dashboard，K9s等等