---
layout: post
title:  "如何测试CPU，IO和Mem 的benchmark"
categories: Performance 
tags:  Performance linux benchmark 
---

* content
{:toc}

本文通过实例来演示如何测试CPU，MEM和IO的benchmark。以此作为性能调优的一个基础。





###  CPU 性能测试

可以通过下面几个命令行测试CPU的性能。 对于CPU，通过时间Time来衡量其性能。   



#### sysbench    
     
 sysbench工具通过计算素数来测试CPU的性能，下面这个命令用四个线程来计算9999以内的素数，看看得花费多少时间，通过时间的长短而评估CPU的性能。     
 时间越短，说明CPU性能越好。   
    
   `#sysbench --test=cpu --num-threads=4 --cpu-max-prime=9999 run`   
   
> 线程的数量一般和cpu的数量相等
 
### IO性能测试
 
 磁盘IO的性能直接影响系统的整体性能和用户体验。通过测试工具得到IOPS和MBPS峰值来评估IO的性能。

#### dd命令测试写性能

下面这条命令在当前目录下产生大概1.6G的内容为0的文件，来评估IO的写性能。   

```
root@sdnhubvm:[12:06]$ dd bs=16k count=102400 oflag=direct if=/dev/zero of=test_data
102400+0 records in
102400+0 records out
1677721600 bytes (1.7 GB) copied, 12.3246 s, 136 MB/s

```

#### dd命令测试读性能  
下面这条命令读 test_data文件，然后把它丢弃到back hole里去，来测试读IO性能。

```
root@sdnhubvm:[12:18]$ dd bs=16K count=102400 iflag=direct if=test_data of=/dev/null
102400+0 records in
102400+0 records out
1677721600 bytes (1.7 GB) copied, 12.759 s, 131 MB/s

```

#### hdparm命令直接读性能  
下面这个命令在没有任何文件系统开销的情况下，测试读磁盘的性能。

```
root@sdnhubvm:[12:20]$ hdparm -t /dev/sda

/dev/sda:
 Timing buffered disk reads: 1728 MB in  3.00 seconds = 575.73 MB/sec
 
```

 
#### fio命令测试4KB随机写的IOPS 

下面通过fio来测试随机写的IOPS。

 > 注：在测试之前清空缓存
   `echo 1 >/proc/sys/vm/drop_caches`
 

```
$fio -ioengine=libaio -bs=4k -direct=1 -thread -rw=randwrite -size=1000G -filename=/dev/vdb 
-name="EBS 4K randwrite test" -iodepth=64 -runtime=60
```

> fio的参数:
- ioengine: 负载引擎，我们一般使用libaio，发起异步IO请求。
- bs: IO大小
- direct: 直写，绕过操作系统Cache。因为我们测试的是硬盘，而不是操作系统的Cache，所以设置为1。
- rw: 读写模式，有顺序写write、顺序读read、随机写randwrite、随机读randread等。
- size: 寻址空间，IO会落在 [0, size)这个区间的硬盘空间上。这是一个可以影响IOPS的参数。一般设置为硬盘的大小。
- filename: 测试对象
- iodepth: 队列深度，只有使用libaio时才有意义。这是一个可以影响IOPS的参数。
- runtime: 测试时长


### Mem性能测试

通过测试工具得到Mem的带宽和延时来评估Mem的性能。

lmbench这个工具可以测试的指标如下所示，功能是比较强大的。通过lmbench来测试Mem的Bandwidth和latency。

- Bandwidth benchmarks    
    `Cached file read`   
    `Memory copy (bcopy)`    
    `Memory read`    
    `Memory write`    
    `Pipe`    
    `TCP`    
- Latency benchmarks    
    `Context switching`    
    `Networking: connection establishment, pipe, TCP, UDP, and RPC hot potato`    
    `File system creates and deletes`    
    `Process creation`    
    `Signal handling`    
    `System call overhead`    
    `Memory read latency`    
- Miscellanious    
    `Processor clock rate calculation`    

### 参考文档

- <https://www.ustack.com/blog/how-benchmark-ebs/>       
- <http://www.latelee.org/using-gnu-linux/linux-memory-bandwidth-test-note.html>

 
 
 
 
  
  