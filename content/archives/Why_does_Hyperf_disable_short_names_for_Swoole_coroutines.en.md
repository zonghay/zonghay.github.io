---
title: "Why does Hyperf disable short names for Swoole coroutines"
date: 2021-01-12T17:44:50+08:00
lastmod: 2021-01-12T17:44:50+08:00
author: ["ZhangYong"]
tags: # 标签
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
---

In the server requirements section of the official Hyperf documentation, it is mentioned:
>Swoole PHP extension >= 4.5, with Short Name disabled

Furthermore, in the documentation's FAQ section, one can find a tag titled[Swoole Short Name Not Disabled.](https://hyperf.wiki/2.0/#/zh-cn/quick-start/questions?id=swoole-%e7%9f%ad%e5%90%8d%e6%9c%aa%e5%85%b3%e9%97%ad)

I would like to inquire why it is imperative for Hyperf to disable Swoole's coroutine short names.

First, let's examine what [Swoole's coroutine short names](https://wiki.swoole.com/#/other/alias?id=%e5%8d%8f%e7%a8%8b%e7%9f%ad%e5%90%8d%e7%a7%b0) are
> All class names prefixed with Swoole\Coroutine are aliased to Co. Additionally, there are the following mappings: the go() function for creating coroutines, the chan() function for channel operations, and the defer() function for deferred execution.

From the above explanation, we understand that Hyperf primarily wishes to prevent the use of these specific functions, but why is their use discouraged? Considering that the go() function is often used in code to resolve blocking issues, does this mean my code is not coroutine-based? After testing in Hyperf, it was found that the go() function is indeed effective in creating coroutines, so what caused the initially disabled go() function to be "revived"?

When clicking on the go() function in PhpStorm, we are redirected to the vendor/hyperf/utils/src/Functions.php file, which is specified as an autoload file in composer.json

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
It can be seen that we have "hijacked" the go() function and made some modifications to it, capturing exceptions thrown during coroutine creation and printing them to the console. (Note: When exceptions occur within the callable of a coroutine created using the Coroutine::create method, a Fatal error is thrown, which is something we do not wish to see.)

Therefore, at this point, it seems we understand why Hyperf disables Swoole's short names. The purpose is to hijack the go() and co() functions to capture exceptions in the callable to avoid the process throwing a Fatal error.