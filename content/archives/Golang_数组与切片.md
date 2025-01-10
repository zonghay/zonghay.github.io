---
title: "Golang 数组与切片"
date: 2025-01-09T15:19:07+08:00
lastmod: 2025-01-09T15:19:07+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- Golang
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

## 数组和切片的异同

数组是定长的，是一片连续的内存，其长度是类型的一部分，定义好之后就不能再更改，比如 [3]int 和 [4]int 就是不同的类型。
所以在日常开发中，我们更多使用的是切片(slice)。

slice 实际上是一个结构体，包含三个字段：长度、容量、底层数组，可以动态扩容，更加灵活。

```
// runtime/slice.go
type slice struct {
	array unsafe.Pointer // 元素指针
	len   int // 长度 
	cap   int // 容量
}
```

![](/images/Go/slice_array.png)

**注意：** 底层数组是可以被多个 slice 同时指向的，因此对一个 slice 的元素进行操作是有可能影响到其他 slice 的。

| 特性     | 数组     | 切片     |
| -------- | -------- | -------- |
| 长度     | 固定     | 可变     |
| 内存分配 | 静态     | 动态     |
| 传递方式 | 值传递   | 引用传递 |
| 使用便捷性 | 较繁琐   | 较灵活   |

```go
package main

import "fmt"

func main() {
    s := []int{5}
    s = append(s, 7)
    s = append(s, 9)
    x := append(s, 11)
    y := append(s, 12)
    fmt.Println(s, x, y)
}
// output: [5 7 9] [5 7 9 12] [5 7 9 12]
```

## 切片扩容

在golang1.18版本之前

> 当原 slice 容量小于 1024 的时候，新 slice 容量变成原来的 2 倍；原 slice 容量超过 1024，新 slice 容量变成原来的1.25倍。

在1.18版本之后

> 当原slice容量(oldcap)小于256的时候，新slice(newcap)容量为原来的2倍；原slice容量超过256，新slice容量newcap = oldcap+(oldcap+3*256)/4

具体可参考[https://golang.design/go-questions/slice/grow/](https://golang.design/go-questions/slice/grow/)上面的说法其实不完全正确。



## 参考
[https://golang.design/go-questions/slice/vs-array/](https://golang.design/go-questions/slice/vs-array/)

[https://golang.design/go-questions/slice/grow/](https://golang.design/go-questions/slice/grow/)