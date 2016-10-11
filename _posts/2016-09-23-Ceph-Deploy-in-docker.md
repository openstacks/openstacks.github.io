---
layout: post
title:  "如何在docker容器中安装ceph"
categories: Ceph
tags:  Ceph Docker ubuntu Storage
---

* content
{:toc}

本文介绍如何在Docker容器中安装ceph分布式存储。使用的操作系统是ubuntu 14.04。
将Ceph运行在Docker中是一个比较有争议的话题，不少人质疑这样操作的意义,但它不失为学习Ceph的一个好方法。




### 部署环境

本文在一台VM上部署一个ceph monitor，用sdb的三个分区作为三个osd设备。



####  安装docker  
    
   使用daocloud的加速方案，安装docker。
     
```   
   [root@sdnhubvm ~]$curl -sSL https://get.daocloud.io/docker | sh
   [root@sdnhubvm ~]$service docker start
   
```
   

####  下载mon和osd docker镜像   
 执行下面命令，从灵雀云<index.alauda.cn>下载ceph monitor和ceph osd镜像到本地。
 
```
[root@sdnhubvm ~]$ docker pull index.alauda.cn/georce/mon:hammer
[root@sdnhubvm ~]$ docker pull index.alauda.cn/georce/osd:hammer

root@sdnhubvm:~[00:04]$ docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
index.alauda.cn/georce/mon   hammer              30bbae8b195f        16 months ago       458.5 MB
index.alauda.cn/georce/osd   hammer              7015d29d98a0        16 months ago       460.1 MB

 
```
 
 
### 启动Ceph monitor

  执行下面的命令启动ceph monitor：
 
 
```
  查看Ceph monitor用的IP：
  root@sdnhubvm:~[18:10]$ ip a |grep eth0|grep inet
    inet 10.0.2.15/24 brd 10.0.2.255 scope global eth0
  
  root@sdnhubvm:~[00:09]$ export MON_IP=10.0.2.15    
  
  root@sdnhubvm:~[00:10]$ docker run -itd --name=mon --net=host -e MON_NAME=mymon \    
         -e MON_IP=$MON_IP -v /etc/ceph:/etc/ceph index.alauda.cn/georce/mon:hammer

```
#### 参数涵义：
 - 你可以配置如下选项：
 - MON_IP是运行Docker的主机IP
 - MON_NAME是你监测模块的名称（默认为$（hostname））
 - 由于监测模块不能在NAT过的网络中进行通信，我们必须使用 --net=host来将主机的网络层开放给容器
 
####  查看Ceph mon的运行日志

 
```
  root@sdnhubvm:~[00:11]$ docker logs -f mon
  creating /etc/ceph/ceph.client.admin.keyring
  creating /etc/ceph/ceph.mon.keyring
  monmaptool: monmap file /etc/ceph/monmap
  monmaptool: set fsid to 76169ceb-2710-486d-ad39-925cf52c4275
  monmaptool: writing epoch 0 to /etc/ceph/monmap (1 monitors)
  creating /tmp/ceph.mon.keyring
  importing contents of /etc/ceph/ceph.client.admin.keyring into /tmp/ceph.mon.keyring
  importing contents of /etc/ceph/ceph.mon.keyring into /tmp/ceph.mon.keyring
  ceph-mon: set fsid to 76169ceb-2710-486d-ad39-925cf52c4275
  ceph-mon: created monfs at /var/lib/ceph/mon/ceph-mymon for mon.mymo
  .......
  
```
 
 
####  查看mon生成的配置文件   

```
root@sdnhubvm:~[00:12]$ ls /etc/ceph
  ceph.client.admin.keyring  ceph.conf  ceph.mon.keyring  monmap

```

####  修改Ceph改集群配置文件


```
osd crush chooseleaf type = 0
osd journal size = 100
osd pool default pg num = 8
osd pool default pgp num = 8
osd pool default size = 1
public network = 10.0.2.0/24

```


###  启动 Ceph osd


####  准备osd设备
第一步先进行分区和格式化，可用建三个分区sdb1,sdb2,sdb3，用来做osd设备。

##### 磁盘分区

```
root@sdnhubvm:/ceph[05:18]$ parted /dev/sdb
GNU Parted 2.3
Using /dev/sdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel                                                          
New disk label type? gpt                                                  
continue?
Yes/No? Yes     

(parted) mkpart                                                           
Partition name?  []? 1                                                    
File system type?  [ext2]? xfs                                            
Start? 0G                                                                
End? 20G                                                                  
(parted) print                                                            
Model: ATA VBOX HARDDISK (scsi)
Disk /dev/sdb: 65.4GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt

(parted) mkpart                                                           
Partition name?  []? 2                                                    
File system type?  [ext2]? xfs                                            
Start? 20G                                                                
End? 40G                                                                  
(parted) print                                                            
Model: ATA VBOX HARDDISK (scsi)
Disk /dev/sdb: 65.4GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt

(parted) mkpart                                                           
Partition name?  []? 3                                                    
File system type?  [ext2]? xfs                                            
Start? 40G                                                                
End? 60G                                                                  
(parted) print                                                            
Model: ATA VBOX HARDDISK (scsi)
Disk /dev/sdb: 65.4GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt

(parted) mkpart  
Number  Start   End     Size    File system  Name  Flags
 1      1049kB  20.0GB  20.0GB  xfs          1
 2      20.0GB  40.0GB  20.0GB               2
 3      40.0GB  60.0GB  20.0GB               3

````

##### 格式化分区

```
root@sdnhubvm:/ceph[05:27]$ mkfs.xfs /dev/sdb2
meta-data=/dev/sdb2              isize=256    agcount=4, agsize=1220736 blks
         =                       sectsz=512   attr=2, projid32bit=0
data     =                       bsize=4096   blocks=4882944, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
root@sdnhubvm:/ceph[05:29]$ mkfs.xfs /dev/sdb3
meta-data=/dev/sdb3              isize=256    agcount=4, agsize=1220672 blks
         =                       sectsz=512   attr=2, projid32bit=0
data     =                       bsize=4096   blocks=4882688, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

````
##### 挂载目录

```
root@sdnhubvm:/ceph[05:31]$ mkdir /ceph/osd0
root@sdnhubvm:/ceph[05:31]$ mkdir /ceph/osd1
root@sdnhubvm:/ceph[05:31]$ mkdir /ceph/osd2
root@sdnhubvm:/ceph[05:31]$ mount /dev/sdb1 /ceph/osd0
root@sdnhubvm:/ceph[05:31]$ mount /dev/sdb2 /ceph/osd1
root@sdnhubvm:/ceph[05:31]$ mount /dev/sdb3 /ceph/osd2


[root@sdnhubvm ~]$lsblk -i
sdb                            8:16   0  60.9G  0 disk 
|-sdb1                         8:17   0   1.9G  0 part /ceph/osd0
|-sdb2                         8:18   0   1.9G  0 part /ceph/osd1
|-sdb3                         8:19   0   2.8G  0 part /ceph/osd2

```




#### 创建osd

```
[root@sdnhubvm ~]$ docker exec mon ceph osd create
0
[root@sdnhubvm ~]$ docker exec mon ceph osd create
1
[root@sdnhubvm ~]$ docker exec mon ceph osd create
2

[root@sdnhubvm ~]$ docker run -itd --name=osd0 --net=host -e CLUSTER=ceph \
     -e WEIGHT=1.0 -e MON_NAME=mymon -e MON_IP=$MON_IP -v /etc/ceph:/etc/ceph \
     -v /ceph/osd0/0:/var/lib/ceph/osd/ceph-0 index.alauda.cn/georce/osd:hammer    
 
[root@sdnhubvm ~]$ docker run -itd --name=osd1 --net=host -e CLUSTER=ceph \
     -e WEIGHT=1.0 -e MON_NAME=mymon -e MON_IP=$MON_IP -v /etc/ceph:/etc/ceph \    
     -v /ceph/osd1/1:/var/lib/ceph/osd/ceph-1 index.alauda.cn/georce/osd:hammer \    
     
[root@sdnhubvm ~]$ docker run -itd --name=osd2 --net=host -e CLUSTER=ceph \    
     -e WEIGHT=1.0 -e MON_NAME=mymon -e MON_IP=$MON_IP -v /etc/ceph:/etc/ceph \    
     -v /ceph/osd2/2:/var/lib/ceph/osd/ceph-2 index.alauda.cn/georce/osd:hammer

```


### 查看ceph群集状态

 从输出结果可以看出，状态不OK，是HEALTH_WARN状态。而且PG状态是undersized+degraded+peered。
 在下一篇blog里会分析问题的根源和解决办法。

```
[root@sdnhubvm ~]#docker exec -it mon ceph -s


 cluster 76169ceb-2710-486d-ad39-925cf52c4275
     health HEALTH_WARN
            128 pgs degraded
            128 pgs stuck degraded
            128 pgs stuck inactive
            128 pgs stuck unclean
            128 pgs stuck undersized
            128 pgs undersized
     monmap e1: 1 mons at {mymon=10.0.2.15:6789/0}
            election epoch 2, quorum 0 mymon
     osdmap e26: 3 osds: 3 up, 3 in
      pgmap v51: 128 pgs, 1 pools, 0 bytes data, 0 objects
            10343 MB used, 29679 MB / 40023 MB avail
                 128 undersized+degraded+peered


```

### 测试ceph群集

```
- step 1:Create a pool demo
[root@train ceph]# docker exec -it mon ceph osd pool create demo 128
pool 'demo' created

- step 2:进入osd0 容器
[root@train ceph]# docker exec -it osd0 bash

- step 3：生成一个文件
#echo "Is it working? :)" > testfile.txt

- step 4：把文件上传到demo pool
#docker exec -it mon rados put testfile testfile.txt --pool=demo

- step 5：删除文件
#rm testfile.txt 

- step 6：下载上传的文件
#docker exec -it mon rados get --pool=demo testfile testfile.txt

#rados -p demo bench 60 write

```
