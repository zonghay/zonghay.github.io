---
title: "Golang 底层实现之sync.map"
date: 2025-03-19T11:13:29+08:00
lastmod: 2025-03-20T11:13:29+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- Golang
- Sync
- Map
- 底层实现
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

之前在文章[《Golang 底层实现之map》](https://zonghay.github.io/archives/golang_%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0%E4%B9%8Bmap/)里介绍过map在高并发场景下赋值or更新时会检查其标志位flags，当有其他协程在进行“写”操作时会触发panic。         
通常为了避免程序出现预期之外的并发读写问题，我们会对map进行加锁保护，或者直接使用官方提供的并发安全map——sync.map。           
所以本文将从底层实现的角度介绍sync.map，并给出在哪些场景下更适合使用sync.map。

## 底层实现

一句话总结：sync.map本质是运用**读写分离**(也可以理解为空间换时间)的数据思路来解决高并发读写问题。

### 数据结构
```go
// go 1.19+
type Map struct {
	_ noCopy

	mu Mutex
	// 1.read 是 map 的只读视图，支持无锁并发访问
	// 2.更新 read 指针需要加锁（mu）
	// 3.常规更新可以直接无锁进行（原子操作*entry）
	// 4.expunged 状态表示条目已被逻辑删除但尚未物理清除
	read atomic.Pointer[readOnly]
	
	// 1.dirty 是需要加锁访问的数据集
	// 2.dirty 包含 read 中所有有效(未逻辑删除)条目，以方便快速提升
	// 3.expunged 条目不会进入 dirty
	// 4.更新 expunged 条目需要先将其移回 dirty
	// 5.dirty 的初始化采用懒加载模式（第一次写入时浅拷贝 read）
	dirty map[any]*entry

	// 1.misses 用于统计自上次更新 read map 以来，需要加锁（mu）才能确定键是否存在的加载操作次数，无论key是否在 dirty 中
	// 2.当 misses 累计值达到阈值（默认是 dirty 的长度）时提升操作会将 dirty 直接替换为新的 read，同时重置状态
	// 3.下次写入时会基于新的 read 重新创建 dirty（触发懒加载初始化）
	misses int
}

type readOnly struct {
    m       map[any]*entry
    amended bool // Map.dirty的数据和Map.read不一样时为true
}

type entry struct {
    p atomic.Pointer[any]
}
```

### 读
```go
// Load returns the value stored in the map for a key, or nil if no
// value is present.
// The ok result indicates whether value was found in the map.
func (m *Map) Load(key any) (value any, ok bool) {
	read := m.loadReadOnly()
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()
		// Avoid reporting a spurious miss if m.dirty got promoted while we were
		// blocked on m.mu. (If further loads of the same key will not miss, it's
		// not worth copying the dirty map for this key.)
		read = m.loadReadOnly()
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			// Regardless of whether the entry was present, record a miss: this key
			// will take the slow path until the dirty map is promoted to the read
			// map.
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load()
}

func (m *Map) loadReadOnly() readOnly {
    if p := m.read.Load(); p != nil {
    return *p
    }
    return readOnly{}
}
```
结合下图和源码注释可以很好的理解Load函数逻辑：
* 快速路径（无锁读取）
    * 首先原子加载 readOnly 结构
    * 直接从 read.m 中查找键
    * 如果找到且 !read.amended，直接返回结果（无锁操作）
* 慢速路径（加锁读取） 
  * 当键不在 read 中且 read.amended 为 true 时：     
    a. 获取互斥锁 m.mu           
    b. 双重检查（避免在加锁期间 dirty 已提升为 read）            
    c. 从 dirty map 中查找键         
    d. 记录一次 miss（用于触发后续的 dirty 提升）          
    e. 释放锁          
* 结果处理 
  * 如果找到 entry，调用 e.load() 加载实际值 
  * 未找到则返回 (nil, false)

![sync_map_load.png](/images/Go/sync_map_load.png)

### 删
```go
// Delete deletes the value for a key.
func (m *Map) Delete(key any) {
	m.LoadAndDelete(key)
}

// LoadAndDelete deletes the value for a key, returning the previous value if any.
// The loaded result reports whether the key was present.
func (m *Map) LoadAndDelete(key any) (value any, loaded bool) {
	read := m.loadReadOnly()
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()
		read = m.loadReadOnly()
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			delete(m.dirty, key)
			// Regardless of whether the entry was present, record a miss: this key
			// will take the slow path until the dirty map is promoted to the read
			// map.
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if ok {
		return e.delete()
	}
	return nil, false
}

func (e *entry) delete() (value any, ok bool) {
	for {
		p := e.p.Load()
		if p == nil || p == expunged {
			return nil, false
		}
		if e.p.CompareAndSwap(p, nil) {
			return *p, true
		}
	}
}
```
删除逻辑大致相似Load，只不过增加了相应的删除逻辑：
* 删除存在于 read 的键：通过原子操作将 entry.p 设为 nil，值仍保留在 read 中（逻辑删除）。
* 删除仅存在于 dirty 的键：加锁从 dirty 中物理删除，记录 miss 计数。
![sync_map_delete.png](/images/Go/sync_map_delete.png)

### 写
```go
// Store sets the value for a key.
func (m *Map) Store(key, value any) {
	_, _ = m.Swap(key, value)
}

// Swap swaps the value for a key and returns the previous value if any.
// The loaded result reports whether the key was present.
func (m *Map) Swap(key, value any) (previous any, loaded bool) {
	read := m.loadReadOnly()
	if e, ok := read.m[key]; ok {
		// trySwap swaps a value if the entry has not been expunged.
		if v, ok := e.trySwap(&value); ok {
			if v == nil {
				return nil, false
			}
			return *v, true
		}
	}

	m.mu.Lock()
	read = m.loadReadOnly()
	if e, ok := read.m[key]; ok {
		if e.unexpungeLocked() {
			// The entry was previously expunged, which implies that there is a
			// non-nil dirty map and this entry is not in it.
			m.dirty[key] = e
		}
		if v := e.swapLocked(&value); v != nil {
			loaded = true
			previous = *v
		}
	} else if e, ok := m.dirty[key]; ok {
		if v := e.swapLocked(&value); v != nil {
			loaded = true
			previous = *v
		}
	} else {
		if !read.amended {
			// We're adding the first new key to the dirty map.
			// Make sure it is allocated and mark the read-only map as incomplete.
			m.dirtyLocked()
			m.read.Store(&readOnly{m: read.m, amended: true})
		}
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
	return previous, loaded
}


func (m *Map) dirtyLocked() {
	if m.dirty != nil {
		return
	}

	read := m.loadReadOnly()
	m.dirty = make(map[any]*entry, len(read.m))
	for k, e := range read.m {
		if !e.tryExpungeLocked() {
			m.dirty[k] = e
		}
	}
}
```

![sync_map_store.png](/images/Go/sync_map_store.png)

### 遍历
如果已经熟悉了Sync.Map的读写逻辑，那么遍历操作没啥好讲的了，直接看源码吧
```go
func (m *Map) Range(f func(key, value any) bool) {
	read := m.loadReadOnly()
	if read.amended {
		m.mu.Lock()
		read = m.loadReadOnly()
		if read.amended {
			read = readOnly{m: m.dirty}
			copyRead := read
			m.read.Store(&copyRead)
			m.dirty = nil
			m.misses = 0
		}
		m.mu.Unlock()
	}

	for k, e := range read.m {
		v, ok := e.load()
		if !ok {
			continue
		}
		if !f(k, v) {
			break
		}
	}
}
```

## 场景推荐

| 场景                     | 推荐方案                   |
|--------------------------|----------------------------|
| 全局配置、只读缓存      | sync.Map                   |
| 高并发读、低频更新      | sync.Map                   |
| 计数器、排行榜          | sync.Mutex + map           |
| 需要遍历或复杂操作      | sync.Mutex + map           |
| 内存敏感                | sync.Mutex + map           |


## 参考
[由浅入深聊聊Golang的sync.Map](https://juejin.cn/post/6844903895227957262?spm=a2c6h.12873639.article-detail.4.5c3e5482Ol6tbV#heading-9)