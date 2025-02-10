---
title: "Golang Channel"
date: 2025-02-10T10:55:03+08:00
lastmod: 2025-02-10T10:55:03+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- Golang
- Channel
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

之前写过一篇文章 [《关于对「Don't communicate by sharing memory, share memory by communicating」的理解》](../关于-不要通过共享内存进行通信通过通信共享内存/) ，在里面介绍了我对Golang CSP设计理念中「通信」概念的理解。    
所以，与前者不同的是，本篇更注重Channel相关底层实现的探索。

## 数据结构
```go
type hchan struct {
    qcount   uint           // 当前队列中的元素数量
    dataqsiz uint           // chan 底层循环数组的长度
    buf      unsafe.Pointer // 指向底层循环数组的指针，只针对有缓冲的 channel
    elemsize uint16         // chan 中元素大小
    closed   uint32         // chan 是否已关闭
    elemtype *_type         // 元素的类型信息
    sendx    uint           // 发送索引
    recvx    uint           // 接收索引
    recvq    waitq          // 等待接收的 goroutine 队列
    sendq    waitq          // 等待发送的 goroutine 队列
    lock     mutex          // 互斥锁，保护 channel 的并发访问
}
```
**重点说明**：
* 对于非缓冲channel， `buf` 字段为 nil，`dataqsiz` 为 0。 发送和接收操作不会使用 `buf`，而是直接通过 `sendq` 和 `recvq` 队列进行同步。 当发送操作发生时，如果 `recvq` 队列中有等待的接收者，数据会直接从发送者拷贝到接收者，反之亦然。
* `sendx`，`recvx` 均指向底层循环数组，表示当前可以发送和接收的元素位置索引值（相对于底层数组）。
* `sendq`，`recvq` 分别表示被阻塞的 goroutine，这些 goroutine 由于尝试读取 channel 或向 channel 发送数据而被阻塞。
* `waitq` 是 `sudog` 的一个双向链表，而 `sudog` 实际上是对 goroutine 的一个封装
```go 
 type waitq struct {
  first *sudog
  last  *sudog
  }
```
* lock 用来保证每个读 channel 或写 channel 的操作都是原子的。     

创建一个容量为 6 的，元素为 int 型的 channel 数据结构如下 ：
![channel_A.png](/images/Go/channel_A.png)

## Channel的阻塞模式和非阻塞模式
### 阻塞模式
在阻塞模式下，通道的发送和接收操作会导致当前 `goroutine` 被阻塞，直到操作可以成功完成，例如
```
ch <- value  // 发送操作会阻塞，直到有空间可用
value := <-ch // 接收操作会阻塞，直到有数据可读
```

### 非阻塞模式
在非阻塞模式下，通道的发送和接收操作不会导致 `goroutine` 被阻塞，而是会立即返回一个结果。 例如使用 `select` 语句结合 `default` 分支来实现
```go
select {
case ch <- value: // 尝试发送值
    // 发送成功
default:
    // 通道已满，发送失败，不会阻塞
}

select {
case value := <-ch: // 尝试接收值
    // 接收成功
default:
    // 通道为空，接收失败，不会阻塞
}
```

## Channel数据操作
### 向Channel发送数据
发送过程涉及到go源码的 `chansend` 函数，位于 `src/runtime/chan.go` 文件。     
代码咱就不看了，来看看涉及到的主要流程。

1. 阻塞模式下，如果检测到 `channel` 是 nil ，当前 `goroutine` 会被挂起。
2. 非阻塞模式下，如果 `channel` 未关闭并且没有多余的缓冲空间，直接返回false。
3. 如果检测到 `channel` 已经关闭，直接 `panic`。
4. 如果能从等待接收队列 `recvq` 里出队一个 `sudog`（代表一个 `goroutine`），说明此时 `channel` 是空的，没有元素，所以才会有等待接收者。这时会调用 `send` 函数将元素直接从发送者的栈拷贝到接收者的栈（无缓冲chan，不用先拷贝到buf，直接由发送者到接收者，效率得以提高）
5. 如果 `c.qcount` < `c.dataqsiz`，说明缓冲区可用（肯定是缓冲型的 `channel`）。先通过函数取出待发送元素应该去到的位置。`c.sendx` 指向下一个待发送元素在循环数组中的位置，然后调用 `typedmemmove` 函数将其拷贝到循环数组中。之后 `c.sendx` 加 1，元素总量加 1 ：`c.qcount++`，最后，解锁并返回。
6. 如果没有命中以上条件的，说明 `channel` 已经满了。不管这个 `channel` 是缓冲型的还是非缓冲型的，都要将这个 `goroutine` 被阻塞。如果 非阻塞模式直接解锁，返回 false。
7. 阻塞模式下先构造一个 `sudog`，将其入队（`channel` 的 `sendq` 字段）。然后调用 `goparkunlock` 将当前 `goroutine` 挂起，并解锁，等待合适的时机再唤醒。 所以，待发送的元素地址其实是存储在 `sudog` 结构体里，也就是当前 `goroutine` 里。

### 向Channel接收数据
接收操作有两种写法，一种带 “ok”，反应 channel 是否关闭；一种不带 “ok”，这种写法，当接收到相应类型的零值时无法知道是真实的发送者发送过来的值，还是 channel 被关闭后，返回给接收者的默认类型的零值。两种写法，都有各自的应用场景。

接收过程对应的是在源码 `src/runtime/chan.go` 文件里的 `chanrecv` 函数。       
主要流程如下：
1. 如果 channel 是一个空值（nil），在非阻塞模式下，会直接返回。在阻塞模式下，会调用 gopark 函数挂起 goroutine，这个会一直阻塞下去。因为在 channel 是 nil 的情况下，要想不阻塞，只有关闭它，但关闭一个 nil 的 channel 又会发生 panic，所以没有机会被唤醒了。
2. 非阻塞模式下，如果 `channel` 未关闭**并且**（非缓冲型，等待发送列队里没有 goroutine 在等待**或者**缓冲型，但 buf 里没有元素）直接返回false
3. 接下来的，如果 channel 已关闭，并且循环数组 buf 里没有元素。对应非缓冲型关闭和缓冲型关闭但 buf 无元素的情况，返回对应类型的零值，但 received 标识是 false，告诉调用者此 channel 已关闭，你取出来的值并不是正常由发送者发送过来的数据。
4. 接下来，如果有等待发送的队列，说明 channel 已经满了，要么是非缓冲型的 channel，要么是缓冲型的 channel，但 buf 满了。这两种情况下都可以正常接收数据。
5. 对于缓冲型 channel，而 buf 又满了的情形。说明发送游标和接收游标重合了，因此需要先找到接收游标。将该处的元素拷贝到接收地址。然后将发送者待发送的数据拷贝到接收游标处。这样就完成了接收数据和发送数据的操作。
6. 然后，如果 channel 的 buf 里还有数据，说明可以比较正常地接收。注意，这里，即使是在 channel 已经关闭的情况下，也是可以走到这里的。这一步比较简单，正常地将 buf 里接收游标处的数据拷贝到接收数据的地址，函数return。
7. 到了最后一步，走到这里来的情形是要阻塞的。当然，如果 block 传进来的值是 false，那就不阻塞，直接返回就好了。

### Channel操作数据的本质
channel 的发送和接收操作本质上都是 “值的拷贝”，无论是从 sender goroutine 的栈到 chan buf，还是从 chan buf 到 receiver goroutine，或者是直接从 sender goroutine 到 receiver goroutine。

### Channel会引起资源泄漏吗？
有一种特殊情况，比如当 goroutine 操作 channel 后，处于发送或接收阻塞状态，而 channel 处于满或空的状态，一直得不到改变。同时，垃圾回收器也不会回收此类资源，进而导致 gouroutine 会一直处于等待队列中，进而引发资源泄漏。

### 总结
| 操作   | nil channel | closed channel | not nil, not closed channel                      |
|--------|-------------|----------------|-------------------------------------------------|
| close  | panic       | panic          | 正常关闭                                         |
| 读 <- ch | 阻塞 | 读到对应类型的零值     | 阻塞或正常读取数据。缓冲型 channel 为空或非缓冲型 channel 没有等待发送者时会阻塞 |
| 写 ch <- | 阻塞 | panic          | 阻塞或正常写入数据。非缓冲型 channel 没有等待接收者或缓冲型 channel buf 满时会被阻塞 |

发生 panic 的情况有三种：
* 向一个关闭的 channel 进行写操作
* 关闭一个 nil 的 channel
* 重复关闭一个 channel

## 关闭Channel

### 从关闭的Channel还能读出数据吗

### 如何优雅地关闭


## Channel应用


## 参考
[https://golang.design/go-questions/channel/send/](https://golang.design/go-questions/channel/send/)        
[https://golang.design/go-questions/channel/recv/](https://golang.design/go-questions/channel/recv/)
