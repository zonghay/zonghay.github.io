---
title: "Golang 底层实现之map"
date: 2025-01-20T15:30:17+08:00
lastmod: 2025-01-20T15:30:17+08:00
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

## Map的实现原理

golang 的 map 底层是哈希表查找，平均查找效率是 O(1)，最差是 O(N)

### go源码
在源码中，表示 map 的结构体是 hmap，它是 hashmap 的“缩写”：
```go
// A header for a Go map.
type hmap struct {
	count     int // 元素个数，调用 len(map) 时，直接返回此值
	flags     uint8
	B         uint8  // buckets 的对数 log_2
	noverflow uint16 // overflow 的 bucket 近似数，做扩容决策
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // 指向 buckets 数组，大小为 2^B，如果元素个数为0，就为 nil
	oldbuckets unsafe.Pointer // 指向旧桶数组的指针（用于扩容时的临时存储）
	nevacuate  uintptr        // 指示扩容进度，小于此地址的 buckets 迁移完成

	extra *mapextra // optional fields
}
```
mapextra 结构体包含了与哈希表相关的额外信息，其定义如下：
```go
type mapextra struct {
    overflow     *[]*bmap // 溢出桶的指针数组
    oldoverflow  *[]*bmap // 旧溢出桶的指针数组（用于扩容）
    nextOverflow *bmap     // 下一个要分配的溢出桶
}
```
buckets 是一个指针，最终它指向的是一个结构体：
```go
type bmap struct {
    topbits  [8]uint8    // 存储每个键的哈希值的高 8 位
    keys     [8]keytype  // 存储 8 个键
    values   [8]valuetype // 存储 8 个值
    pad      uintptr     // 填充字段，用于内存对齐
    overflow uintptr     // 指向下一个溢出桶的指针
}
```
* topbits [8]uint8 存储每个键的哈希值的高 8 位（称为 tophash），用于快速判断某个键是否可能存在于当前桶中。如果 topbits[i] 与某个键的哈希值高 8 位匹配，则进一步检查键是否相等。
* 键和值分开存储，以减少内存对齐带来的填充（padding），节省内存。
* 当一个桶中的键值对数量超过 8 时，会通过 overflow 指针链接到一个新的桶，形成链表结构。

![hmap.png](/images/Go/hmap.png)

### 创建Map
从语法层面上来说，创建 map 很简单：
```go
ageMp := make(map[string]int)
// 指定 map 长度
ageMp := make(map[string]int, 8)

// ageMp 为 nil，不能向其添加元素，会直接panic
var ageMp map[string]int
```
实际上底层调用的是 makemap 函数，主要做的工作就是初始化 hmap 结构体的各种字段，例如计算 B 的大小，设置哈希种子 hash0 等等。
```go
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap {
	// 省略各种条件检查...

	// 找到一个 B，使得 map 的装载因子在正常范围内
	B := uint8(0)
	for ; overLoadFactor(hint, B); B++ {
	}

	// 初始化 hash table
	// 如果 B 等于 0，那么 buckets 就会在赋值的时候再分配
	// 如果长度比较大，分配内存会花费长一点
	buckets := bucket
	var extra *mapextra
	if B != 0 {
		var nextOverflow *bmap
		buckets, nextOverflow = makeBucketArray(t, B)
		if nextOverflow != nil {
			extra = new(mapextra)
			extra.nextOverflow = nextOverflow
		}
	}

	// 初始化 hamp
	if h == nil {
		h = (*hmap)(newobject(t.hmap))
	}
	h.count = 0
	h.B = B
	h.extra = extra
	h.flags = 0
	h.hash0 = fastrand() // hash0 是一个随机值，用于哈希计算。每次初始化 map 时，hash0 都会不同，以防止哈希碰撞攻击。
	h.buckets = buckets
	h.oldbuckets = nil
	h.nevacuate = 0
	h.noverflow = 0

	return h
}
```
注意，这个函数返回的结果：*hmap，它是一个指针。

### key定位
不同类型 key 根据对应的哈希函数以及 hmap.hash0 计算哈希值 hash
hash 低 B 位确定桶索引 ```bucketIndex := hash & (1<<B - 1)```
hash 高 8 位找到此 key 在 bucket 中的位置，这是在寻找已有的 key。最开始桶内还没有 key，新加入的 key 会找到第一个空位，放入。

![hmap_key_search.png](/images/Go/hmap_key_search.png)

再说一下 minTopHash，当一个 cell 的 tophash 值小于 minTopHash 时，标志这个 cell 的迁移状态。因为这个状态值是放在 tophash 数组里，为了和正常的哈希值区分开，会给 key 计算出来的哈希值一个增量：minTopHash。这样就能区分正常的 top hash 值和表示状态的哈希值。

```go
// 空的 cell，也是初始时 bucket 的状态
empty          = 0
// 空的 cell，表示 cell 已经被迁移到新的 bucket
evacuatedEmpty = 1
// key,value 已经搬迁完毕，但是 key 都在新 bucket 前半部分，
// 后面扩容部分会再讲到。
evacuatedX     = 2
// 同上，key 在后半部分
evacuatedY     = 3
// tophash 的最小正常值
minTopHash     = 4
```
### 两种get操作
```go
// src/runtime/hashmap.go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool)
```

## Map赋值/更新

实际上插入或修改 key 的语法是一样的，只不过前者操作的 key 在 map 中不存在，而后者操作的 key 存在 map 中。

mapassign 有一个系列的函数，根据 key 类型的不同，编译器会将其优化为相应的“快速函数”。

| key | 类型 |
| :----: | :----: | 
| uint32 | mapassign_fast32(t *maptype, h *hmap, key uint32) |
| uint64 | mapassign_fast64(t *maptype, h *hmap, key uint64) |
| string | mapassign_faststr(t *maptype, h *hmap, ky string) |

源码大体和之前讲的类似，核心还是一个双层循环，外层遍历 bucket 和它的 overflow bucket，内层遍历整个 bucket 的各个 cell。

针对这个过程提几点重要的细节：
* 函数首先会检查 map 的标志位 flags。如果 flags 的写标志位此时被置 1 了，说明有其他协程在执行“写”操作，进而导致程序 panic。
* 我们知道扩容是渐进式的，如果 map 处在扩容的过程中，那么当 key 定位到了某个 bucket 后，需要确保这个 bucket 对应的老 bucket 完成了迁移过程。即老 bucket 里的 key 都要迁移到新的 bucket 中来（分裂到 2 个新 bucket），才能在新的 bucket 中进行插入或者更新的操作。
* 在循环的过程中，如果在 map 没有找到 key 的存在，也就是说原来 map 中没有此 key，这意味着插入新 key。那最终 key 的安置地址就是第一次发现的“空位”（tophash 是 empty）。
* 如果这个 bucket 的 8 个 key 都已经放置满了，那在跳出循环后，发现 inserti 和 insertk 都是空，这时候需要在 bucket 后面挂上 overflow bucket。当然，也有可能是在 overflow bucket 后面再挂上一个 overflow bucket。这就说明，太多 key hash 到了此 bucket。
* 在正式安置 key 之前，还要检查 map 的状态，看它是否需要进行扩容。如果满足扩容的条件，就主动触发一次扩容操作。
* 最后，会更新 map 相关的值，如果是插入新 key，map 的元素数量字段 count 值会加 1；在函数之初设置的 hashWriting 写标志flag清零。


## Map删除

删除操作同样是两层循环，核心还是找到 key 的具体位置。寻找过程都是类似的，在 bucket 中挨个 cell 寻找。
找到对应位置后，对 key 或者 value 进行“清零”操作：        
1. 对 key 清零
2. 对 value 清零
3. 将 count 值减 1，将对应位置的 tophash 值置成 Empty

## Map扩容

## Map遍历

**Q:「为什么遍历map的key是无序的？」**

map 在扩容后，会发生 key 的搬迁，原来落在同一个 bucket 中的 key，搬迁后，有些 key 就要远走高飞了。这样即使遍历的过程是顺序的，在经历扩容后再次遍历后的结果也会变得错乱，和原来的顺序有很大出入。     

当然，如果我就一个 hard code 的 map，我也不会向 map 进行插入删除的操作，按理说每次遍历这样的 map 都会返回一个固定顺序的 key/value 序列吧。的确是这样，但是 Go 杜绝了这种做法。

甚至 Go 做得更绝，当我们在遍历 map 时，并不是固定地从 0 号 bucket 开始遍历，每次都是从一个随机值序号的 bucket 开始遍历，并且是从这个 bucket 的一个随机序号的 cell 开始遍历。这样，即使你是一个写死的 map，仅仅只是遍历它，也不太可能会返回一个固定序列的 key/value 对了。

假设我们有下图所示的一个 map，起始时 B = 1，有两个 bucket，后来触发了扩容（这里不要深究扩容条件，只是一个设定），B 变成 2。并且， 1 号 bucket 中的内容搬迁到了新的 bucket，1 号裂变成 1 号和 3 号；0 号 bucket 暂未搬迁。老的 bucket 挂在在 *oldbuckets 指针上面，新的 bucket 则挂在 *buckets 指针上面。

![map_inter_A.png](/images/Go/map_inter_A.png)

这时，我们对此 map 进行遍历。假设经过初始化后，startBucket = 3，offset = 2。于是，遍历的起点将是 3 号 bucket 的 2 号 cell，下面这张图就是开始遍历时的状态：

![map_inter_B.png](/images/Go/map_inter_B.png)

标红的表示起始位置，bucket 遍历顺序为：3 -> 0 -> 1 -> 2。

因为 3 号 bucket 对应老的 1 号 bucket，因此先检查老 1 号 bucket 是否已经被搬迁过。           
判断方法就是：
```go
func evacuated(b *bmap) bool {
h := b.tophash[0]
return h > empty && h < minTopHash
}
```
如果 b.tophash[0] 的值在标志值范围内，即在 (0,4) 区间里，说明已经被搬迁过了。
在本例中，老 1 号 bucket 已经被搬迁过了。所以它的 tophash[0] 值在 (0,4) 范围内，因此只用遍历新的 3 号 bucket。

依次遍历 3 号 bucket 的 cell，这时候会找到第一个非空的 key：元素 e。到这里，我们的遍历结果仅有一个元素：
![map_inter_C.png](/images/Go/map_inter_C.png)

由于返回的 key 不为空，继续从上次遍历到的地方往后遍历，从新 3 号 overflow bucket 中找到了元素 f 和 元素 g。
遍历结果集也因此壮大：
![map_inter_D.png](/images/Go/map_inter_D.png)

新 3 号 bucket 遍历完之后，回到了新 0 号 bucket。0 号 bucket 对应老的 0 号 bucket，经检查，老 0 号 bucket 并未搬迁，因此对新 0 号 bucket 的遍历就改为遍历老 0 号 bucket。那是不是把老 0 号 bucket 中的所有 key 都取出来呢？

并没有这么简单，回忆一下，老 0 号 bucket 在搬迁后将裂变成 2 个 bucket：新 0 号、新 2 号。而我们此时正在遍历的只是新 0 号 bucket（注意，遍历都是遍历的 *bucket 指针，也就是所谓的新 buckets）。所以，我们只会取出老 0 号 bucket 中那些在裂变之后，分配到新 0 号 bucket 中的那些 key。

因此，`lowbits == 00` 的将进入遍历结果集：
![map_inter_E.png](/images/Go/map_inter_E.png)

和之前的流程一样，继续遍历新 1 号 bucket，发现老 1 号 bucket 已经搬迁，只用遍历新 1 号 bucket 中现有的元素就可以了。
继续遍历新 2 号 bucket，它来自老 0 号 bucket，因此需要在老 0 号 bucket 中那些会裂变到新 2 号 bucket 中的 key，也就是 lowbit == 10 的那些 key。

这样，遍历结果集变成：
![map_inter_F.png](/images/Go/map_inter_F.png)
最后，继续遍历到新 3 号 bucket 时，发现所有的 bucket 都已经遍历完毕，整个迭代过程执行完毕。


## 其他


## 参考
https://golang.design/go-questions/map/principal/


