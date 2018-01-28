如何自动化大批量生成固定大小的docker镜像


---
layout: post
title:  "如何自动化大批量生产固定大小的docker镜像 "
categories: Docker 
tags:  Docker images s2i performance
---

* content
{:toc}


## 引子
 在测试和生产环境中，都会用自己的私有镜像仓库，需要测试pull和push的性能，所以需要准备大量的docker镜像。本文的目的就是解决如何自动化大批量生成docker镜像。
 
## 用busybox基础镜像
用busybox作为基础镜像，使用dd命令附加大小不等的层，来批量生成docker镜像。


```
#!/bin/bash


if [ $# -lt 1 ]
then
        echo "Usage: $0 Number TotalSize"
        echo "Number: The number of image"
        echo "TotalSize: Total storage size which image will be consumed"
        exit -1
fi

echo "Image num:$1M"
echo "Total storage size: $2M"

#Average size every images
avg=$(($2/$1))

for((i = 1; i <= $1; i++))
do
  docker ps -f name=perfimages$1|grep perfimages$i
  if [ $? -eq 0 ];then
    docker stop perfimages$i
    docker rm perfimages$i
  fi
  s=$(($RANDOM%$avg+10))
  echo $s
  docker run --name perfimages$i busybox /bin/sh -c "dd if=/dev/zero of=test.file bs=1M count=$s"
  docker commit perfimages$i 172.30.247.186:5000/perfimages$i:latest
  docker rm perfimages$i
done

```

## 使用红帽的s2i工具
s2i工具可以把git仓库里的源代码直接生成docker镜像。不同语言开发的代码使用不同的基础镜像，下面这个例子是一个ruby的例子。她使用的基础镜像里已经有了ruby语言的工具集。

关于s2i的原理，可以参考openshift的相关文档


```
#!/bin/bash

# Usage
if [ $# -lt 1 ]
then
        echo "Usage: $0 Number TotalSize"
        echo "Number: The number of image"
        echo "TotalSize: Total storage size which image will be consumed"
        exit -1
fi

echo "Image num:$1M"
echo "Total storage size: $2M"


# Average size every images
avg=$(($2/$1))

# S2I produces ready-to-run images by injecting source code into a Docker container
for((i = 1; i <= $1; i++))
do
  docker ps -f name=rubycontainer$1|grep rubycontainer$i
  if [ $? -eq 0 ];then
    docker stop rubycontainer$i
    docker rm rubycontainer$i
  fi
  s=$(($RANDOM%$avg+10))
  echo $s
  docker run --name rubycontainer$i centos/ruby-23-centos7 /bin/sh -c "dd if=/dev/zero of=test.file bs=1M count=$s"
  docker commit rubycontainer$i rubyimages$i:latest
  docker rm rubycontainer$i
  s2i build https://github.com/openshift/ruby-hello-world rubyimages$i 172.30.247.186:5000/perfimages$i --loglevel 5
  docker rmi rubyimages$i
  docker push 192.168.100.186:5000/perfimages$i
done

```
