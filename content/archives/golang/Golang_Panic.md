---
title: "Golang中Panic的捕获和处理机制"
date: 2025-06-04T11:40:05+08:00
lastmod: 2025-06-04T11:40:05+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- Golang
- Panic
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

golang生态的错误处理机制和常见的面向对象语言不尽相同。通常情况下，我们可以将`panic`和`recover`机制结合起来处理程序运行期间发生的异常错误。            
不过这也不是万能的，在语言层面有一些致命的错误是无法被recover捕获的。      
本篇文章将会介绍常见的可恢复和不可恢复两大类Panic。        

## 可恢复Panic
### 显式触发的Panic
```go
func safeOperation() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered:", r)
        }
    }()
    panic("manual panic trigger")
}
```

### 运行时错误
#### 空指针解引用
```go
var ptr *int
fmt.Println(*ptr) // runtime error: invalid memory address or nil pointer dereference
```

#### 数组/切片越界
```go
arr := []int{1}
fmt.Println(arr[10]) // panic: runtime error: ndex out of range [10] with length 1
```

#### 类型断言失败
```go
var i interface{} = "string"
_ = i.(int) // panic: interface conversion: interface {} is string, not int
```

#### 关闭已关闭的channel
```go
ch := make(chan int)
close(ch)
close(ch) // panic: close of closed channel
```


## 不可恢复的致命错误
### 并发读写Map
```go
func TestPanic(t *testing.T) {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("Recovered:", r)
		}
	}()
	m := make(map[int]int)
	go func() {
		for {
			m[1] = 1
		}
	}()
	go func() {
		for {
			_ = m[1]
		}
	}()
    select {}
}
// fatal error: concurrent map read and map write
```

### 栈内存耗尽
```go
func TestPanic(t *testing.T) {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("Recovered:", r)
		}
	}()
	var infiniteRecursion func()
	infiniteRecursion = func() {
		infiniteRecursion()
	}
	infiniteRecursion()
}
// runtime: goroutine stack exceeds 1000000000-byte limit
// runtime: sp=0x140201603a0 stack=[0x14020160000, 0x14040160000]
// fatal error: stack overflow
```

### 所有 Goroutine 死锁
```go
func TestPanic(t *testing.T) {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("Recovered:", r)
		}
	}()
	var wg sync.WaitGroup
	ch := make(chan int) // 无缓冲通道

	wg.Add(1)
	go func() {
		defer wg.Done()
		<-ch // 阻塞等待数据
	}()

	wg.Wait() // 主goroutine等待
	ch <- 1   // 永远执行不到这里
}
// fatal error: all goroutines are asleep - deadlock!
```

### 内存分配失败(OOM)

### 程序主动退出
```go
func TestPanic(t *testing.T) {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("Recovered:", r)
		}
	}()
	os.Exit(1) 
}
// 立即终止，不触发 recover
```

## Echo框架的Recover中间件
实现原理其实很简单，就是在每个处理http请求的goroutine中使用recover函数handle期间的panic，并进行自定义的日志输出和http response结果返回。
```go
// RecoverWithConfig returns a Recover middleware with config.
// See: `Recover()`.
func RecoverWithConfig(config RecoverConfig) echo.MiddlewareFunc {
	// Defaults
	if config.Skipper == nil {
		config.Skipper = DefaultRecoverConfig.Skipper
	}
	if config.StackSize == 0 {
		config.StackSize = DefaultRecoverConfig.StackSize
	}

	return func(next echo.HandlerFunc) echo.HandlerFunc {
		return func(c echo.Context) error {
			if config.Skipper(c) {
				return next(c)
			}

			defer func() {
				if r := recover(); r != nil {
					err, ok := r.(error)
					if !ok {
						err = fmt.Errorf("%v", r)
					}
					stack := make([]byte, config.StackSize)
					length := runtime.Stack(stack, !config.DisableStackAll)
					if !config.DisablePrintStack {
						c.Logger().Printf("[PANIC RECOVER] %v %s\n", err, stack[:length])
					}
					c.Error(err)
				}
			}()
			return next(c)
		}
	}
}
```
但有一点需要注意的是：中间件仅能捕获当前HTTP请求的Goroutine中可恢复的panic，后台Goroutine的panic或致命错误无法被中间件处理。
