---
title: "阿里云PolarDB读写分离模式的主从架构"
date: 2025-04-17T14:56:11+08:00
lastmod: 2025-04-17T14:56:11+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- PolarDB
- Mysql
- 阿里云
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

大型服务架构往往对于RDS数据库的高并发、高可用和数据一致性有着严格要求，本文将介绍实际项目中使用阿里云PolarDB MySQL版本搭建满足上述要求的RDS数据库的经验。
![polar_architecture.png](/images/Mysql/polar_architecture.png)

## 主从备份

热备切换：连接/事务不中断，短暂阻塞5~10秒

## 数据一致性

数据一致性：
- 最终一致性
- 会话一致性
- 全局一致性

## 读写分离
![polar_proxy.png](/images/Mysql/polar_proxy.png)

PolarDB集群版是一个由多节点构成的数据库集群，包括一个**主节点**和多个**只读节点**。对外默认提供两个地址，分别为主地址和集群地址。其中，集群地址功能由PolarDB数据库代理提供，集群地址分为可读可写（自动读写分离）和只读两种读写模式。

多节点的架构可用于保障集群的高可用，当系统发生故障时，可读写的主节点和只读节点之间会自动进行故障切换（Failover）。

## 监控和告警

## 参考
[PolarDB MySQL备份原理_云原生数据库 PolarDB(PolarDB)-阿里云帮助中心](https://help.aliyun.com/zh/polardb/polardb-for-mysql/user-guide/how-backup-works?spm=a2c4g.11186623.help-menu-2249963.d_5_11_5.6379211djwNfNi)          
[阿里云PolarDB及其共享存储PolarFS技术实现分析](https://sq.sf.163.com/blog/article/209129602406035456?tag=M_tg_373_65)