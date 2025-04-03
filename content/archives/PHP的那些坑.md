---
title: "PHP的那些「坑」"
date: 2023-01-05T18:47:02+08:00
lastmod: 2025-01-05T18:47:02+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- PHP
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

## 使用引用变量foreach数组，未unset问题
foreach数组是引用传递变量，未unset导致再次foreach末尾元素值被覆盖
https://www.cnblogs.com/sunshineliulu/p/12853531.html

## in_array严格检查问题(PHP8之前)
bool in_array ( mixed $needle , array $haystack [, bool $strict = FALSE ] )
在 haystack 中搜索 needle，如果第三个参数 strict 的值为 true，则 in_array 函数还会检查 needle 的 类型 是否和 haystack 中的相同。
```php
<?php

$arr = [0, 1];
$a = '1test';
if (in_array($a, $arr)) {
    echo true;
} else {
    echo false;
}

// 输出结果
// true
```

## PHP浮点数计算精度丢失问题
使用PHP官方推荐BCMath库进行计算        
> 在内部，BCMath库将每个数字作为字符串处理，其中每个字符代表数字的一位。这样，数字的大小不再受到浮点数或整数类型的限制                 

https://blog.csdn.net/qq_25123887/article/details/128233993

## PHP的强等是===,Golang的强等是==
https://www.php.net/manual/zh/language.operators.comparison.php

## Fatal error、Exception、Throwable
* Throwable是最顶层的接口，包含了所有的错误和异常。
* Exception是Throwable的一个子类，专门用于表示程序中的常规异常情况。
* Fatal Error原本是一种独立于异常之外的严重错误类型，但在PHP7及以后的版本中，很多致命错误被重新分类为实现了Throwable接口的特殊异常（如FatalErrorException），从而使得它们也可以被捕获和处理。

## empty("0") return true
