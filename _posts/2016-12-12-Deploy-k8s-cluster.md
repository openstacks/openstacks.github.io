---
layout: post
title:  "如何构建本地k8s 集群"
categories: Kubernet
tags:  kubernet docker Paas vagrant 
---

* content
{:toc}

本文介绍如何使用Vagrant和VirtualBox部署运行Kubernetes集群，进行测试/开发的简单方法。   
参考资料可以参考<https://www.kubernetes.org.cn/doc-6/>







### 预备知识

Vagrant和VirtualBox安装好。 

 
 
### 安装k8s集群 

 运行下面代码创建k8s集群：
 
 
```
＃export KUBERNETES_PROVIDER=vagrant
＃curl -sS https://get.k8s.io | bash  
```
 
 环境变量KUBERNETES_PROVIDER用来告诉所有不同的集群管理脚本该使用哪一个脚本管理器（比如Vagrant）。        
 默认情况下，Vagrant会创建一个单独的master VM（被称为kubernetes-master），以及一个节点VM（被称为kubernetes-minion-1）。     
 每个VM会占用1G内存，所以确保你有至少2G-4G的空余内存（以及合适的空闲硬盘空间）。

Vagrant会提供集群中每台机器运行Kuberbetes所有必须的组件。每台机器会花费几分钟完成初始化设置。
 
 
###  访问K8s集群

 
```
＃vagrant ssh master
＃vagrant ssh minion-1
```
 
 
###  查看kubernetes-master上的服务状态或者日志   
查看kubernetes-master上的服务状态或者日志


```
[vagrant@kubernetes-master ~] $ vagrant ssh master
[vagrant@kubernetes-master ~] $ sudo su
[root@kubernetes-master ~] $ systemctl status kubelet
[root@kubernetes-master ~] $ journalctl -ru kubelet
[root@kubernetes-master ~] $ systemctl status docker
[root@kubernetes-master ~] $ journalctl -ru docker
[root@kubernetes-master ~] $ tail -f /var/log/kube-apiserver.log
[root@kubernetes-master ~] $ tail -f /var/log/kube-controller-manager.log
[root@kubernetes-master ~] $ tail -f /var/log/kube-scheduler.log
```

####  查看任意节点的服务状态



```
[vagrant@kubernetes-master ~] $ vagrant ssh minion-1
[vagrant@kubernetes-master ~] $ sudo su
[root@kubernetes-master ~] $ systemctl status kubelet
[root@kubernetes-master ~] $ journalctl -ru kubelet
[root@kubernetes-master ~] $ systemctl status docker
[root@kubernetes-master ~] $ journalctl -ru docker
```



####  运行容器
现在开始运行一些容器吧！

现在你可以用cluster/kube-*.sh的任何命令来与你的VMs交互。启动容器之前并没有 pods 、 service 和 replication controller ：

```
$ ./cluster/kubectl.sh get pods
NAME READY STATUS RESTARTS AGE

$ ./cluster/kubectl.sh get services
NAME LABELS SELECTOR IP(S) PORT(S)

$ ./cluster/kubectl.sh get replicationcontrollers
CONTROLLER CONTAINER(S) IMAGE(S) SELECTOR REPLICAS
```

启动一个运行nginx、使用复制控制器并且副本数为3的容器：

```
$ ./cluster/kubectl.sh run my-nginx --image=nginx --replicas=3 --port=80
```
（此时）列出pods，会看到3个容器已经被启动了，并且处于等待状态：

```
$ ./cluster/kubectl.sh get pods
NAME READY STATUS RESTARTS AGE
my-nginx-5kq0g 0/1 Pending 0 10s
my-nginx-gr3hh 0/1 Pending 0 10s
my-nginx-xql4j 0/1 Pending 0 10s
```

你需要等待（Vagrant）分配好资源，这时可以通过命令来监测节点


```
$ vagrant ssh minion-1 -c 'sudo docker images'
```

下载好docker nginx镜像后容器就会启动，通过下面的命令查看：

```
$ vagrant ssh minion-1 -c 'sudo docker ps'
```

这时再次列举pods、服务和复制控制器，会看到：

```
$ ./cluster/kubectl.sh get pods
NAME READY STATUS RESTARTS AGE
my-nginx-5kq0g 1/1 Running 0 1m
my-nginx-gr3hh 1/1 Running 0 1m
my-nginx-xql4j 1/1 Running 0 1m

$ ./cluster/kubectl.sh get services
NAME LABELS SELECTOR IP(S) PORT(S)

$ ./cluster/kubectl.sh get replicationcontrollers
CONTROLLER CONTAINER(S) IMAGE(S) SELECTOR REPLICAS
my-nginx my-nginx nginx run=my-nginx 3
```


