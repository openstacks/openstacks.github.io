---
layout: post
title:  "一个Host里面多个OSD的Crush Rule"
categories: Ceph
tags:  Ceph CRUSH Storage
---

* content
{:toc}

当我们把Ceph安装在一个Host里，并且在这个Host里有多个osd。安装很顺利，但Ceph的状态却是不对的。本文分析了其中的原因，并提供解决办法。
通过这个问题，可以学习了解Ceph CRUSH map的基本原理。




### 预备知识

 - Ceph管理
 - Ceph CRUSH MAP的原理



#### 问题描述  
    
   当我们装完Ceph后，执行状态检查命令，发现状态是不对的，如下所示。
   health状态是：HEALTH_WARN，而且pg的状态是undersized+degraded+peered
     
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
   

####  问题分析   

 peered关键字给我们一个提示，在加上Ceph osd在同一个Host上，所以我们就怀疑是不是replication policy的问题。
 下面是详细的步骤：

##### 检查当前pool用的rule

  先检查一个ceph里都有那些pool。
  
```
  root@sdnhubvm:~[08:09]$ docker exec -it mon ceph osd lspools
   0 rbd,

```
上面的命令的结果显示当前系统只有一个pool：rbd。

##### 检查replication policy

检查当前pool用的replication policy

```
root@sdnhubvm:~[08:11]$ docker exec -it mon ceph osd dump --format=json-pretty

 "pools": [
        {
            "pool": 0,
            "pool_name": "rbd",
            "flags": 1,
            "flags_names": "hashpspool",
            "type": 1,
            "size": 3,
            "min_size": 2,
            "crush_ruleset": 1, 
            
````
上面的命令显示"crush_ruleset": 1，表明rbd pool用的是ruleset是1。下面我们查看一下这个ruleset在crushmap里的具体内容。

```
  root@sdnhubvm:~[08:17]$ docker exec -it mon ceph osd crush dump --format=json-pretty

      {
            "rule_id": 1,
            "rule_name": "same-host",
            "ruleset": 1,
            "type": 1,
            "min_size": 1,
            "max_size": 10,
            "steps": [
                {
                    "op": "take",
                    "item": -1,
                    "item_name": "default"
                },
                {
                    "op": "choose_firstn",
                    "num": 0,
                    "type": "host"
                },
                {
                    "op": "emit"
                }
            ]
        }
```

从上面输出的结果，可以看出："type": "host"，host意味着peer必须是个host。可在这个环境里，ceph只装在一个Host上。这就是问题的根源所在。


####  解决方法
 
 解决方法很简单：把host改成osd即可。   

#####  新建一个ruleset

```
root@sdnhubvm:~[08:17]$ docker exec -it mon ceph osd crush rule create-simple same-host default osd

```

#####  查看新建一个ruleset

通过下面的命令查看新建的ruleset的id。

```
root@sdnhubvm:~[08:25]$ docker exec -it mon ceph osd crush dump

   {
            "rule_id": 1,
            "rule_name": "same-host",
            "ruleset": 1,
            "type": 1,
            "min_size": 1,
            "max_size": 10,
            "steps": [
                {
                    "op": "take",
                    "item": -1,
                    "item_name": "default"
                },
                {
                    "op": "choose_firstn",
                    "num": 0,
                    "type": "osd"
                },
                {
                    "op": "emit"
                }
            ]
        }
```
从上面的输出可知，id是1。

#####  使用ruleset

让rbd pool使用这个ruleset。

```
root@sdnhubvm:~[08:28]$ docker exec -it mon ceph osd pool set rbd crush_ruleset 1

```

####  查看结果

最后检查一个Ceph cluster的状态是否正常。一切正常。

```
root@sdnhubvm:~[08:28]$ docker exec -it mon ceph -s
    cluster 76169ceb-2710-486d-ad39-925cf52c4275
     health HEALTH_OK
     monmap e1: 1 mons at {mymon=10.0.2.15:6789/0}
            election epoch 2, quorum 0 mymon
     osdmap e194: 3 osds: 3 up, 3 in
      pgmap v369: 128 pgs, 1 pools, 0 bytes data, 0 objects
            10494 MB used, 29528 MB / 40023 MB avail
                 128 active+clean
  
```
