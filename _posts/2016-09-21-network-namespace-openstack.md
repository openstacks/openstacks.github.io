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

#### Linux系统的全局资源        
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

 > ![](/assets/ns1.jpg)

每个Namespace看上去就像一个单独的Linux系统,从而实现了系统的隔离(Isolating Your System with Linux Namespaces)。与hypervisor比较，这是一个轻量级的系统虚拟化解决方案。被openstack广泛使用，并且是docker的核心技术之一。

 > ![](/assets/ns2.png)

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

 > ![](/assets/ns3.png)
经过上面几步，创建出如下的网络拓扑图：

 > ![](/assets/ns4.png)

#### 创建veth pair    
   创建一个veth pair, 一端接在在新建的namespace中, 通常命名为eth0，一端接在openvswitch, 通常命名为veth。通过openvswitch进行路由转发, 达到两个namespace通信的目的。
  

 > ![](/assets/ns5.png)
 > ![](/assets/ns6.png) 
 > ![](/assets/ns7.png) 

对green namespace做同样的操作，把他和openvswitch也连起来。


 > ![](/assets/ns8.png)

 > ![](/assets/ns9.png)

#### 配置IP地址      
给eth-r和eth-g配置IP地址后，两个namespace就可以互相交流了。 

 > ![](/assets/ns10.png)

 > ![](/assets/ns11.png)

 > ![](/assets/ns12.png)

### DHCP在Openstack中的实现
虚拟机，DHCP服务和Linux bridge在Openstack中的逻辑结构图如下图
所示。本文以linux bridge为例来解释说明。    


 > ![](/assets/ns13.png)

虚拟机和dhcp namespace都连在linux bridge上,dnsmasq服务在dhcp 
namespace的veth pair端口监听虚拟机dhcp的请求。具体实现步骤如下：

 - 新建一个namespace:dhcp-r
 - 新建一个veth-pair(tab-1,ns-1)
 - 把dhcp的Ip地址配置在ns-1端口上
 - 起dnsmasq服务,让他监听在ns-1上


 > ![](/assets/ns14.png)

 > ![](/assets/ns15.png)

 > ![](/assets/ns16.png)