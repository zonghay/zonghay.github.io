---
title: "PHP底层实现之数组"
date: 2024-11-15T16:22:11+08:00
lastmod: 2024-11-15T16:22:11+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- PHP
- HashTable
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

曾经听前同事讲过一句话「**PHP的精髓是他的数组**」。       
彼时年龄和经验都尚浅，对这句话体会不是很深。但自从开始写golang，我是越来越怀念PHP数组那种简单、粗暴的，不管三七二十一，拿来直接往里面塞的编码体验了。     
于是，便有了这篇探索PHP到底是如何实现这个能够「容纳万物」的数据类型的文章。         
知其然，更要知其所以然。

## HashTable定义
首先，PHP数组底层的数据结构是zend自己用C语言实现的HashTable，而且这个HashTable类型被用在PHP内核的很多地方。        
下面是PHP7.4中HashTable的源码定义(php-src/Zend/zend_types.h)：
```C
//Bucket：散列表中存储的元素
typedef struct _Bucket {
	zval              val;
	zend_ulong        h;                /* hash value (or numeric index)   */
	zend_string      *key;              /* string key or NULL for numerics */
} Bucket;

typedef struct _zend_array HashTable;

struct _zend_array {
	zend_refcounted_h gc;               //这是用于垃圾回收的引用计数头部。所有PHP的zval和复合类型（如数组）都使用了这种结构，以便能够进行引用计数和垃圾收集
	union {
		struct {
			ZEND_ENDIAN_LOHI_4(
				uint8_t    flags,
				uint8_t    _unused,
				uint8_t    nIteratorsCount,
				uint8_t    _unused2)
		} v;
		uint32_t flags;
	} u;                                //这是一个联合体，用于以不同的方式访问相同的内存区域。ZEND_ENDIAN_LOHI_4 是一个宏，用于处理不同字节序（大端和小端）的系统。它定义了一个包含四个 uint8_t 类型的匿名结构体，分别对应 flags, _unused, nIteratorsCount, 和 _unused2。同时，整个结构体可以作为一个 uint32_t 类型的 flags 被访问。
	union {
		uint32_t     *arHash;           /* hash table (allocated above this pointer) */ //指向哈希表的指针，用于关联数组
		Bucket       *arData;           /* array of hash buckets */                     //指向包含键值对的桶数组的指针（存在不连续的索引数组）
		zval         *arPacked;         /* packed array of zvals */                     //指向紧密打包的 zval 数组的指针，用于索引数组
	};                                  // 这是存储数组元素的联合体，可以以上面三种不同的形式访问
	uint32_t          nNumUsed;         //表示 arData 中已经使用的 Bucket 数量，包括了可能存在的空洞（已删除的元素）
	uint32_t          nNumOfElements;   //表示哈希表中有效元素的数量，即实际存储在 HashTable 中的元素数量
	uint32_t          nTableSize;       //哈希表的总大小，必须是 2 的幂次方，用于分配 arData 和 arHash
	uint32_t          nTableMask;       //哈希值计算掩码，用于快速计算哈希值在哈希表中的位置，等于 nTableSize 的负值（nTableMask = -nTableSize）
	uint32_t          nInternalPointer; //当前数组内部指针的位置，用于遍历数组时追踪位置
	zend_long         nNextFreeElement; //下一个可用的数值索引。例如，当你使用类似 arr[] = 1; 的语法添加元素到数组时，nNextFreeElement 将被用来分配键值，并且在添加后自增。如果还有类似 arr["a"] = 2; arr[] = 3; 的操作，则nNextFreeElement = 2，nNextFreeElement 会跳过任何明确指定的键，并找到下一个可用的数值索引 
	dtor_func_t       pDestructor;      //元素的析构函数指针。当一个元素要从 HashTable 中移除时，这个函数会被调用来执行任何必要的清理操作，例如减少引用计数或释放内存。
};

```
HashTable结构体成员变量重点说明：
* 当将一个元素从哈希表删除时并不会将对应的Bucket移除，而是将Bucket存储的zval修改为IS_UNDEF。并且只将nNumOfElements--
* 另外一个非常重要的值arData，这个值指向存储元素数组的第一个Bucket，插入元素时按顺序 依次插入 数组，比如第一个元素在arData[0]、第二个在arData[1]...arData[nNumUsed]。PHP数组的有序性正是通过arData保证的，这是第一个与普通散列表实现不同的地方。

![](/images/PHP/hashtable.png)
图片摘自于[PHP7中数组的一些改进](https://solupro.org/PHP7-something-about-array/)

## HashTable初始化

1. 计算内存大小：首先计算总的内存大小，这包括了 arHash 和 arData 所需的内存。arHash 的大小取决于 nTableSize（最小值为8），而 arData 的大小取决于 nTableSize * Bucket 的大小。
2. 分配内存：首先为 HashTable 结构分配内存。
3. 设置初始值：初始化 HashTable 的字段，如设置初始元素数目、哈希表大小等。
4. 内存对齐：计算 arData 的起始地址，使其对齐以便于高效访问。arData 是一个指向 Bucket 数组的指针，每个 Bucket 存储一个键值对。
5. 分配 arData 数组：根据需要的初始大小，为 Bucket 数组分配内存。
6. 初始化 arData 数组：将 arData 数组中的每个 Bucket 初始化为空或未使用状态。
7. 设置哈希掩码：nTableMask 用于确定哈希值如何映射到 arData 数组的索引。它通常设置为 nTableSize - 1，其中 nTableSize 是 arData 数组的大小。
8. 设置析构函数：如果提供，设置 pDestructor，这是一个回调函数，用于在删除数组元素时释放分配的资源。
9. 其他字段：初始化其他字段，如迭代器计数、内部指针、下一个空闲元素的索引等。

## Hash函数

hash函数（DJBX33A用于字符串的键，DJBX33A是PHP用到的一种hash算法）返回的hash值一般是32位或者64位的整型数，这个数有点大，不能直接作为hash数组的索引。首先通过求余操作将这个数调整到hashtable的大小的范围之内。求余是通过计算hash & (ht->nTableSize - 1)，而不是hash % ht->nTableSize，因为ht->nTableSize是2的幂，所以这两种计算的结果是一样的（注意第一种情况用的是与运算&），第二种计算方式需要使用开销更大的整数除法运算。ht->nTableSize -1 的值保存在ht->nTableMask中。

## HashTable插入

我们这里以关联数组为例： 
1. 元素按顺序插入arData中（PHP数组的有序性正是通过arData保证的，这是第一个与普通散列表实现不同的地方），获取元素所在位置idx
2. 计算哈希值,根据key的哈希值映射到散列表中的某个位置nIndex，将idx存入这个位置
3. 如果nIndex冲突，则放弃更新此处的idx，找到「前夫哥」在arData中的索引位，将自己的idx位置更新在「前夫哥」Bucket的zval.u2.next值上
4. 在散列表中的位置存储nNextFreeElement指向 arData 中新插入 Bucket 的索引
5. 增加 nNumOfElements。如果插入的是一个新 Bucket（而不是替换或更新现有的），也增加 nNumUsed。

## HashTable查找

查找的流程其实和插入流程的2、3就很像了。       
只是如果出现hash冲突的话：         
在hash数组中查找索引为idx = ht->arHash[hash & ht->nTableMask]的元素。这个索引对应的元素是冲突处理链表的头元素。所以ht->arData[idx]是我们要检查的第一个元素。如果这个元素中保存的键跟我们要查找的键相同，那么查找就搞定了。          
如果不的相同的话，则需要查找冲突处理链表的下一个元素，这个元素的索引保存在bucket->val.u2.next中，它保存在zval结构体中很少会用到的最后4个字节中。我们继续遍历这个链表（使用索引而不是指针）直到找到我们要找的bucket，或者是碰到INVALID_IDX——这意味着你要查找的key并不存在。

## HashTable扩容

散列表可存储的value数是固定的，当空间不够用时就要进行扩容了。

PHP散列表的大小为2^n，插入时如果容量不够则首先检查已删除元素所占比例，如果达到阈值（ht->nNumUsed > ht->nNumOfElements + (ht->nNumOfElements >> 5)，则将已删除元素移除，重建索引，如果未到阈值则进行扩容操作，扩大为当前大小的2倍，将当前Bucket数组复制到新的空间，然后重建索引。

### 参考
[PHP内核剖析-2.2数组](https://github.com/pangudashu/php7-internal/blob/master/2/zend_ht.md)
[PHP7中新的Hashtable实现和性能改进](https://gywbd.github.io/posts/2014/12/php7-new-hashtable-implementation.html)