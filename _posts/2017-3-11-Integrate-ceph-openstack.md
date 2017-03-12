---
layout: post
title:  "Step by Step 部署Ceph集群，集成Openstack"
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
- Ceph：Jewel



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

###### 操作系统配置
- 启用Ceph monitor OSD端口    
```
firewall-cmd --zone=public --add-port=6789/tcp --permanent
firewall-cmd --zone=public --add-port=6800-7100/tcp --permanent
firewall-cmd --reload
firewall-cmd --zone=public --list-all
```

- 禁用Selinux    
```
setenforce 0
```

- 安装ntp   
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
#### 总结
通过上面的步骤，一个all in one的ceph就成功部署了。

- 检查ceph的状态。
```
ceph -s
```
#### 集成ceph与Openstack Nova   
- 安装ceph客户端    
集成ceph与Openstack的第一步就是要在openstack的节点上安装ceph客户端（一些ceph命令行工具和连接ceph集群需要的libraries)。 
```
$ ceph-deploy install --cli openstack
$ ceph-deploy config push openstack
```
- 创建pool    
给虚拟机的ephemeral disks创建一个ceph pool。   
```
 $ ceph osd pool create compute 128
 pool 'compute' created
```
 - 给nova创建一个ceph用户     
 给nova创建一个ceph用户，并赋予合适的权限。 
 ```
 [root@ceph ceph]# ceph auth get-or-create client.compute mon "allow r" osd "allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=compute, allow rx pool=images"
[client.compute]
key = AQBLHcJYm1XxBBAA75foQeQ72bT3GsGVDzBZcg==
 ```
 - 为用户添加秘钥，并修改秘钥文件的group和权限     
 客户端需要ceph秘钥去访问集群，Ceph 创建了一个默认用户client.admin,他有足够的权限去访问ceph集群。不能把这个用户共享给其他客户端。更好的做法是用分开的秘钥创建一个新的ceph用户去访问特定的pool。  
 ```
[root@ceph ceph]# ceph auth get-key client.compute | ssh openstack tee /etc/ceph/ceph.client.compute.keyring
AQBLHcJYm1XxBBAA75foQeQ72bT3GsGVDzBZcg==
[root@openstack]# chgrp nova /etc/ceph/ceph.client.compute.keyring
[root@openstack]# chmod 0640 /etc/ceph/ceph.client.compute.keyring
 ```
 
 - 配置openstack节点的ceph.conf文件     
 把keyring加到ceph.conf文件。 
 ```
 vi /etc/ceph/ceph.conf
 [client.compute]
 keyring = /etc/ceph/ceph.client.compute.keyring
 ```
 - 集成ceph和libvirt   
 libvirt进程需要有访问ceph集群的权限。需要生成一个uuid，然后创建，定义和设置秘钥给libvirt。：  
 
> 生成一个uuid
 ```
 [root@openstack]# uuidgen
  c1261b3e-eb93-49bc-aa13-557df63a6347
 ```


 > 创建秘钥文件，并将uuid设置给他     
  ```
<secret ephemeral="no" private="no">
<uuid>c1261b3e-eb93-49bc-aa13-557df63a6347</uuid>
<usage type="ceph">
<name>client.compute secret</name>
</usage>
</secret>
 ```
 
 定义秘钥文件，生成保密字符串
 ```
[root@openstack]# virsh secret-define --file ceph.xml
Secret c1261b3e-eb93-49bc-aa13-557df63a6347 created
 ```

 在virsh里设置好上一步生成的保密字符串     
```
[root@openstack]# virsh secret-set-value --secret c1261b3e-eb93-49bc-aa13-557df63a6347  --base64 $(cat client.compute.key)
Secret value set
[root@openstack]# virsh secret-list
setlocale: No such file or directory
 UUID                                  Usage
--------------------------------------------------------------------------------
 c1261b3e-eb93-49bc-aa13-557df63a6347  ceph client.compute secret
 ```  

 
- 配置nova        
 
 修改/etc/nova/nova.conf文件里的libvirt部分，增加ceph的连接信息。     
 ```
 [libvirt]
images_rbd_pool=compute
images_type=rbd
rbd_secret_uuid=c1261b3e-eb93-49bc-aa13-557df63a6347
rbd_user=compute
 ```
 
- 重启nova compute服务      
  ```
  [root@openstack]#systemctl restart openstack-nova-compute
  ```
 
- 测试
 
 新建一个vm，然后检查VM’s ephemeral disk是否健在ceph上。      
```
[root@ceph ceph]# rbd -p compute ls
24e6ca7f-05c8-411b-b23d-6e5ee1c809f9_disk

[root@ceph ceph]# rbd -p compute info 24e6ca7f-05c8-411b-b23d-6e5ee1c809f9_disk
rbd image '24e6ca7f-05c8-411b-b23d-6e5ee1c809f9_disk':
size 1024 MB in 256 objects
order 22 (4096 kB objects)
block_name_prefix: rbd_data.fb042ae8944a
format: 2
features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
flags:
```

#### 参考文档 

- <http://xuxiaopang.com/2016/10/09/ceph-quick-install-el7-jewel/>         
- <http://www.stratoscale.com/blog/storage/integrating-ceph-storage-openstack-step-step-guide/> 
- ceph cookbook
 
