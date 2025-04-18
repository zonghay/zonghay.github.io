---
title: "Golang GMP模型"
date: 2025-03-28T11:15:39+08:00
lastmod: 2025-03-31T11:15:39+08:00
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
hasMermaid: true
cover:
    image: "" #图片路径例如：posts/tech/123/123.png
    zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

# 什么是GMP
![GMP_overview.png](/images/Go/GMP_overview.png)

## G（Goroutine）
G 是调度的基本单位，由 Go 运行时管理，比 OS 线程更轻量代表goroutine，主要保存状态信息以及寄存器的值。           
当 goroutine 被调离 CPU 时，调度器负责把 CPU 寄存器的值保存在 g 对象的成员变量之中。
当 goroutine 被调度起来运行时，调度器又负责把 g 对象的成员变量所保存的寄存器值恢复到 CPU 的寄存器。

### Goroutine和线程的区别
* 内存开销
* 切换(调度)开销
* 运行环境

## M（Machine）
M代表一个工作线程或者系统线程，是真正工作的单位。M中保存自身使用的栈信息、当前正在M上执行的G信息与之绑定的P信息。

### M:N模型
M 个 goroutines (G) 被调度到 N 个内核线程 (M) 上执行，这些线程运行在最多 GOMAXPROCS 个逻辑处理器 (P) 上。      

P 负责管理 G 的本地运行队列（LRQ），每个 P 必须绑定一个 M 才能执行 G。如果某个 M 阻塞（如系统调用），与该 M 绑定的 P 会解绑，并由其他空闲的 M（或新建的 M）接管这个 P，继续执行其 LRQ 中的 G。

### M阻塞和G阻塞的区别
| 维度    | M 阻塞（线程级）| G 阻塞（协程级）  |
|-------|----------------|-----------------|
| 触发原因	 |系统调用、CGO 等底层操作	|Channel、锁、Gosched() 等用户态操作|
| 阻塞对象	 |内核线程（M）	|Goroutine（G）|
| 调度影响	 |P 与 M 解绑，可能创建新 M	|G 被移出运行队列，M 执行其他 G|
| 恢复机制	 |系统调用完成后重新绑定 P	|条件满足后重新加入运行队列|
| 性能代价	 |高（线程切换、内核态开销）	|低（用户态调度，无线程切换）|

### M的自旋状态
自旋（Spinning） 是指 M 在暂时没有可执行的 G（Goroutine）时，不立即进入休眠，而是空转循环（忙等待），主动寻找可运行的 G。
自旋的 M 会占用 CPU 资源，但避免了线程休眠和重新唤醒的代价（系统调用、上下文切换等）。

一个 M 进入自旋状态通常需要满足以下条件：
* 当前 M 绑定的 P 的本地运行队列（LRQ）为空，且无法立即从全局队列（GRQ）或其他 P 偷取到 G。
* 仍有其他 P 正在运行 G（即系统整体有任务可执行，只是当前 P 暂时无任务）。
* 自旋的 M 数量未超过阈值（默认最多 GOMAXPROCS/2 个自旋 M，防止过度占用 CPU）。

自旋的 M 会持续执行以下操作：
* 检查全局队列（GRQ）：
* 定期扫描 GRQ，看是否有新加入的 G 可执行。
* 尝试从其他 P 偷取 G（Work Stealing）：
* 随机选择其他 P，从其 LRQ 中偷取一半的 G（最少偷 1 个）。 
* 检查网络轮询器（Netpoller）： 查看是否有就绪的网络 I/O 相关的 G 可恢复执行。 
* 自旋超时后休眠： 如果自旋一段时间（约 10ms）仍找不到 G，M 会退出自旋状态并休眠。

## P（Processor）
* 本地可运行队列（LRQ）: 存储本地（也就是具体的 P）的可运行 G
* 全局可运行队列（GRQ）: 存储全局的可运行 G，这些 G 还没有分配到具体的 P

为什么需要 P 这个组件，直接把 G 放到 M 不行吗？
当一个线程阻塞的时候，将和它绑定的 P 上的 G 转移到其他线程。Go scheduler 会启动一个后台线程 sysmon，用来检测长时间（超过 10 ms）运行的 goroutine，将其调度到 global runqueues。这是一个全局的 runqueue，优先级比较低，以示惩罚。

### G、M、P的数量关系
P（逻辑处理器）的数量（X）：
由 GOMAXPROCS 环境变量或 runtime.GOMAXPROCS() 设置，默认值为当前 CPU 核数，例如4核机器默认 X = 4。

M（内核线程）的数量（Y）：
由运行时动态管理，初始时通常 Y = X（每个 P 绑定一个 M），当发生系统调用阻塞或需要更多线程时，Y 可能增长（如阻塞的 M 不释放，运行时创建新 M）。理论上 Y >= X，但一般不会无限增长（有上限限制，如 runtime/debug.SetMaxThreads）。

G（Goroutine）的数量（Z）：
用户代码动态创建，理论上 Z ≫ X 且 Z ≫ Y。 例如，一个服务器程序可能同时处理数万个 Goroutines，但 X 和 Y 可能仅为几十。

# 什么是调度器Scheduler

![go_scheduler.png](/images/Go/go_scheduler.png)
Go scheduler 是 Go runtime 的一部分，它内嵌在 Go 程序里，和 Go 程序一起运行。因此它运行在用户空间，在 kernel 的上一层。

Runtime 起始时会启动一些 G：垃圾回收的 G，执行调度的 G，运行用户代码的 G；并且会创建一个 M 用来开始 G 的运行。随着时间的推移，更多的 G 会被创建出来，更多的 M 也会被创建出来。

Go scheduler 的核心思想是：
1. reuse threads；
2. 限制同时运行（不包含阻塞）的线程数为 N，N 等于 CPU 的核心数目；
3. 线程私有的 runqueues，并且可以从其他线程 stealing goroutine 来运行，线程阻塞后，可以将 runqueues 传递给其他线程。

## 调度器的初始化

![mermaid](https://kroki.io/mermaid/svg/eNorTi0sTc1LTnXJTEwvSszlUgCCgsSikszkzILEvBIF_2AMIV8DDCF3TCHfxMw8dy6wsH-wrp2dr4GVwtOO2U9373qyY_fzXfufr-h-v6fD1-D9nk6wIl8DoCJ3uKIXG5qf7lr2dOaKJzsmP5-y4vmsFpgl7gZw0-Y-Xd79tGeaQsDjhkb3oEAg6RMUqPB8bSeSQpAzYIYq5AJ5Cun5RfmlJZl5qUD7n--e-HTdLIWi0rySzNxUPZA83EEgnVCrnk3Z97R1qUKAAsgdQDtQXfxi_4QXC3ugLt636nnfeqDJxckZqSmlOakw45Bd82R3H8gUhRd925_2T0NzFablncuBxgMD4VnH9ie7Fz9f0Ag0_umyJrA-DU2QBVwAd3K_yQ==)

### M0
第一个操作系统线程（主线程），负责运行 Go 程序的初始逻辑。           
M0 是唯一一个由操作系统直接创建的线程，直接使用系统分配的栈空间（而非 Go 管理的轻量栈），后续的 M 均由 Go 运行时动态管理。

MO初始化的主要工作：
* 设置 GOMAXPROCS，创建 GOMAXPROCS 个 P。 初始化全局队列（GRQ）、内存分配器等。
* 生成一个普通的 G，绑定 runtime.main 函数（即用户程序的入口）。 将该 G 放入某个 P 的本地队列（LRQ）。 
* M0 开始执行调度逻辑，从 P 的 LRQ 中获取 main goroutine 并运行。

### G0
为 M0 分配一个特殊的 G0（调度器的“管家” Goroutine），栈直接复用 M0 的系统栈（大小约 8KB，普通 Goroutine 初始栈为 2KB）。       
G0 不执行用户代码，仅用于 调度器的栈切换（如创建新 G 时切换栈）、执行垃圾回收（GC）等运行时任务。

G0的核心工作：
* 当创建新 Goroutine 时，G0 负责从堆中分配栈内存并切换上下文。 当 Goroutine 退出时，G0 负责回收栈空间。
* 在 runtime.schedule() 中，G0 决定下一个要执行的 G。
* GC 的标记和清理阶段由 G0 协调执行。

## 调度Goroutine的时机
GMP模型的阻塞调度可能发生在下面几种情况：
* block on syscall
* I/O
* channel
* mutex
* runtime.Gosched主动让出

用户态阻塞调度G，系统调用阻塞调度M。

## 工作窃取
Go scheduler 的职责就是将所有处于 runnable 的 goroutines 均匀分布到在 P 上运行的 M 并执行它。
![GMP_relationship.png](/images/Go/GMP_relationship.png)

当一个 P 发现自己的 LRQ 已经没有 G 时，会从其他 P “偷” 一些 G 来运行。
![GMP_steal.png](/images/Go/GMP_steal.png)

当 P2 上的一个 G 执行结束，它就会去 LRQ 获取下一个 G 来执行。如果 LRQ 已经空了，就是说本地可运行队列已经没有 G 需要执行，并且这时 GRQ 也没有 G 了。这时，P2 会随机选择一个 P（称为 P1），P2 会从 P1 的 LRQ “偷”过来一半的 G。
（如果总是优先偷取其他 P 的 G，可能导致全局队列的 G 长时间得不到执行。）

## 调度流程
![GMP_schduler_flow.png](/images/Go/GMP_schduler_flow.png)
1. 创建的G放置流程是本地P->全局P
2. M获取P执行的流程是本地P->全局P->随机P
3. 当G因系统调用(syscall)阻塞时会阻塞M，此时P会和M解绑即hand off，并寻找新的idle的M，若没有idle的M就会新建一个M。当M唤醒后，优先绑定原来的 P（如果它仍然空闲）。如果原 P 已被占用，则尝试从全局的 P 空闲列表 中获取一个新的 P。 如果所有 P 均被占用，则： G 会被标记为 可运行状态（_Grunnable），并放入 全局运行队列（GRQ）。M 进入休眠状态，等待后续被调度器唤醒。
4. 当G因channel或者network I/O阻塞时，不会阻塞M，M会寻找其他runnable的G；当阻塞的G恢复后会重新放入某个 P 的 LRQ（不一定是原来的 P）

## 协作式调度和抢占式调度
协作式调度一般会由用户设置调度点，例如 python 中的 yield 语法。           
但是由于在 Go 语言里，goroutine 调度的事情是由 Go runtime 来做，并非由用户控制，所以我们依然可以将 Go scheduler 看成是抢占式调度，因为用户无法预测调度器下一步的动作是什么。

## 堆栈关系

| **维度**         | **栈（Goroutine Stack）**               | **堆（Heap）**                     |
|------------------|----------------------------------------|------------------------------------|
| **抽象角色**     | 函数调用的临时空间                     | 动态内存池                         |
| **物理存储**     | 位于进程的堆区（由 Go 运行时从堆分配）  | 进程的全局内存区域                 |
| **管理方**       | Go 运行时（动态扩容/收缩）             | Go 垃圾回收器（GC）               |
| **生命周期**     | 随 Goroutine 创建/销毁                 | 由 GC 决定回收时机                |

# 参考
[GMP 原理与调度](https://www.topgoer.com/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/GMP%E5%8E%9F%E7%90%86%E4%B8%8E%E8%B0%83%E5%BA%A6.html#gmp-%E5%8E%9F%E7%90%86%E4%B8%8E%E8%B0%83%E5%BA%A6)           
[GMP模型](https://go.cyub.vip/gmp/gmp-model/)         
[调度器](https://draven.co/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/)