---
title: "OpenTelemetry+Tempo+Grafana搭建分布式Trace监控"
date: 2025-03-08T14:33:56+08:00
lastmod: 2025-03-19T14:33:56+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- Go
- 分布式
- Trace
- OpenTelemetry
- Grafana
- Tempo
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

上篇文章提到了Context[在OpenTelemetry-Go中的应用](https://zonghay.github.io/archives/golang%E5%8E%9F%E7%94%9F%E5%8C%85%E4%B9%8Bcontext/#%e5%9c%a8opentelemetry-go%e4%b8%ad%e7%9a%84%e5%ba%94%e7%94%a8)          
其中的经验来自于之前工作中对公司内部分布式服务做链路追踪的经历，所以这篇文章来做下关于分布式链路追踪的实践回顾。

## 私有规范
最开始接触是公司准备针对内网微服务间的调用来做链路追踪，想法来自于Google的那篇著名文章[《Dapper, a Large-Scale Distributed Systems Tracing Infrastructure》](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/papers/dapper-2010-1.pdf)

文章主要介绍Google内部使用Dapper来监控分布式系统的性能和异常问题，并介绍以下概念和实现细节：
* 使用TraceID概念来唯一标识一次请求，可以把TraceID当作一棵Trace链路追踪树的唯一ID
* 使用Span来表示分布式链路中每个工作单元，一个Span即Trace树的一个节点。
* 每个span都有一个唯一的span ID和一个父span ID，用于构建链路调用结构(数/瀑布图)。没有父span ID的span称为根span。
* 使用采样率对应用服务进行指标采集，从而实现低开销接入，避免因引入Dapper造成业务性能影响。
  * > the tracing system should have negligible performance impact on running services. In some highly optimized services even small monitor- ing overheads are easily noticeable, and might compel the deployment teams to turn the tracing system off
* Dapper通过hook常见库行为来实现业务无感知接入，如基础RPC、线程和控制流库。
  * > Perhaps the most critical part of Dapper’s code base is the instrumentation of basic RPC, threading and control flow libraries, which includes span creation, sampling, and logging to local disks.

在介绍我们当时私有化实现Trace架构的细节之前，有必要针对当时公司内部微服务架构的特点进行一些说明：
1. 内部微服务以HTTP Restful API调用为主，主要集中在各业务模块间如商场、主页、支付等。
2. 所有的服务端请求都会经过两大网关(OpenResty实现)，分别处理来自外部的请求和内部微服务间请求。
3. 服务端框架以PHP、Go两种语言为主，自研和开源均有涉及。
4. 客户端、服务端、网关日志统一上报至日志中心，做后续检索、监控和实时/离线计算等处理。

所以针对上述特点和对Dapper的理解，我们实现了第一版私有的Trace链路追踪架构：
* 针对所有经过统一网关的HTTP请求，我们和网关伙伴合作通过制定以下规范
  * 所有经过网关的请求透传HTTP Header里的Trace ID和Span ID
  * 网关服务在请求HTTP Header里识别Trace ID
    * 初始TraceID由客户端生成，如果没有携带则由网关补充
    * TraceID生成规则：毫秒级时间戳+IP+进程ID+自增序列
  * 网关服务在请求HTTP Header里识别Span ID
    * 如果没有则生成根Span即:"1"
    * 如果有则对Span ID进行迭代，在原有基础上++，如"1"->"2"，"2"->"3"以此类推
    * 应用服务可通过请求获取SpanID，可在逻辑中自行对SpanID进行尾部追加，如"1"->"1.1"，"2"->"2.1"等
    * 无论网关日志还是服务端自定义日志，最终都会通过服务端日志采集进入Trace日志存储模块
* 需要服务端发起请求时在HTTP Header里携带Trace数据。由于服务端框架太多难以统一，无法像Google那样通过hook底层库来实现无侵入式接入，所以我们采用两种策略
  * 自研框架通过升级HTTP SDK实现
  * 其余框架需要业务配合在代码中携带




自定义网关日志 + 日志平台
网关+服务基础框架支持

宏观Span(Nginx透传)、微观Span(服务端传递)

TraceID生成规则
SpanID生成规则

## OpenTelemetry开源规范
OpenTelemetry + Tempo + Grafana






## 参考
[阿里云-可观测链路OpenTelemetry版](https://help.aliyun.com/zh/opentelemetry/quick-start?spm=a2c4g.11186623.help-menu-90275.d_1.20352e39UBASEA&scm=20140722.H_91317._.OR_help-T_cn~zh-V_1)                 
[How to send traces to Grafana Cloud's Tempo service with OpenTelemetry Collector](https://grafana.com/blog/2021/04/13/how-to-send-traces-to-grafana-clouds-tempo-service-with-opentelemetry-collector/)

