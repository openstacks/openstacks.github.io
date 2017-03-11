---
layout: post
title:  "部署一个Ceph集群，并用他作为Openstack的后端存储"
categories: Ceph
tags:  Ceph cluster centos Storage
---

* content
{:toc}

本文介绍如何一步步在centos上部署ceph分布式存储集群。然后用他做为Openstack的后端存储。 

### 部署环境

#### 硬件环境
包括五台VM，其中三台vm用了部署ceph集群；一台用来部署Openstack Ocata AIO，另外一台部署一个Openstack的测试环境。

<table>
    <tr>
        <td>主机</td>
        <td>IP</td>
        <td>功能</td>
    </tr>
    <tr>
        <td>ceph</td>
        <td>192.168.1.101/104/105</td>
        <td>deploy,mon*3,osd*9</td>
    </tr>
    <tr>
        <td>openstack</td>
        <td>192.168.1.102</td>
        <td>Openstack ocata all-in-one</td>
    </tr>
    <tr>
        <td>test</td>
        <td>192.168.1.103</td>
        <td>Openstack Test environment:Rally& Shark</td>
    </tr>
</table>

#### 软件环境
- 操作系统：Centos 7.3    
- Openstack：Ocata    
- Ceph准备：Jewel



####  安装Ceph  

##### 准备Ceph安装环境

###### 准备repo
    
  - 使用aliyun镜像，来加速安装过程。
     
```   
yum clean all
rm -rf /etc/yum.repos.d/*.repo
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
sed -i '/aliyuncs/d' /etc/yum.repos.d/CentOS-Base.repo
sed -i '/aliyuncs/d' /etc/yum.repos.d/epel.repo
sed -i 's/$releasever/7/g' /etc/yum.repos.d/CentOS-Base.repo
   
```

- 准备ceph Jewel的源

```
#vi /etc/yum.repos.d/ceph.repo

[ceph]
name=ceph
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/x86_64/
gpgcheck=0
[ceph-noarch]
name=cephnoarch
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/noarch/
gpgcheck=0

```
```
yum update -y
```

###### 启用Ceph monitor OSD端口

下面命令分别在三个ceph 节点上执行。

```
firewall-cmd --zone=public --add-port=6789/tcp --permanent
firewall-cmd --zone=public --add-port=6800-7100/tcp --permanent
firewall-cmd --reload
firewall-cmd --zone=public --list-all

```

###### 禁用Selinux
下面命令分别在三个ceph 节点上执行。
```
setenforce 0
```

###### 安装ntp
下面命令分别在三个ceph 节点上执行。
```
yum install ntp ntpdate -y
systemctl restart ntpdate.service ntpd.service
systemctl enable ntpd.service ntpdate.service

```
#### 在Ceph-node1上创建Ceph集群

- 安装ceph-deploy
```
yum install ceph-deploy -y
```
- 用Ceph-deploy创建Ceph集群 
```
mkdir /etc/ceph
cd /etc/ceph
ceph-deploy new ceph-node1
```
- 安装ceph二进制软件包
```
ceph-deploy install ceph-node1 ceph-node2 ceph-node3
ceph -v
```
- 在ceph-node1上创建第一个ceph monitor
```
ceph-deploy mon create-initial
ceph -s
```
- 在ceph-node1上创建OSD
```
ceph-deploy disk list ceph-node1(列出disk）
ceph-deploy disk zap ceph-node1:sdb ceph-node1:sdc ceph-node1:sdd
ceph-deploy osd create ceph-node1:sdb ceph-node1:sdc ceph-node1:sdd
ceph -s
```
