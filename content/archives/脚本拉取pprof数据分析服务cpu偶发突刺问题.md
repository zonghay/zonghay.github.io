---
title: "脚本拉取pprof数据分析服务cpu偶发突刺问题"
date: 2024-06-19T15:11:19+08:00
lastmod: 2024-06-19T15:11:19+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- Golang
- pprof
description: "脚本拉取pprof数据分析服务cpu偶发突刺问题"
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

最近工作中遇到一个比较奇怪的问题，容器化部署的服务container会在不定时偶发CPU突刺，从而引发频繁动态扩容。
服务是go开发的，所以准备让服务提供pprof接口来分析CPU的使用情况。
但由于CPU突刺是不定时的，并且网关一般都有请求读写超时配置。
所以我们不能一次拉取太长时间的pprof数据，需要通过shell脚本每分钟把数据转存到特定路径。
最后再结合出现问题时的pprof文件监控进行分析。   

运行

```shell
go tool pprof /tmp/dump/xxx.pprof
```

查看对应时间服务CPU采样数据。    

shell脚本如下：
```shell
#!/bin/bash

# 创建保存采样结果的目录
output_dir="/tmp/dump"
mkdir -p "${output_dir}"

# 定义采样的总时间（单位：秒）
total_duration=3600

# 获取当前时间戳
start_time=$(date +%s)

# 循环采样，直到达到总时间
while [ $(($(date +%s) - start_time)) -lt ${total_duration} ]; do
    # 生成输出文件名，时间格式为小时-分钟
    timestamp=$(date +%Y-%m-%d-%H-%M)
    output_file="${output_dir}/cpu_profile_${timestamp}.pprof"

    # 执行采样，采样时间设为60s
    if curl -o "${output_file}" "http://localhost/debug/pprof/profile?seconds=60"; then
        echo "Profile saved to: ${output_file}"
    else
        echo "Error: Failed to download CPU profile."
        exit 1
    fi
done

echo "Profiling completed."
```

后续通过pprof生成的graph图，我们将问题定位到某段用于解析用户缓存数据的json.unmarshall代码处。     
这里补充一个必要信息是，由于游戏刚上线，运维初期给我这个服务容器配置的资源较低，所以更容易引发CPU突刺。   
由于问题处缓存的文本量较大且struct对象的类型也较为复杂，所以当稍微请求量多一些时，就会造成unmarshall占用更多计算资源。所以，后面我们决定给容器加资源并更改json包(valyala/fastjson)来解决此问题。

参考：[深入 Go 中各个高性能 JSON 解析库](https://www.luozhiyun.com/archives/535)
