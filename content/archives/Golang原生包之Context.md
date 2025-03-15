---
title: "Golang 原生包之Context"
date: 2025-02-24T17:07:57+08:00
lastmod: 2025-03-03T17:07:57+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- Golang
- Context
- OpenTrace
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

首先要说的是[context/context.go](https://golang.org/src/context/context.go)源码是一个非常值得学习的实现案例，非常建议大家对照着源码看文章。
其次关于context的功能简单点可以一言以蔽之：       
> **「context 用来解决 goroutine 之间退出通知、元数据传递的功能。」**

具体来说，`context` 提供了以下能力：
1. **退出通知**：通过 `context` 的取消机制，可以通知多个 goroutine 优雅退出。
2. **元数据传递**：通过 `context` 的 `WithValue` 方法，可以在 goroutine 之间安全地传递数据。
3. **超时控制**：通过 `WithTimeout` 和 `WithDeadline`，可以为操作设置超时时间。

## 四个常见方法
`context` 包提供了四个核心方法，用于创建和操作 `context`：
* `func WithCancel(parent Context) (ctx Context, cancel CancelFunc)`
* `func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)`
* `func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)`
* `func WithValue(parent Context, key, val interface{}) Context`

## 三个常见使用场景
### 传递共享的数据
通过 `WithValue`，可以在 goroutine 之间传递共享的数据。

```go
func main() {
	ctx := context.Background()
	process(ctx)

	ctx = context.WithValue(ctx, "traceId", "qcrao-2019")
	process(ctx)
}

func process(ctx context.Context) {
	traceId, ok := ctx.Value("traceId").(string)
	if ok {
		fmt.Printf("process over. trace_id=%s\n", traceId)
	} else {
		fmt.Printf("process over. no trace_id\n")
	}
}
```

### 取消 goroutine
通过 `WithCancel`、`WithDeadline` 或 `WithTimeout`，可以创建一个可取消的 `context`，并将其传递给多个 goroutine。当需要取消这些 goroutine 时，只需调用 `cancel` 函数即可。

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), time.Hour)
    go Perform(ctx)
    cancel()
}

func Perform(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            // 被取消，直接返回
            return
        default:
            doA()
            doB()
        }
    }
}
```

### 防止 goroutine 泄漏
`context` 的取消机制可以确保 goroutine 在不再需要时及时退出，从而避免 goroutine 泄漏。

```go
func gen(ctx context.Context) <-chan int {
	ch := make(chan int)
	go func() {
		var n int
		for {
			select {
			case <-ctx.Done():
				return
			case ch <- n:
				n++
				time.Sleep(time.Second)
			}
		}
	}()
	return ch
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel() // 避免其他地方忘记 cancel，且重复调用不影响

	for n := range gen(ctx) {
		fmt.Println(n)
		if n == 5 {
			cancel()
			break
		}
	}
	// ……
}
```

## Context.Value查找过程

通过 `WithValue` 函数，可以创建层层的 `valueCtx`，存储 `goroutine` 间可以共享的变量，最终形成这样一棵树：
![context_search_val.png](/images/Go/context_search_val.png)
和链表有点像，只是它的方向相反：`Context` 指向它的父节点，链表则指向下一个节点。       
取值的过程，实际上是一个递归查找的过程：
```go
func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)
}
```
它会顺着链路一直往上找，比较当前节点的 `key` 是否是要找的 `key`，如果是，则直接返回 `value`。否则，一直顺着 `context` 往前，最终找到根节点（一般是 `emptyCtx`），直接返回一个 `nil`。所以用 `Value` 方法的时候要判断结果是否为 `nil`。

因为查找方向是往上走的，所以，父节点没法获取子节点存储的值，子节点却可以获取父节点的值。

`WithValue` 创建 `context` 节点的过程实际上就是创建链表节点的过程。两个节点的 `key` 值是可以相等的，但它们是两个不同的 `context` 节点。查找的时候，会向上查找到最后一个挂载的 `context` 节点，也就是离得比较近的一个父节点 `context`。所以，整体上而言，用 `WithValue` 构造的其实是一个低效率的链表。

### 查找过程的坑
- **子 context 覆盖父 context**：如果在子 `context` 中设置了与父 `context` 相同的键，子 `context` 的值会覆盖父 `context` 的值。这种行为可能会导致意外的结果，因此在使用 `WithValue` 时需要特别注意键的唯一性。

## 源码解析
### Context 接口
`Context` 是一个`interface`，定义了四个方法：
```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```
- **`Deadline`**：返回 `context` 的截止时间。
- **`Done`**：返回一个只读channel，用于接收取消通知。
- **`Err`**：返回 `context` 被取消的原因。
- **`Value`**：返回与键关联的值。

包中以下结构体实现了此接口：
* **`emptyCtx`**：最简单的，无实现逻辑，基础`Context`结构体
* **`backgroundCtx`**：由 `Background()` 函数返回的
* **`todoCtx`**：由 `TODO()` 函数返回的
* **`afterFuncCtx`**：go 1.21 版本后引入`AfterFunc(ctx Context, f func()) (stop func() bool)` 用于定义 `Context` 被取消或超时后执行一个回调函数。（每个 `context` 的 `AfterFunc` 回调是独立的，父注册的函数不会作用到子）
* **`stopCtx`** 同上在`AfterFunc`函数内间接使用
* **`cancelCtx`**
* **`withoutCancelCtx`**
* **`timerCtx`**
* **`valueCtx`**

### cancelCtx
`cancelCtx` 是 `WithCancel` 返回的 `context` 类型，其核心是一个 `done` channel 和一个 `children` map。当调用 `cancel` 函数时，会关闭 `done` channel 并递归取消所有子 `context`。

### valueCtx
`valueCtx` 是 `WithValue` 返回的 `context` 类型，其核心是一个键值对和一个指向父 `context` 的指针。`Value` 方法会递归查找键值对。

### timerCtx
`timerCtx` 是 `WithDeadline` 和 `WithTimeout` 返回的 `context` 类型，它在 `cancelCtx` 的基础上增加了定时器功能。当定时器触发时，会自动调用 `cancel` 函数。


## OpenTracing-Go



## 参考
[context如何被取消](https://golang.design/go-questions/stdlib/context/cancel/)            
[OpenTracing](https://opentracing.io/docs/overview/)            
