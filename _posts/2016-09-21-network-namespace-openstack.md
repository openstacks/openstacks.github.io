---
layout: post
title:  "Network Namespace在Openstack中的应用"
categories: Network
tags:  network namespace openstack neutron dhcp
---

* content
{:toc}

本文以DHCP为例，介绍了network namespace的基本原理，
以及他在Openstack中的应用。







### 基本概念
####  Linux系统的全局资源        
 - user: 用户ID和 组ID
 - uts: 主机名和域名
 - pid: 进程ID
 - mount: 文件系统挂载点
 - network: 网路资源
 - ipc: 进程间通信
 
#### Linux Namespace    

Linux Namespaces提供了一种隔离系统全局资源的方法,
 通过这个方法，每个namespace都了有一份独立的资源。
 这样,不同的进程在各自的namespace里对同一种资源的访问不会发生冲突。
![](http://mmbiz.qpic.cn/mmbiz/Bh66jm0ozvbqdgIhf5pYL1ke5AZL6JH2wNaF73HSDtWETQ0XVjExMPKbJyicuYq9uWHrNh6KWbxrJl940Ho1rKQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1)

每个Namespace看上去就像一个单独的Linux系统,从而实现了系统的隔离(Isolating Your System with Linux Namespaces)。与hypervisor比较，这是一个轻量级的系统虚拟化解决方案。被openstack广泛使用，并且是docker的核心技术之一。

![](http://mmbiz.qpic.cn/mmbiz/Bh66jm0ozvbqdgIhf5pYL1ke5AZL6JH24vc1rS1icGF6p1qdCic1T3QDIahAsPtxYBRKg9jj8pKtqwHg1I4zP8ag/640?wx_fmt=png&wxfrom=5&wx_lazy=1)

#### Linux Network Namespace 

   Network namespace主要实现了网络资源的隔离，网络资源包括网络
   设备、IPv4和IPv6协议栈、IP路由表、防火墙、socket等。给一个或多个进程私有的网络资源。在openstack里，用来实现L3层网路的虚拟化。


####  DHCP在Openstack中的实现

   DHCP的基本功能就是给客户端动态提供IP，具体原理不在这里描述，下面只是简单地介绍一下DHCP在Openstack里的如何工作的。

   neutron-dhcp-agent 用参数"--dhcp-hostsfile=filename" 启动dnsmasq服务。当新的port被创建或者旧的port被删除时，Openstack以dnsmasq-format的格式更新hostsfile文件。如下例所示：          
     fa:17:4e:86:18:a6,,192.168.10.10,192800    
     fa:17:4e:78:18:9b,,192.168.10.20,192800    
     
   当Openstack从hostsfile增加或者删除一条记录时，他给dnsmasq服务发送一个HUP信号，dnsmasq会重新读取配置文件和hostsfile。当VM发出DHCP-Discover后,dnsmasq分配IP地址给VM。

### 管理Network Namespace

#### 新建network namespace  
![](http://mmbiz.qpic.cn/mmbiz/Bh66jm0ozvbqdgIhf5pYL1ke5AZL6JH2I2HHzxNTsbriapBzuia3IhLhKVwGNhapPwQrDZl6Yiaia1xLVFUD1j0c0A/640?wx_fmt=png&wxfrom=5&wx_lazy=1)

经过上面几步，创建出如下的网络拓扑图：
![](http://mmbiz.qpic.cn/mmbiz/Bh66jm0ozvbqdgIhf5pYL1ke5AZL6JH2Ru87VnwXQKEj6jXBnKkNibqP4JAPbToFahFrnmKaVjsT08iaaTMicJt2w/640?wx_fmt=png&wxfrom=5&wx_lazy=1)


#### 创建veth pair    
   创建一个veth pair, 一端接在在新建的namespace中, 通常命名为eth0，一端接在openvswitch, 通常命名为veth。通过openvswitch进行路由转发, 达到两个namespace通信的目的。
   ![](http://mmsns.qpic.cn/mmsns/Bh66jm0ozvbqdgIhf5pYL1ke5AZL6JH2O43ib5TMBIX6soY6ru5AcvA/0?wx_lazy=1)
   
![](http://mmbiz.qpic.cn/mmbiz/Bh66jm0ozvbqdgIhf5pYL1ke5AZL6JH2v3o1XrF3uR4ibnnUjjUiaSdKVvIo5tnsC6L3NfCqXbWApRvGZXbIjKYw/640?wx_fmt=png&wxfrom=5&wx_lazy=1)


对green namespace做同样的操作，把他和openvswitch也连起来。
![](http://mmbiz.qpic.cn/mmbiz/Bh66jm0ozvbqdgIhf5pYL1ke5AZL6JH22apiaRSmzX5ibr5ic9BbYNhULMJHsz3XhHCicP4BbPicV3yTnna5Bq1EjRg/640?wx_fmt=png&wxfrom=5&wx_lazy=1)

![](http://mmbiz.qpic.cn/mmbiz/Bh66jm0ozvbqdgIhf5pYL1ke5AZL6JH2XGIibRuOGPAzcogygGuIhXaicBwp1RWrW4I9LuoL8LP4pmIRw2wLkWmQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1)
#### 配置IP地址      
给eth-r和eth-g配置IP地址后，两个namespace就可以互相交流了。 
![](http://mmbiz.qpic.cn/mmbiz/Bh66jm0ozvbqdgIhf5pYL1ke5AZL6JH2aIJEnLicCzYzRZQEBauSTCCZlaug7xAtdd2CKiaibLYKXS7JM87GBvNicA/640?wx_fmt=png&wxfrom=5&wx_lazy=1)

![](http://mmbiz.qpic.cn/mmbiz/Bh66jm0ozvbqdgIhf5pYL1ke5AZL6JH2d4zoib0CI5ad7zsK1AntPqic5pLxdun7NtwzM4ZfKXFamuIgk9eLNPJg/640?wx_fmt=png&wxfrom=5&wx_lazy=1)

![](http://mmbiz.qpic.cn/mmbiz/Bh66jm0ozvbqdgIhf5pYL1ke5AZL6JH2IwsaRLS1qMJEFLqLjkoibQrRAUDTH1fDrOG2l37fIyDDv3Scr3cwqmg/640?wx_fmt=png&wxfrom=5&wx_lazy=1)

### DHCP在Openstack中的实现
虚拟机，DHCP服务和Linux bridge在Openstack中的逻辑结构图如下图
所示。本文以linux bridge为例来解释说明。    
![](http://mmbiz.qpic.cn/mmbiz/Bh66jm0ozvbqdgIhf5pYL1ke5AZL6JH2FQy8r02GFFMtPJ11aGZTBINKsPdZYG0Tylk1arcTk9aTdS0PNaC4Gw/640?wx_fmt=png&wxfrom=5&wx_lazy=1)

虚拟机和dhcp namespace都连在linux bridge上,dnsmasq服务在dhcp 
namespace的veth pair端口监听虚拟机dhcp的请求。具体实现步骤如下：

 - 新建一个namespace:dhcp-r
 - 新建一个veth-pair(tab-1,ns-1)
 - 把dhcp的Ip地址配置在ns-1端口上
 - 起dnsmasq服务,让他监听在ns-1上

![](http://mmbiz.qpic.cn/mmbiz/Bh66jm0ozvbqdgIhf5pYL1ke5AZL6JH2vMSibrPXa67Vbgyk0wgoVoDUKce5CBckwdcWYnYctn52W0RRLZ1ibqXg/640?wx_fmt=png&wxfrom=5&wx_lazy=1)

![](http://mmbiz.qpic.cn/mmbiz/Bh66jm0ozvbqdgIhf5pYL1ke5AZL6JH22fiadgjicj8j0hLHEicw536d9iaLdeHkibZwuiaEnXY0L3YvCoB8SEuhkqSA/640?wx_fmt=png&wxfrom=5&wx_lazy=1)

![](http://mmbiz.qpic.cn/mmbiz/Bh66jm0ozvbqdgIhf5pYL1ke5AZL6JH2GedwANt1xDUS8dCicFBVAenP17mywl7U5ZiaPrpo2y3GRvRDq9dAVRvA/640?wx_fmt=png&wxfrom=5&wx_lazy=1)
