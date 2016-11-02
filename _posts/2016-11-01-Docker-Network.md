---
layout: post
title:  "Docker Multi Host Networking with Etcd and Vagrant"
categories: Docker 
tags:  Docker vxlan Etcd Vagrant
---

* content
{:toc}

本文通过实例来演示跨Host的Docker容器之间如何通过overlay(vxlan)进行通信。






###  拓扑图

下图是本次测试的拓扑图。总共三个VM，在一个VM上安装etcd，用来做key－value存储。另外两个     
VM安装两个docker容器，他们通过overlay network互相通信。
   
  > ![](/assets/multi-host.png) 


### 建立测试环境    
     
 本测试通过virtualBox和vagrant来生成三台VM，然后在VM上安装etcd和docker容器。所以首先必须安装      
 virtualbox和vagrant。   
 
- [如何安装virtualbox](https://www.virtualbox.org/wiki/Downloads)
    
- [如何安装vagrant](https://www.vagrantup.com/downloads.html)

   

 
### 创建VM
 
 通过vagrant创建三个VM。步骤如下：

#### Develop Vagrantfile

Vagrantfile的内容如下所示   

  {% highlight ruby %}
  
Vagrant.configure(2) do |config|

  config.vm.box = "ubuntu/wily64"
  config.vm.provider "virtualbox" do |vb|
  vb.memory = "2048"
  vb.cpus = 1
  end
  
  id_rsa_ssh_key = File.read(File.join(Dir.home,".ssh", "id_dsa"))
  id_rsa_ssh_key_pub = File.read(File.join(Dir.home,".ssh", "id_dsa.pub"))
  
  config.ssh.insert_key = false
  config.vm.provision :shell, path: "bootstrap.sh"
  config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"

  config.hostmanager.enabled = true
  config.hostmanager.manage_host = false
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true

  config.vm.define "vm1" do |vm1|
    vm1.vm.hostname = 'vm1-hostname'
    vm1.vm.network "private_network", ip: "172.28.128.5"
  end

  config.vm.define "vm2" do |vm2|
   vm2.vm.hostname = 'vm2-hostname'
   vm2.vm.network "private_network", ip: "172.28.128.6"
  end

  config.vm.define "vm3" do |vm3|
   vm3.vm.hostname = 'vm3-hostname'
   vm3.vm.network "private_network", ip: "172.28.128.7"
  end

end

  {% endhighlight %}

#### Develop bootstrap.sh 
bootstrap.sh文件里面定义的命令是用来在三个host上安装docker。

{% highlight shell %}

#!/usr/bin/env bash

sed -i '/127.0.1.1/d' /etc/hosts

sudo apt-get install apt-transport-https ca-certificates -y
sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D

sudo echo "deb https://apt.dockerproject.org/repo ubuntu-xenial main" >/etc/apt/sources.list.d/docker.list 
sudo apt-get update 

sudo apt-get install linux-image-extra-$(uname -r) linux-image-extra-virtual -y

sudo apt-get install aufs-tools -y 
sudo apt-get install cgroupfs-mount cgroup-lite -y 

sudo wget http://archive.ubuntu.com/ubuntu/pool/main/libt/libtool/libltdl7_2.4.6-0.1_amd64.deb
sudo dpkg -i libltdl7_2.4.6-0.1_amd64.deb 

sudo apt-get install docker-engine --force-yes -y

 {% endhighlight %}

#### 启动VM
执行下面的命令启动三个host。


`＃vagrant up`


 
#### 检测VM的状态 


`＃vagrant status`

 

#### SSH到VM 

`＃vagrant ssh vm1` 


### 启动 etcd 容器

#### 登录到vm1

`vagrant ssh vm1`

#### 启动 etcd 容器

```
root@vm1-hostname:/home/vagrant# export HostIP="172.28.128.5"

root@vm1-hostname:/home/vagrant# docker run -d -v /usr/share/ca-certificates/:/etc/ssl/certs -p 4001:4001 -p 2380:2380 -p 2379:2379 \
>  --name etcd quay.io/coreos/etcd \
>  /usr/local/bin/etcd \
>  -name etcd0 \
>  -advertise-client-urls http://${HostIP}:2379,http://${HostIP}:4001 \
>  -listen-client-urls http://0.0.0.0:2379,http://0.0.0.0:4001 \
>  -initial-advertise-peer-urls http://${HostIP}:2380 \
>  -listen-peer-urls http://0.0.0.0:2380 \
>  -initial-cluster-token etcd-cluster-1 \
>  -initial-cluster etcd0=http://${HostIP}:2380 \
>  -initial-cluster-state new

Unable to find image 'quay.io/coreos/etcd:latest' locally
latest: Pulling from coreos/etcd
3690ec4760f9: Pull complete 
92d817b4b80b: Pull complete 
6308cfa41eed: Pull complete 
df513d9e6e20: Pull complete 
bb13ac445088: Pull complete 
Digest: sha256:806ef957103453cbdfc02f621019a878027ff40d08a81e49229352954c59dd4e
Status: Downloaded newer image for quay.io/coreos/etcd:latest
c6938f183151360c891b21cdf2e84a10bc8ec840e3c5fd08133a713a86577e42

root@vm1-hostname:/home/vagrant# docker ps
CONTAINER ID        IMAGE                 COMMAND                  CREATED              STATUS              PORTS                                                      NAMES
c6938f183151        quay.io/coreos/etcd   "/usr/local/bin/etcd "   About a minute ago   Up About a minute   0.0.0.0:2379-2380->2379-2380/tcp, 0.0.0.0:4001->4001/tcp   etcd
```

#### 检查etcd的状态

```
root@vm1-hostname:/home/vagrant# docker exec -it c6938f183151 /usr/local/bin/etcdctl cluster-health
member 7eca6ad3f4f2297f is healthy: got healthy result from http://172.28.128.5:2379
cluster is healthy
```

#### 启动Docker Daemon


VM2和VM3上的Docker Engine daemon应该带着Etcd cluster parameters --cluster-store and --cluster-advertise启动,       
这样所有运行在不同vms上的Docker Engine可以相互通信交流。 
-  --cluster-store: VM1上Etcd service的IP和端口
-  --cluster-advertise: 本机的IP和Docker Daemon端口 


```
root@vm2-hostname# sudo /usr/bin/docker daemon -H tcp://0.0.0.0:2375 \
 -H unix:///var/run/docker.sock \
 --cluster-store=etcd://172.28.128.5:2379 \
 --cluster-advertise=172.28.128.6:2375 

root@vm2-hostname# sudo /usr/bin/docker daemon -H tcp://0.0.0.0:2375 \
 -H unix:///var/run/docker.sock \
 --cluster-store=etcd://172.28.128.5:2379 \
 --cluster-advertise=172.28.128.7:2375 
```

### 创建Docker overlay网络 

在vm2上运行下面的命令创建一个overlay网络.在vm2创建的overy network会通过KV store自动同步到vm3.


` root@vm2-hostname# docker network create --driver overlay --subnet=10.0.4.0/24 my-net`      

` root@vm2-hostname# docker network ls`

 
### 在VM2和VM3上运行container

` root@vm3-hostname# docker run -it --net=my-net ubuntu:trusty /bin/bash`  
      
` root@vm2-hostname# docker run -it --net=my-net ubuntu:trusty /bin/bash`
 
### 查看容器内部的网络

进入容器，它有两块网卡，其中eth1的网络是一个内部的网段，其实它走的还是普通的NAT模式；
而eth0是overlay网段上分配的IP地址，也就是它走的是overlay网络，它的MTU是1450而不是1500

```
root@8293faef3880:/# ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:0a:00:04:02  
          inet addr:10.0.4.2  Bcast:0.0.0.0  Mask:255.255.255.0
          inet6 addr: fe80::42:aff:fe00:402/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:15 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:1206 (1.2 KB)  TX bytes:648 (648.0 B)

eth1      Link encap:Ethernet  HWaddr 02:42:ac:12:00:02  
          inet addr:172.18.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe12:2/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:15 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:1206 (1.2 KB)  TX bytes:648 (648.0 B)
```

### 查看overlay network的信息

libnetwork为overlay network创建单独的net namespace。登录VM2或者VM3查看。

```
root@vm3-hostname:/# docker network ls|grep my-net
ec3ea5fa4f44        my-net              overlay             global              
root@vm3-hostname:/# ls /var/run/docker/netns
1-ec3ea5fa4f  5738d71324b3
```

其中，1-ec3ea5fa4f为overlay driver sandbox对应的net namespace。


```
root@vm3-hostname:/# mkdir /var/run/netns
root@vm3-hostname:/# ln -s /var/run/docker/netns/1-ec3ea5fa4f /var/run/netns/1-ec3ea5fa4f
root@vm3-hostname:/# ip netns exec 1-ec3ea5fa4f ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether 26:10:d6:c0:2c:0d brd ff:ff:ff:ff:ff:ff
    inet 10.0.4.1/24 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::7cf3:3bff:fe4a:a113/64 scope link 
       valid_lft forever preferred_lft forever
7: vxlan1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN group default 
    link/ether 26:10:d6:c0:2c:0d brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::2410:d6ff:fec0:2c0d/64 scope link 
       valid_lft forever preferred_lft forever
9: veth2@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP group default 
    link/ether c2:e7:15:46:5d:32 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::c0e7:15ff:fe46:5d32/64 scope link 
       valid_lft forever preferred_lft forever
```


```
root@vm3-hostname:/# ip netns exec 1-ec3ea5fa4f brctl show
bridge name	bridge id		STP enabled	interfaces
br0		8000.2610d6c02c0d	no		     veth2
	        			        	     vxlan1
```

### 测试Docker overlay网络 
 从vm2上的容器里ping vm3里的容器。
 
```
root@8293faef3880:/# ping 10.0.4.3
PING 10.0.4.3 (10.0.4.3) 56(84) bytes of data.    
64 bytes from 10.0.4.3: icmp_seq=1 ttl=64 time=0.435 ms    
64 bytes from 10.0.4.3: icmp_seq=2 ttl=64 time=0.468 ms    
64 bytes from 10.0.4.3: icmp_seq=3 ttl=64 time=0.639 ms    
```
  
### 网络拓扑图
   ![](/assets/vxlan.png) 
   
### 参考资料

- <http://shelan.org/blog/2016/03/01/docker-multi-host-networking-with-consul-and-vagrant/>       
- <http://hustcat.github.io/docker-overlay-network-practice/>
- <http://chunqi.li/2015/11/09/docker-multi-host-networking/>

 
  