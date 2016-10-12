---
layout: post
title:  "Ceph寻址原理"
categories: Ceph
tags:  Ceph OSD CRUSH Hashing Algorithm
---

* content
{:toc}

本文介绍Ceph中最为核心的、基于计算的对象寻址机制及其相关的概念解释。





###  寻址流程

Ceph系统中的寻址流程如下图所示： 

 > ![](/assets/ceph.jpeg)  
 
 本图来源于Ceph论文 [Ceph论文](http://ceph.com/papers/weil-ceph-osdi06.pdf)
 
 > 论文对上图描述的寻址过程描述如下：    
 Files are striped across many objects, grouped into placement groups (PGs), and distributed to OSDs via CRUSH, a specialized replica placement function.    
 翻译成中文：    
  Step 1: 将需要保存的file切分成大小一致的一系列object        
  Step 2: 把objects组织到PGs里    
  Step 3: 最后通过CRUSH保存到OSD    



###  对象映射 

#### File到object的映射    
     
 > (ino,ono)->oid    
    
从上图可以看出，第一层映射是file到object的映射。    
具体过程是将需要保存的file，转换为RADOS能够处理的object。类似RAID中的条带化过程，按照object的最大size对file进行切分，把file切分成一批大小一致的数据块object。
   

   
 
#### Object到PG的映射:
 
 > hash(oid) & mask -> pgid
   
  在把file切分为一批object后，紧接着要做的是，需要把每个object独立地映射到一个PG中去。也就是得到每个object对应的PG。   
   
  > 计算方法: 首先计算object的Hash值并将结果和PG数目取余，以得到object对应的PG编号。        
  > 公式如下：     
     hash(oid) % PG Num -> pgid
     
Ceph中在对象和设备之间有两个概念，Pool和Placement Group(PG)，每个对象要先计算对应的Pool，然后计算对应的PG，通过PG可得到该对象对应的多个副本的位置，三个副本中第一个是Primary，其余被称为replcia。
假设一个对象image，其所在的pool是glance，计算device的方式如下：
1. 计算image的hash值得到0x3F4AE6E3
2. 计算glance的pool id得到3
3. pool glance中的PG数量为256，0x3F4AE6E3 mod 256 = 23，所以PG的id为3.23
  
  {% include_relative cephhash.html %}  
  
      
 
#### PG到OSD的映射 

 > CRUSH(pgid) -> (osd1,osd2,osd3) 
 
 最后一步是通过CRUSH算法将PG映射到一组OSD中。最终结果是file切分的object就会存放在在这一组OSD中。     
 CRUSH算法的目的是，为给定的PG分配一组存储数据的OSD节点。
 
 CRUSH算法通过每个设备的权重来计算数据对象的分布。对象分布是由cluster map和data distribution policy决定的。      
 - Cluster map描述了可用存储资源和层级结构(比如有多少个机架，每个机架上有多少个服务器，每个服务器上有多少个磁盘)。     
 - Data distribution policy由placement rules组成。rule决定了每个数据对象有多少个副本，这些副本存储的限制条件(比如3个副本放在不同的机架中)。
 - CRUSH算出PG到一组OSD集合(OSD是对象存储设备)
 
 CRUSH根据cluster map, rule和pgid算出x到一组OSD集合(OSD是对象存储设备)：    
 > (osd0, osd1, osd2 … osdn) = CRUSH(cluster, rule, pgid)
 
 CRUSH算法计算出PG中目标主和次OSD的ID[24, 3, 12]，其中24为primary,3,12为replica
 
 - ceph中Pool的属性有：1.object的副本数  2.Placement Groups的数量  3.所使用的CRUSH Ruleset

##### Cluster map结构

Cluster map由device和bucket组成，它们都有id和权重值。Bucket可以包含任意数量item。item可以都是的devices或者都是buckets。
管理员控制存储设备的权重。权重和存储设备的容量有关。Bucket的权重被定义为它所包含所有item的权重之和。     
 结构图如下：    
 
  > ![](/assets/crushmap.jpg) 
  
  - Ceph使用树型层级结构描述OSD的空间位置以及权重(同磁盘容量相关)大小。
  - 如上图所示，层级结构描述了OSD所在主机、主机所在机架以及机架所在机房等空间位置。
  - 这些空间位置隐含了故障区域，例如使用不同电源的不同的机架属于不同的故障域。
  - CRUSH能够依据一定的规则将副本放置在不同的故障域。
  - OSD节点在层级结构中也被称为Device，它位于层级结构的叶子节点，所有非叶子节点称为Bucket。
  - Bucket拥有不同的类型，如上图所示，所有机架的类型为Rack，所有主机的类型为Host。
  - 使用者还可以自己定义Bucket的类型。
  - Device节点的权重代表存储节点的性能，磁盘容量是影响权重大小的重要参数。Bucket节点的权重是其子节点的权重之和


### 参考资料

- [shanno吳：Ceph剖析](http://www.cnblogs.com/shanno/p/3958298.html)
- [陈小跑：CRUSH算法](http://www.cnblogs.com/chenxianpao/p/5568207.html)
- [刘世民：Openstack与Ceph](http://www.cnblogs.com/sammyliu/p/4836014.html)

   感谢以上作者的无私分享