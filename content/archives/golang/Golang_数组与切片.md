---
title: "Golang 数组与切片"
date: 2023-01-19T15:19:07+08:00
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
	
	// 示例2
    src := []int{1, 2, 3}
    _ = append(src[:0], src[1:]...)
	println(src)
    // [2, 2, 3]
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

## copy函数
在 Go 语言中，`copy` 函数用于将一个切片的内容复制到另一个切片中，从而创建一个新的切片，避免共享底层数组。          
在同一个切片中进行部分复制时，copy 是安全的。

### copy的行为
- **复制规则**：
    - 从源切片 `src` 复制元素到目标切片 `dst`。
    - 复制的元素数量是 `dst` 和 `src` 长度的较小值。
    - 不会改变目标切片 `dst` 的长度和容量。
- **注意事项**：
    - 如果 `dst` 或 `src` 是 `nil`，`copy` 会返回 0（不执行任何操作）。
    - 如果 `dst` 和 `src` 是同一个切片，`copy` 会正确处理重叠的情况。

### 示例代码
```go
package main

import "fmt"

func main() {
    // 示例 1: 普通复制
    src := []int{1, 2, 3, 4, 5}
    dst := make([]int, 3) // 目标切片长度为 3
    n := copy(dst, src)   // 复制 3 个元素
    fmt.Println("Copied elements:", n) // 3
    fmt.Println("dst:", dst)          // [1 2 3]

    // 示例 2: 目标切片长度大于源切片
    src = []int{10, 20}
    dst = make([]int, 5) // 目标切片长度为 5
    n = copy(dst, src)   // 复制 2 个元素
    fmt.Println("Copied elements:", n) // 2
    fmt.Println("dst:", dst)          // [10 20 0 0 0]

    // 示例 3: 目标切片长度小于源切片
    src = []int{100, 200, 300}
    dst = make([]int, 2) // 目标切片长度为 2
    n = copy(dst, src)   // 复制 2 个元素
    fmt.Println("Copied elements:", n) // 2
    fmt.Println("dst:", dst)          // [100 200]

    // 示例 4: 目标切片和源切片是同一个切片
    src = []int{1, 2, 3, 4, 5}
    n = copy(src[2:], src) // 重叠复制
    fmt.Println("Copied elements:", n) // 3
    fmt.Println("src:", src)          // [1 2 1 2 3]
}
```

## sort.Slice踩坑

```go
// nums=[2,3,1]
var tmp = nums[1:]
sort.Slice(tmp, func(x, y int) bool {
	return tmp[x] < tmp[y]
})
```

```go
// nums=[2,3,1]
sort.Slice(nums[1:], func(x, y int) bool {
	return nums[x] < nums[y]
})
```

上面的两段代码基本一致，但第一段运行后nums数组依然是```[2,3,1]```，第二段代码符合预期，执行后nums为```[2,1,3]```，想一想是为什么？



## 参考
[https://golang.design/go-questions/slice/vs-array/](https://golang.design/go-questions/slice/vs-array/)

[https://golang.design/go-questions/slice/grow/](https://golang.design/go-questions/slice/grow/)