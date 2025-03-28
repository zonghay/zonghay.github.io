---
title: "Go中的指针和二级指针"
date: 2024-11-28T14:16:19+08:00
lastmod: 2024-11-28T14:16:19+08:00
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

## Go语言指针基础
### 什么是指针
简单来说指针是存储另一个变量的内存地址的变量。
![](/images/Go/what_is_pointer.png)
在上图中，变量 b 的值被定义为252并存储在内存地址 0x1040a124 上。       
变量 a 保存 b 的地址，现在 a 指向 b。

### 声明指针
```go
package main

import (  
    "fmt"
)

func main() {  
    b := 252
    var a *int = &b
    fmt.Printf("a 的类型为：%T\n", a)
    fmt.Println("b的地址为：", a)
}
```
上述程序我们将 b 的地址分配给类型是 *int 的变量a。现在说 a 指向 b。当我们打印 a 中的值时，将会显示 b 的地址。
```
a 的类型为：*int
b的地址为： 0xc0000ac1a0
```

### 指针的零值
指针的零值是nil。          
注：在Go中，除了指针类型，nil 还可以代表其他几种类型的零值，包括切片（slice）、映射（map）、通道（channel）、接口（interface）和函数类型。 对于这些类型，nil 同样表示它们没有被赋予任何具体的值或实例。
```go
package main

import (
	"fmt"
)

func main() {
	var a *int // or var a = new(int)
	var b *[]int
	var c *int
	var d **int
	fmt.Printf("a 的类型为：%T\n", a)
	fmt.Printf("b 的类型为：%T\n", b)
	fmt.Printf("c 的类型为：%T\n", c)
	fmt.Printf("d 的类型为：%T\n", d)
	fmt.Println("a的值为：", a)
	fmt.Println("b的值为：", b)
	fmt.Println("c的值为：", c)
	fmt.Println("d的值为：", d)
	fmt.Println("a的地址为：", &a)
	fmt.Println("b的地址为：", &b)
	fmt.Println("c的地址为：", &c)
	fmt.Println("d的地址为：", &d)
}
```
```
a 的类型为：*int
b 的类型为：*[]int
c 的类型为：*int
d 的类型为：**int
a的值为： <nil>
b的值为： <nil>
c的值为： <nil>
d的值为： <nil>
a的地址为： 0xc00000e038
b的地址为： 0xc00000e040
c的地址为： 0xc00000e048
d的地址为： 0xc00000e050
```

### 访问指针变量
如何访问指针指向的变量值？需要在变量前面加上星号 *。 例如：*a 是访问a指向的变量值。
```go
package main  
import (  
    "fmt"
)

func main() {
	b := 252
	a := &b
	fmt.Println("b 的地址是：", a)
	fmt.Println("b 的值是：", *a)
	*a++
	fmt.Println("修改后 b 的地址是：", a)
	fmt.Println("修改后 b 的值是：", b)
}
```
```
b 的地址是： 0xc0000162e8
b 的值是： 252
修改后 b 的地址是： 0xc0000162e8
修改后 b 的值是： 253
```

### 指针作为函数的参数
```go
package main

import (  
    "fmt"
)

func change(val *int) {  
    *val = 55
}
func main() {  
    a := 58
    fmt.Println("变量 a 在调用函数之前：", a)
    b := &a
    change(b)
    fmt.Println("变量 a 在调用函数之后：", a)
}
```
在上面的程序中，我们将a 的地址保存到指针变量 b 中，并将b传递给函数change。在change函数内部，更改 a 的值。因为指针是指向的内存地址，因此函数内部的修改会反应到函数外部的变量中。
```
变量 a 在调用函数之前： 58
变量 a 在调用函数之后： 55
```

### 不支持指针运算
Go 不支持其他语言（如 C 和 C++）中存在的指针运算。
```
package main

func main() {  
    b := [...]int{109, 110, 111}
    p := &b
    p++
}
```
上面的程序会抛出编译错误

### 二级指针
如果一个指针变量存放的又是另一个指针变量的地址，则称这个指针变量为指向指针的指针变量。这就是我们所说的二级指针。        
![](/images/Go/pointer_to_pointer.png)

```go
package main
import (
	"fmt"
)

func main() {
	var a int
	var ptr *int
	var pptr **int

	a = 3000
	/* 指针 ptr 地址 */
	ptr = &a
	/* 指向指针 ptr 地址 */
	pptr = &ptr

	/* 获取 pptr 的值 */
	fmt.Println("变量 a = ", a)
	fmt.Println("变量 a 地址 = ", &a)
	fmt.Println("指针 ptr = ", ptr)
	fmt.Println("指针变量 ptr 地址 = ", &ptr)
	fmt.Println("指针变量 *ptr = ", *ptr)
	fmt.Println("指针 pptr = ", pptr)
	fmt.Println("指针变量 pptr 地址 = ", &pptr)
	fmt.Println("指向指针的指针变量 **pptr = ", **pptr)
}
```
```
变量 a =  3000
变量 a 地址 =  0xc0000162e8
指针 ptr =  0xc0000162e8
指针变量 ptr 地址 =  0xc00000e038
指针变量 *ptr =  3000
指针 pptr =  0xc00000e038
指针变量 pptr 地址 =  0xc00000e040
指向指针的指针变量 **pptr =  3000
```


## 实战
### 《合并两个有序链表》
下面我们使用二级指针来实现LeetCode上的[《合并两个有序链表》](https://leetcode.cn/problems/merge-two-sorted-lists/description/)       
通过这道题帮助我们深刻立即：
* **函数要改变指针，参数就要是指针的指针**
* **函数要改变指针指向的内容，参数只需要指针**

```go
func MergeTwoLists(list1 *ListNode, list2 *ListNode) *ListNode {
	var head, current *ListNode
	for list1 != nil && list2 != nil {
		if list2.Val >= list1.Val {
			AppendList(&head, &current, list1.Val)
			list1 = list1.Next
		} else {
			AppendList(&head, &current, list2.Val)
			list2 = list2.Next
		}
	}
	if list1 != nil {
		current.Next = list1
	} else if list2 != nil {
		current.Next = list2
	}
	return head
}

// AppendList 函数用于创建链表，使用二级指针对链表进行修改
func AppendList(headPointer, currentPointer **ListNode, val int) {
	newNode := &ListNode{Val: val}
	if *headPointer == nil {
		*headPointer = newNode
	} else {
		(*currentPointer).Next = newNode
	}
	*currentPointer = newNode
}
```

### Golang源码中的二级指针

Golang Map源码的实现中使用二级指针来存放bucket中的key和value值             
详情请阅读[GO源码分析-map](https://www.yinlijun.com/2023/02/14/go-map/)

## unsafe.Pointer

通过上述内容我们已知，Go指针无法进行数学运算、比较以及不同类型指针间不能互转和赋值。     
unsafe.Pointer提供了一个重要的能力：任何类型的指针和 unsafe.Pointer 可以相互转换。     

一个实际应用就是使用unsafe.Pointer实现**零拷贝**字符串和bytes之间的转换。
```go
func string2bytes(s string) []byte {
	return *(*[]byte)(unsafe.Pointer(&s))
}
func bytes2string(b []byte) string{
	return *(*string)(unsafe.Pointer(&b))
}
```
原理上是利用指针的强转。

## 参考
[Go 语言指针详解](https://www.jiyik.com/w/go/go-pointers)         
[Go 语言多级指针 - 指向指针的指针](https://www.jiyik.com/w/go/go-pointer-to-pointer)         
[Go 语言指针与函数](https://www.jiyik.com/w/go/go-pointer-func)          
[GO源码分析-map](https://www.yinlijun.com/2023/02/14/go-map/)               
[为什么要用指针，什么时候该用指针，什么时候该用指针的指针](https://blog.csdn.net/qq_39392646/article/details/126195446)
