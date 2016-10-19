---
layout: post
title:  "如何构建SDN应用开发环境"
categories: SDN
tags:  SDN Openflow Opendaylight Network
---

* content
{:toc}

本文介绍如何构建一个在Opendaylight上开发SDN应用的开发环境的。英文   
参考资料可以参考<https://sdnhub.org/tutorials/opendaylight/>







### 开发环境安装

####  下载虚拟机        
   从[SDN Hub Tutorial VM ](https://sdnhub.org/tutorials/sdn-tutorial-vm/)下载vm文件到本地。

####  使用虚拟机   
 - 把下载的VM镜像导入到Virtualbox或者VMware Player，然后启动  
 - vm的用户名和密码都是ubuntu  
 - 登录到vm，测试从vm可以访问internet  

```
 ubuntu@sdnhubvm:~[00:41]$ ping 8.8.8.8
 PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
 64 bytes from 8.8.8.8: icmp_seq=1 ttl=63 time=194 ms
 64 bytes from 8.8.8.8: icmp_seq=2 ttl=63 time=176 ms
 
```
 
 
### 安装Opendaylight 
####  更新代码
 登录到vm后，执行下面的命令获取最新的源代码：
 
 
```
  ubuntu@sdnhubvm:~$ cd SDNHub_Opendaylight_Tutorial
  ubuntu@sdnhubvm:~$ git pull --rebase
  
```
 
 
####  升级Maven
vm自带的Maven版本比较老，会导致编译opendaylight失败，执行下面的命令升级到3.2.1：
 
```
 ubuntu@sdnhubvm:~[00:41]$sudo apt-get remove maven*
 ubuntu@sdnhubvm:~[00:41]$sudo apt-get install gdebi
 ubuntu@sdnhubvm:~[00:41]$wget http://ppa.launchpad.net/natecarlson/maven3/ubuntu/pool/main/m/maven3/maven3_3.2.1-0~ppa1_all.deb    
 ubuntu@sdnhubvm:~[00:41]$sudo gdebi maven3_3.2.1-0~ppa1_all.deb
 
```
 
 
####  获取settings.xml   
执行下面的命令下载settings.xml


```
#wget -q -O - https://raw.githubusercontent.com/opendaylight/odlparent/master/settings.xml > /root/.m2/settings.xml

```

####  执行build
   
执行下面的命令build opendaylight.


```
root@sdnhubvm:~/maven install -nsu -X

```


```
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
[INFO] 
[INFO] SDN Hub Tutorial project common properties ........ SUCCESS [  2.965 s]
[INFO] SDN Hub tutorial project common utils ............. SUCCESS [  3.211 s]
[INFO] learning-switch-parent ............................ SUCCESS [  0.090 s]
[INFO] SDN Hub tutorial project learning switch Impl ..... SUCCESS [ 11.785 s]
[INFO] SDN Hub tutorial project Learning Switch Config ... SUCCESS [  0.725 s]
[INFO] SDN Hub tutorial project Tap application Model .... SUCCESS [  4.727 s]
[INFO] tapapp-parent ..................................... SUCCESS [  0.055 s]
[INFO] SDN Hub tutorial project Tap application Impl ..... SUCCESS [  9.506 s]
[INFO] SDN Hub tutorial Project Tap application Config ... SUCCESS [  0.167 s]
[INFO] SDN Hub tutorial project ACL application Model .... SUCCESS [  1.125 s]
[INFO] acl-parent ........................................ SUCCESS [  0.061 s]
[INFO] SDN Hub tutorial project ACL application Impl ..... SUCCESS [  8.451 s]
[INFO] SDN Hub tutorial Project ACL application Config ... SUCCESS [  0.185 s]
[INFO] SDN Hub tutorial project Netconf exercise Model ... SUCCESS [  1.394 s]
[INFO] netconf-exercise-parent ........................... SUCCESS [  0.082 s]
[INFO] SDN Hub tutorial project Netconf exercise Impl .... SUCCESS [ 20.803 s]
[INFO] SDN Hub tutorial Project Netconf exercise Config .. SUCCESS [  0.223 s]
[INFO] SDN Hub tutorial project features ................. SUCCESS [11:55 min]
[INFO] distribution-parent ............................... SUCCESS [  0.119 s]
[INFO] SDN Hub tutorial project Karaf branding ........... SUCCESS [  0.387 s]
[INFO] SDN Hub tutorial project distribution packaging ... SUCCESS [05:44 min]
[INFO] main .............................................. SUCCESS [  0.059 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 18:47 min
[INFO] Finished at: 2016-09-22T00:37:46-08:00
[INFO] Final Memory: 119M/568M



```

####  启动opendaylight
执行下面的命令，启动opendaylight。

```

  ＃cd SDNHub_Opendaylight_Tutorial/distribution/opendaylight-karaf/target/assembly/bin
  ＃./karaf
  
```


```
karaf: JAVA_HOME not set; results may vary
karaf: Enabling Java debug options: -Xdebug -Xnoagent -Djava.compiler=NONE -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=5005
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=512m; support was removed in 8.0
Listening for transport dt_socket at address: 5005
      _____ _____  _   _   _    _       _     
     / ____|  __ \| \ | | | |  | |     | |    
    | (___ | |  | |  \| | | |__| |_   _| |__  
     \___ \| |  | | . ` | |  __  | | | | '_ \ 
     ____) | |__| | |\  | | |  | | |_| | |_) |
    |_____/|_____/|_| \_| |_|  |_|\__,_|_.__/ 

Hit '<tab>' for a list of available commands
and '[cmd] --help' for help on a specific command.
Hit '<ctrl-d>' or type 'system:shutdown' or 'logout' to shutdown OpenDaylight.

opendaylight-user@root>feature:list | grep sdnhub
sdnhub-tutorial-netconf-exercise           | 1.0.0-SNAPSHOT      |           | tutorial-features-1.0.0-SNAPSHOT           | SDN Hub Tutorial :: OpenDaylight :: Netconf exerci
sdnhub-tutorial-tapapp                     | 1.0.0-SNAPSHOT      | x         | tutorial-features-1.0.0-SNAPSHOT           | SDN Hub Tutorial :: OpenDaylight :: Tap applicatio
sdnhub-tutorial-learning-switch            | 1.0.0-SNAPSHOT      |           | tutorial-features-1.0.0-SNAPSHOT           | SDN Hub Tutorial :: OpenDaylight :: Learning switc
sdnhub-tutorial-acl                        | 1.0.0-SNAPSHOT      |           | tutorial-features-1.0.0-SNAPSHOT           | SDN Hub Tutorial :: OpenDaylight :: Access Control

```