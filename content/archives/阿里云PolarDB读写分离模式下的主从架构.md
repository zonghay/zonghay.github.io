---
title: "阿里云PolarDB读写分离模式的高可用架构"
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

## 高可用

通常情况下我们可以通过 **虚拟IP（VIP）** 技术和 **MHA（Master High Availability）** 在专有网络VPC内来自行搭建高可用Mysql主从集群，并在主节点故障时自动进行主从切换。
1. 搭建Mysql主从集群（一般为一主双从），配置数据同步策略
2. 创建VPC内部VIP，绑定到集群主节点
3. 部署MHA，并在MHA配置文件中中指定故障转移脚本
4. 当主节点故障时MHA根据主从复制状态和延迟，选择一个从节点成为新的主节点并执行VIP漂移脚本，将VIP绑定到新的主节点上（通常是10-30秒）
5. 待旧主节点恢复后，重新加入到集群中

![mermaid](https://kroki.io/mermaid/svg/eNqFkM9KAlEUxvc9xV3WwhcIEoSQWkxKQtu4OBcTptFmjGqngpRDE4GlZYKGiiuHFjFOM-XTzL0z8xbdP2T5j87iLu73-875ztHR2TlSs2g3D3MaPN0AtIpQK-Wz-SJUS0ACUAfSXgJIUIU5pC0BKUXmDGmNfMcLjWpQ_ViCDtCFgJpv66EMI3DN9r2m7939gfgjxeJxNmsbYKtNOl1cGWJ3Qvpl0h1yAColMGsuVP7PihljtAFzN0zsPpBxHzsO15Gio18jeaxF7c6C8ZL6SKsnrDONJWKLUY2nwAMT39iBYZNyZQ7K_EuINlG57jtT33HFmeYIsTqpj8LX26P9dDDyoufBpq6fbK3K84ORiYeN3lrs5Z1OShymg88GHj8tZo6uTSpHNTP4skRy4t6vaBNaU0poCMrHBVW52kklk-KwqvwNryn-Eg==)

上述流程在实际应用中面临以下问题：
1. 切换流程复杂，学习、维护成本较高，需要DBA专职维护
2. 由于切换操作设计到的步骤较多，所以一般需要10-30秒
3. 当遇到主从数据同步延迟较大时需要停止切换，人工介入

![polar_architecture.png](/images/Mysql/polar_architecture.png)
所以，后续当我们将线上服务陆续迁移到阿里云后，直接将数据库产品替换为PolarDB Mysql版并对数据进行迁移。           
相比于上述自建方式，在Mysql高可用方面PolarDB Mysql版:
1. 采用计算与存储分离的架构，计算节点仅保留元数据，数据文件等保存在共享分布式存储节点
2. 计算节点包括一主和多个只读节点，当系统发生故障时，可读写的主节点和只读节点之间会自动进行故障切换(开启热备下，影响5-10秒)
3. 各计算节点之间仅需同步Redo Log相关的元数据信息，极大地降低了主节点和只读节点间的复制延迟
4. 数据库存储节点的数据采用多副本形式，确保数据的可靠性，并通过Parallel-Raft协议保证数据的一致性。

对于自建Mysql可用选择mysqldump工具，在停服期间将数据迁移到PolarDB中。
[从自建MySQL迁移至PolarDB MySQL版（mysqldump工具）](https://help.aliyun.com/zh/polardb/polardb-for-mysql/user-guide/migrate-data-from-a-self-managed-mysql-database-to-a-polardb-for-mysql-cluster-by-using-mysqldump?spm=a2c4g.11186623.help-menu-2249963.d_5_2_3_1.3cc45170zilInK)

## 读写分离
PolarDB通过数据库计算节点上一层的数据库代理来支持读写分离。其对外默认提供两个地址，分别为主地址和集群地址。       
主地址可直接向主节点发送读写请求，集群地址会根据命令类型进行读写分发和多读节点间的负载均衡。      
在实际项目应用中，可以将业务一致性高的逻辑强制走主节点，其余的读请求走默认的集群地址。

## 数据一致性
在保证数据一致性方面，PolarDB提供三种数据一致性级别：
- **最终一致性**: 
  - 功能介绍：类似传统的读写分离架构，主从复制延迟可能会导致从不同节点查询到的结果不同。
  - 使用场景：若需要减轻主节点压力，让尽量多的读请求路由到只读节点，可以选择最终一致性。
- **会话一致性**：
  - 功能介绍：会话一致性保证了同一个会话（同一连接）内，一定能够查询到读请求执行前已更新的数据，确保了数据单调性。 在PolarDB的读写分离层会追踪各个节点已经应用的Redo日志位点，即日志序号（Log Sequence Number，简称LSN）。同时每次数据更新时PolarDB会记录此次更新的位点为Session LSN。当有新请求到来时，PolarDB会比较Session LSN和当前各个节点的LSN，如果只读节点的LSN不满足，会根据您配置的超时时间等待只读节点同步到最新数据，仅将请求发往LSN大于或等于Session LSN的节点，从而保证了会话一致性。
  - 使用场景：PolarDB的一致性级别越高，对主库的压力越大，集群性能也越低。推荐使用会话一致性，该级别对性能影响很小而且能满足绝大多数应用场景的需求。
- **全局一致性**:
  - 功能介绍：解决不同会话间存在数据逻辑因果依赖关系。每个读请求到达PolarDB数据库代理时，代理都会先去主节点确认当前最新的LSN位点，假设为LSN0（， 然后等待所有只读节点的LSN都更新至主节点的LSN0位点后，代理再将读请求发送至只读节点。这样就能保证该读请求能够读到至请求发起时刻为止，任意一条已完成更新的数据。
  - 使用场景：当主从延迟较高时，使用全局一致性可能会导致更多的请求被路由到主节点，造成主节点压力增大，业务延迟也可能增加。因此建议在读多写少的场景下选择全局一致性。

![polar_session_lsn.png](/images/Mysql/polar_session_lsn.png)

由于PolarDB一致性级别越高，集群性能越低。
所以官方推荐使用会话一致性，该级别对性能影响很小而且能满足绝大多数应用场景的需求。           
如果对于不同会话间的一致性需求较高，可使用HINT将特定查询强制发送至主节点执行。
```/*FORCE_MASTER*/ select * from user;```

## 参考
[PolarDB MySQL备份原理_云原生数据库 PolarDB(PolarDB)-阿里云帮助中心](https://help.aliyun.com/zh/polardb/polardb-for-mysql/user-guide/how-backup-works?spm=a2c4g.11186623.help-menu-2249963.d_5_11_5.6379211djwNfNi)          
[阿里云PolarDB及其共享存储PolarFS技术实现分析](https://sq.sf.163.com/blog/article/209129602406035456?tag=M_tg_373_65)          
[Mysql主从切换不一致问题](https://www.cnblogs.com/gered/p/16199141.html#_label1)             
[阿里云PolarDB及其存储PolarFS技术实现分析](https://zhuanlan.zhihu.com/p/44874330)