---
title: "PHP的内存回收机制"
date: 2025-01-08T16:33:24+08:00
lastmod: 2025-01-08T16:33:24+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- PHP
- GC
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

本篇文章介绍PHP的内存回收机制，帮助我们理解在函数执行结束后，其作用域下的变量是如何释放其内存空间的。      

「引用计数」和「Garbage Collector」是内存回收的主要机制，「内存池」部分介绍“回收”这个动作代表的更底层的Zend Engine实现细节。

## 引用计数

PHP变量的基本单位是一个zval结构体，其中的ref_count属性用于跟踪指向该变量的引用数量。     

```c
typedef struct _zval_struct {
    zend_value value;        /* 变量的值 */
    ......
    zval *refcounted;          /* 引用计数指针，对于引用计数为1的变量，此字段为NULL */
    zend_ulong refcount;       /* 引用计数 */
} zval;
```

Zend Engine通过以下操作管理引用计数：
* 增加引用计数：
  * 当一个变量被赋值给另一个变量时，目标变量的ref_count会增加。
  * 当使用&符号进行引用赋值时，源变量和目标变量的ref_count都会增加。
* 减少引用计数：
  * 当一个变量不再被使用时（例如超出作用域，return，unset），其ref_count会减少。
  * 当一个变量被重新赋值时，其原有的ref_count会减少。

当变量的引用计数减少时，Zend Engine会检查其ref_count是否为零，如果为零：
* 对于简单类型（如整数、浮点数、字符串等），Zend Engine会直接释放其占用的内存。
* 对于复杂类型（如数组、对象等），Zend Engine会调用相应的析构函数，以确保资源的正确释放，然后释放内存。


## Garbage Collector

上面介绍的引用计数机制，能够处理绝大部分场景下的变量内存回收，但是有一种情况是这个机制无法解决的，从而因变量无法回收导致内存始终得不到释放，这种情况就是循环引用。

简单的描述就是变量的内部成员引用了变量自身，比如数组中的某个元素指向了数组，这样数组的引用计数中就有一个来自自身成员，试图释放数组时因为其ref_count仍然大于0而得不到释放，而实际上已经没有任何外部引用了，这种变量不可能再被使用，所以PHP引入了另外一个机制用来处理变量循环引用的问题。这就是GC(Garbage Collector)

### Garbage定义

下面看一个数组循环引用的例子：
```php
$a = [1];
$a[] = &$a;

unset($a);
```
**unset($a)** 之前引用关系：

![](/images/PHP/array_garbage.png)

注意这里$a的类型在&操作后已经转为引用，**unset($a)** 之后：

![](/images/PHP/array_garbage_unset.png)

可以看到，**unset($a)** 之后由于数组中有子元素指向$a，所以ref_count = 1，此时是无法通过正常的gc机制回收的，但是$a已经已经没有任何外部引用了，所以这种变量就是垃圾，垃圾回收器要处理的就是这种情况。

目前垃圾只会出现在array、object两种类型中，数组的情况上面已经介绍了，object的情况则是成员属性引用对象本身导致的，其它类型不会出现这种变量中的成员引用变量自身的情况，所以垃圾回收只会处理这两种类型的变量。

### 回收过程

如果当变量的ref_count减少后大于0，PHP并不会立即进行对这个变量进行垃圾鉴定，而是放入一个缓冲buffer中，等这个buffer满了以后(10000个值)再统一进行处理，加入buffer的是变量zend_value的zend_ref_counted_h:

```c
typedef struct _zend_ref_counted_h {
    uint32_t         ref_count; //记录zend_value的引用数
    union {
        struct {
            zend_uchar    type,  //zend_value的类型,与zval.u1.type一致
            zend_uchar    flags, 
            uint16_t      gc_info //GC信息，垃圾回收的过程会用到
        } v;
        uint32_t type_info;
    } u;
} zend_ref_counted_h;
```

一个变量只能加入一次buffer，为了防止重复加入，变量加入后会把zend_ref_counted_h.gc_info置为GC_PURPLE，即标为**紫色**，下次ref_count减少时如果发现已经加入过了则不再重复插入。

垃圾缓存区是一个双向链表，等到缓存区满了以后则启动垃圾检查过程：遍历缓存区，再对当前变量的所有成员进行遍历，然后把成员的ref_count减1(如果成员还包含子成员则也进行递归遍历，其实就是深度优先的遍历)，最后再检查当前变量的引用，如果减为了0则为垃圾。这个算法的原理很简单，垃圾是由于成员引用自身导致的，那么就对所有的成员减一遍引用，结果如果发现变量本身ref_count变为了0则就表明其引用全部来自自身成员。具体的过程如下：

1. 从buffer链表的roots开始遍历，把当前value标为**灰色**(zend_ref_counted_h.gc_info置为GC_GREY)，然后对当前value的成员进行深度优先遍历，把成员value的ref_count减1，并且也标为**灰色**；
2. 重复遍历buffer链表，检查当前value引用是否为0，为0则表示确实是垃圾，把它标为**白色**(GC_WHITE)，如果不为0则排除了引用全部来自自身成员的可能，表示还有外部的引用，并不是垃圾，这时候因为步骤(1)对成员进行了ref_count减1操作，需要再还原回去，对所有成员进行深度遍历，把成员ref_count加1，同时标为**黑色**；
3. 再次遍历buffer链表，将非GC_WHITE的节点从roots链表中删除，最终roots链表中全部为真正的垃圾，最后将这些垃圾清除。

综上，一次GC其实遍历了3遍buffer链表。

### 触发条件

1. 当zend_gc_globals的buf数量达到某个阈值(10000)时，会触发垃圾回收。
2. 手动调用gc_collect_cycles()函数可以强制进行垃圾回收。

PHP提供了几个配置选项来调整垃圾回收的行为：
* gc_enable：启用或禁用垃圾回收，默认是启用的。
* gc_probability 和 gc_divisor：这两个参数共同决定了触发垃圾回收的概率。例如，如果gc_probability设置为1，gc_divisor设置为1000，那么每创建1000个对象，就有1个机会触发垃圾回收。
* gc_maxlifetime：设置对象在内存中的最大存活时间，默认是24分钟。

## Zend内存池

在了解引用计数和GC后，我们可能会好奇PHP内部是如何做到释放变量内存的。了解下面关于Zend内存池的内容后，这个问题可能就迎刃而解了。

内存池是一种预先分配内存并管理其分配和释放的技术，旨在提高内存分配和释放的效率，减少内存碎片。其核心思想是预先申请一大块内存，然后将其分割成固定大小的小块，按需分配和回收这些小块。

Zend定义了三种粒度的内存块：chunk、page、slot，每个chunk的大小为2M，page大小为4KB，一个chunk被切割为512个page，而一个或若干个page被切割为多个slot，所以申请内存时按照不同的申请大小决定具体的分配策略：
* Huge(chunk): 申请内存大于2M，直接调用系统分配，分配若干个chunk
* Large(page): 申请内存大于3092B(3/4 page_size)，小于2044KB(511 page_size)，分配若干个page
* Small(slot): 申请内存小于等于3092B(3/4 page_size)，内存池提前定义好了30种同等大小的内存(8,16,24,32，...3072)，他们分配在不同的page上(不同大小的内存可能会分配在多个连续的page)，申请内存时直接在对应page上查找可用位置

![](/images/PHP/memory_pool.png)

Zend内存池初始化时会申请一个主chunk，后续申请的chunk会以双向链表的形式追加在链上。每个chunk都有一个free_map属性记录其下page的内存占用情况，**所以简单来讲，内存的申请和释放就是针对具体的chunk->free_map标识位进行更新**。

内存池的优点
* 提高性能：减少了系统调用的次数，内存分配和释放更快。
* 减少碎片：通过管理内存分配策略，可以有效减少内存碎片问题。
* 可控性：Zend Engine可以更好地控制内存的使用，有利于内存泄漏的检测和防止。


## 参考

[https://github.com/pangudashu/php7-internal/blob/master/5/gc.md](https://github.com/pangudashu/php7-internal/blob/master/5/gc.md)

[https://github.com/pangudashu/php7-internal/blob/master/5/zend_alloc.md](https://github.com/pangudashu/php7-internal/blob/master/5/zend_alloc.md)

[PHP的垃圾回收机制](https://blog.csdn.net/MrWangisgoodboy/article/details/130148349)