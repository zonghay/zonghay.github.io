---
title: "Golang中defer函数的使用"
date: 2022-08-08T14:56:30+08:00
lastmod: 2023-08-08T14:56:30+08:00
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
## 基本概念
defer 是 Go 语言中的一个关键字，用于延迟（推迟）一个函数或方法的执行，直到包含它的函数执行完毕时才会被调用，无论包含它的函数是通过 return 正常结束还是由于 panic 导致的异常结束。           
defer 语句通常用于资源的释放、解锁以及确保某些关键操作的完成。


## 参数求值与陷阱
在 Go 中，defer 语句中的函数参数会在 defer 语句执行时立即求值。这意味着，无论 defer 语句之后的代码如何改变变量的值，defer 调用的函数将使用 defer 语句执行时的参数值。
```go
package main

import "fmt"

func printValue(v int) {
    fmt.Println("print_value on defer: ", v)
}

func TestDeferArg() {
    i := 10
    fmt.Println("print_value before defer: ", i) // 输出: print_value before defer: 10
    i++
    defer printValue(i * 10) // 参数在此时求值，即 11 * 10 = 110
    i++
    fmt.Println("print_value after defer: ", i) // 输出: print_value after defer: 12
}

func main() {
    TestDeferArg()
    // 输出: print_value on defer: 110
}
```

### 解释
参数预计算：defer 语句定义时即计算并固定参数值。     
具体来说，在把 defer 压入“栈”时，会同时压入函数地址和函数形参，也就是会在这个时候就把参数先算好。所以在执行到第 3 行代码的时候，就会把 i*10 算好，然后同 printValue 一同压入到延迟执行栈中。

## 环境变量捕获
当 defer 语句后面跟的是一个匿名函数时，这个匿名函数可以捕获其外部作用域中的变量。这意味着，即使外部函数的执行已经结束，匿名函数仍然可以访问和操作这些变量。
```go
package main

import "fmt"

func printValue(v int) {
    fmt.Println("print_value in defer: ", v)
}

func TestDeferNoArg() {
    i := 10
    fmt.Println("print_value before defer: ", i) // 输出: print_value before defer: 10
    i++
    defer func() {
        printValue(i * 10) // 匿名函数捕获了 i 的当前值
    }()
    i++
    fmt.Println("print_value after defer: ", i) // 输出: print_value after defer: 12
}

func main() {
    TestDeferNoArg()
    // 输出: print_value in defer: 120
}
```

### 解释
这个时候其实没有参数，所以会直接将下面闭包压入延迟栈中。
而闭包是可以捕获环境变量的，所以在 TestDeferNoArg return 后，defer 可以捕获到 i 的值，为更新后的 i++，最后再进行 printValue(i * 10)。

## 使用场景
defer 语句在 Go 语言中有许多实际应用场景，例如：
* 资源释放：确保文件、数据库连接等资源在使用后被正确关闭。
* 解锁互斥锁：在并发编程中，确保互斥锁在函数退出前被释放。
* 错误处理：在函数执行过程中，如果发生错误，可以使用 defer 来执行一些清理工作。

###  注意事项
* defer 语句的执行顺序是后进先出（LIFO），即最后一个 defer 语句会最先执行。
* defer 语句不能保证立即执行，它会在包含它的函数即将返回之前执行。
* 过度使用 defer 可能会导致性能问题，因为每个 defer 语句都会增加程序的调用栈深度。
