---
title: "LangChain Note"
date: 2025-07-04T16:54:23+08:00
lastmod: 2025-07-04T16:54:23+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- AI
- LangChain
description: ""
weight:
slug: ""
draft: true # 是否为草稿
publishDate: 2026-07-04T16:54:23+08:00
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

Chain -> DAG 有向无环图
RAG 检索加强

## WorkFlow VS AI Agent

* 工作流(WorkFlow) 是指通过预定义的代码路径（predefined code paths）来协调 LLM 和工具的系统。
* 智能体(AI Agent) 则是指 LLM 能够动态地指引自身的过程和工具使用，保持对任务完成方式的控制。

## 工作流构件
### 工作流：路由(Routing) 
* 何时使用此工作流： 
  * 路由适用于那些任务复杂、可以明确区分为不同类别并且更适合单独处理的场景，同时分类可以通过 LLM 或更传统的分类模型/算法来准确完成。
* 路由有用的示例：
  * 将不同类型的客户服务查询（如常见问题、退款请求、技术支持）分配到不同的后续流程、提示词和工具中。
  * 将简单/常见问题路由到较小的模型（如 Claude 3.5 Haiku），将困难/不常见问题路由到更强大的模型（如 Claude 3.5 Sonnet），以优化成本和速度。

### 

WorkFlow是传统软件的AI包装版本。 
> Agent的价值不在于解决已知问题--而在于帮用户发现他们不知道自己有的问题。这种探索性体验,正是很多企业老板在 AI FOMO 情绪下真正想要的。       
> [《我曾嘲笑 AI Agent 只是概念炒作，不如 workflow 高效，今天我道歉》](https://x.com/topickai/status/1942203390001586186)

[从API到Agent：万字长文洞悉LangChain工程化设计](https://cloud.tencent.com/developer/article/2397398?policyId=1004)

[多AI Agent代理：使用LangGraph和LangChain创建多代理工作流](https://cloud.tencent.com/developer/article/2425298?policyId=1004)

[TKE 助力 Agent 可观测及评估体系建设，靠谱助手轻松养成！](https://cloud.tencent.com/developer/article/2425298?policyId=1004)

[构建高效的智能体(Buildingeffective agents)](https://hulz.life/post/20241220204806.html)


GPT Chat-GPT TransForm 多模态 数据标注 
