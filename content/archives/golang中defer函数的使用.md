---
title: "Golang中defer函数的使用"
date: 2024-08-08T14:56:30+08:00
lastmod: 2024-08-08T14:56:30+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- Golang
- defer
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

二、参数求值与陷阱

```golang
func TestDeferArg(t *testing.T) {
i := 10
defer printValue(i * 10)

    i++
    fmt.Println("print_value before defer: ", i) // print_value before defer:  11
}

func printValue(v int) {
fmt.Println("print_value on defer: ", v) // print_value on defer:  100
}
```
解释
参数预计算：defer 语句定义时即计算并固定参数值。
具体来说，在把 defer 压入“栈”时，会同时压入函数地址和函数形参，也就是会在这个时候就把参数先算好。所以在执行到第 3 行代码的时候，就会把 i*10 算好，然后同 printValue 一同压入到延迟执行栈中。

三、环境变量捕获
代码示例
```go
func TestDeferNoArg(t *testing.T) {
i := 10
defer func() {
printValue(i * 10)
}()

    i++
    fmt.Println("print_value before defer: ", i) // print_value before defer:  11
}

func printValue(v int) {
fmt.Println("print_value in defer: ", v) // print_value in defer:  110
}
```
解释
这个时候其实没有参数，所以会直接将下面闭包压入延迟栈中。
func() {printValue(i * 10)}
而闭包是可以捕获环境变量的，所以在 TestDeferNoArg return 后，defer 可以捕获到 i 的值，为更新后的 i++，最后再进行 printValue(i * 10)。