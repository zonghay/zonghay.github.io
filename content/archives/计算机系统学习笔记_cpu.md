---
title: "计算机系统学习之CPU"
date: 2024-08-29T17:49:10+08:00
lastmod: 2024-08-29T17:49:10+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- CPU
- Computer System
description: ""
weight:
slug: ""
draft: false # 是否为草稿
comments: true # 本页面是否显示评论
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hideMeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
hideSummary: false
ShowWordCount: true
ShowBreadCrumbs: true #顶部显示路径
cover:
    image: "" #图片路径例如：posts/tech/123/123.png
    zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false 
---
# CPU

## 从晶体管到运算器

简单晶体管 -> AND/OR/NOT门 -> 加法（与或非门）-> 运算器（arithmetic/logic unit，ALU）

![](/images/CPU/jingtiguan.png)

## 寄存器

简单晶体管 -> NAND门（与非门）-> 存储1bit位 -> 寄存器

![](/images/CPU/jicunqi.png)

## 指令集

CPU运算器的菜谱，最底层也是最基础的计算机语言。

![](/images/CPU/zhilingji.png)

这条指令占据16比特，其中前四个比特告诉CPU这是加法指令，这意味着该CPU的指令集中可以包含2^4也就是16个机器指令，这四个比特位告诉CPU该做什么，剩下的bit告诉CPU该怎么做，也就是把寄存器R6和寄存器R2中的值相加然后写到寄存器R6中。

所以汇编语言是最接近指令集的底层语言，可执行文件就是一条条指令的集合，运行时加载到内存只交给CPU执行。

### 复杂指令集 Complex Instruction Set Computer

当今普遍存在于桌面PC以及服务器端的x86架构就是基于复杂指令集CISC，生产x86处理器的厂商就是英特尔以及AMD。（目前最常见的复杂指令集x86 CPU，虽然指令集是CISC的，但因对常用的简单指令会以硬件线路控制尽全力加速，不常用的复杂指令则交由微码循序器“慢慢解码、慢慢跑”，因而有“RISCy x86”之称。）    

复杂指令集的本质就是提供更丰富更方便占用内存更小的机器指令来实现程序。     
但与此带来的问题就是，需要支持复杂指令集的CPU硬件设计太过复杂。       

### 简单指令集 Reduced Instruction Set Computer

内存越来越便宜，加上编译技术的成熟催生简单指令集的诞生。科学家化繁为简，把复杂指令集转化为更小的简单指令集，转而让编译器去组合复杂操作。        
RISC中每条指令更加简单，执行时间比较标准，因此可以很高效的利用流水线技术，这一切都让采用RISC架构的CPU获得了很好性能。


## 控制器


## 时钟信号

控制运算器、寄存器协同工作的「指挥家」。    
主频越高，CPU一秒内完成的操作越多。     
谐振器产生脉冲，一个脉冲即一个时钟信号。

# 参考

[你管这破玩意叫 CPU ？](https://mp.weixin.qq.com/s?__biz=Mzg4OTYzODM4Mw==&mid=2247485736&idx=1&sn=a70558b5200e840ef251e19a2eef099b&chksm=cfe995a8f89e1cbe8fab1240515f35ec90fb520d122ec60761b71a8664ae3af390689be370aa#rd)



