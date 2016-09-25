---
layout: post
title:  "跟我一步一步学习ARM开发"
categories: ARM
tags:  ARM C IOT
---

* content
{:toc}


本文从学习者的角度出发,分别描述了下面几部分内容:
 - ARM编程的基本知识
 - BOOT代码流程和功能分析
 - OS中断程序的编写举例
 - BOOT代码的流程图  

希望这些内容能为初学ARM的朋友拨开迷雾,以最快的速度和最短的时间走进嵌入世界的大们。







###  资料的获取方式  

  - 直接阅读电子书：     
      {% include_relative test.html %}
  - 从baidu网盘下载：    
      <http://pan.baidu.com/s/1o7IHvx4>

  - 出版书购:     
     京东：<http://item.jd.com/10116867.html>    
     易购: <http://www.egou.com/product/01_1080927.html>  
   
   > > ![](/51HAvkjRIzL._AA190_.jpg)


### 目录

#### 第一章 ARM ABC

##### The ARM Processor

- 缩写
- 处理器模式及对应的寄存器

- ARM寄存器总结

##### ARM Instructions

- 指令集概述

- 指令的条件执行

- 程序分支

- Data Movement Memory Reference Instructions

##### Examples

- 向量乘

- 字符串比较

- 子程序调用

#### 第二章:引导代码分析

##### 前言

##### 概述

- 与BOOT相关硬件:FLASH ROM

- BOOT的主要功能

##### 执行流程及代码分析

- 参数初始化

- 中断

- 初始化硬件

- 跳转到C语言程序，开始第二阶段的初始化和系统引导.

- 初始化堆栈

#### 第三章:中断服务程序编写

##### 必需的变量定义

- 服务程序地址

- I/O端口

- INTERRUPT 控制寄存器

- EINT4567的Pending 位

##### 变量解释

##### 中断服务程序的实现

- 定义中断服务程序

- 主程序

- 中断服务子程序中关键的变量类型

- 断服务程序运行流程图

#### 第四章:ＢＯＯＴ流程图

#### 附录:BOOT程序源代码
   

### 问题反馈   

学习过程中用到的任何问题，可以发邮件到：840814120@qq.com