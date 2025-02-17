---
title: "Golang 数组与切片"
date: 2025-01-19T15:19:07+08:00
lastmod: 2025-01-19T15:19:07+08:00
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
    y := append(s, 12) // y := append(s,12)。这里s的容量还是4，长度3，所以append 12会写入到第四个位置，覆盖原来的11，变为12。此时y的底层数组是[5,7,9,12]，长度4。而x的底层数组也被修改了，因为x和y共享同一个底层数组。所以x的值现在也是[5,7,9,12]。
    fmt.Println(s, x, y)
}
// output: [5 7 9] [5 7 9 12] [5 7 9 12]
```
面对这类问题，始终记住：
* 切片底层指向一块儿数组内存，扩容或对数组的修改都会影响所有指向这块地址的切片。
* 时刻关注切片的**容量**和**长度**
  * 长度（len）决定了可见元素的范围。
  * 容量（cap）决定了是否触发扩容。

## 切片扩容

在golang1.18版本之前

> 当原 slice 容量小于 1024 的时候，新 slice 容量变成原来的 2 倍；原 slice 容量超过 1024，新 slice 容量变成原来的1.25倍。

在1.18版本之后

> 当原slice容量(oldcap)小于256的时候，新slice(newcap)容量为原来的2倍；原slice容量超过256，新slice容量newcap = oldcap+(oldcap+3*256)/4

具体可参考[https://golang.design/go-questions/slice/grow/](https://golang.design/go-questions/slice/grow/)上面的说法其实不完全正确。

## nil切片和空切片
- **nil 切片**：
    - 未初始化的切片，值为 `nil`。
    - 长度和容量都为 0。
    - 示例：
      ```
      var nilSlice []int
      fmt.Println(nilSlice == nil) // 输出: true
      fmt.Println(len(nilSlice))   // 输出: 0
      fmt.Println(cap(nilSlice))   // 输出: 0
      ```

- **空切片**：
    - 已初始化的切片，但没有任何元素。
    - 长度和容量都为 0。
    - 示例：
      ```
      emptySlice := []int{}
      fmt.Println(emptySlice == nil) // 输出: false
      fmt.Println(len(emptySlice))   // 输出: 0
      fmt.Println(cap(emptySlice))   // 输出: 0
      ```


## 参考
[https://golang.design/go-questions/slice/vs-array/](https://golang.design/go-questions/slice/vs-array/)

[https://golang.design/go-questions/slice/grow/](https://golang.design/go-questions/slice/grow/)