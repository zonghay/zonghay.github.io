---
title: "Hyperf为什么要关闭Swoole协程短名称"
date: 2021-01-12T17:44:50+08:00
lastmod: 2021-01-12T17:44:50+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- PHP
- SWOOLE
- Hyperf
description: "PHP SWOOLE Hyperf为什么要关闭Swoole协程短名称"
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

在Hyperf官方文档的服务器要求中提到
>Swoole PHP 扩展 >= 4.5，并关闭了 Short Name

并且，在文档的常见问题中也会看到[Swoole 短名未关闭](https://hyperf.wiki/2.0/#/zh-cn/quick-start/questions?id=swoole-%e7%9f%ad%e5%90%8d%e6%9c%aa%e5%85%b3%e9%97%ad)这一个tag。

我想问了，那为什么hyperf一定要关闭掉Swoole的协程短名称呢

首先，我们先看一下什么是Swoole的[协程短名称](https://wiki.swoole.com/#/other/alias?id=%e5%8d%8f%e7%a8%8b%e7%9f%ad%e5%90%8d%e7%a7%b0)
> 所有的 Swoole\Coroutine 前缀的类名映射为 Co。此外还有下面的一些映射：创建协程 go() 函数，通道操作 chan() 函数，延迟执行 defer() 函数

从上面的解释我们知道了，hyperf主要就是不想让我们使用以上这几个函数，但是为啥不让我们使用的呢？想到之前在代码中经常使用go()函数来解决代码中的阻塞问题，难道说我写的代码并没有协程化？在Hyperf经过测试之后发现，go()函数协程话确实是生效的，那究竟是什么让本来已经被禁用的go()函数又“复活”了呢？

在phpStrom只点击go()函数我们跳转到了vendor/hyperf/utils/src/Functions.php文件，该文件是在composer.json中指定的自动化加载文件

```php
if (! function_exists('go')) {
    /**
     * @return bool|int
     */
    function go(callable $callable)
    {
        $id = Coroutine::create($callable);
        return $id > 0 ? $id : false;
    }
}
```
如果框架里没定义go()函数的话，就会执行这里的逻辑去调用Coroutine::create($callable)，注意这里的Coroutine类并不是Swoole\Coroutine，而是vendor/hyperf/utils/src/Coroutine.php

```php
public static function create(callable $callable): int
    {
        $result = SwooleCoroutine::create(function () use ($callable) {
            try {
                call($callable);
            } catch (Throwable $throwable) {
                if (ApplicationContext::hasContainer()) {
                    $container = ApplicationContext::getContainer();
                    if ($container->has(StdoutLoggerInterface::class)) {
                        /* @var LoggerInterface $logger */
                        $logger = $container->get(StdoutLoggerInterface::class);
                        /* @var FormatterInterface $formatter */
                        if ($container->has(FormatterInterface::class)) {
                            $formatter = $container->get(FormatterInterface::class);
                            $logger->warning($formatter->format($throwable));
                        } else {
                            $logger->warning(sprintf('Uncaptured exception[%s] detected in %s::%d.', get_class($throwable), $throwable->getFile(), $throwable->getLine()));
                        }
                    }
                }
            }
        });
        return is_int($result) ? $result : -1;
    }
```
可以看到，我们“劫持”了go()函数，给他做了一些改动，捕获了创建协程时抛出的异常，将异常打印到控制台上。（注：对于Coroutine::create方式创建的协程在callable中存在异常时会抛出Fatal error，这是我们不愿意看到的）。

所以，到这里我们似乎理解了为什么Hyperf要关闭Swoole的短名称，目的就是劫持go()、co()函数来捕获callable的异常避免进程抛出Fatal error