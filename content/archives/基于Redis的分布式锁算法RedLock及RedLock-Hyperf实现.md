---
title: "基于Redis的分布式锁算法RedLock及RedLock-Hyperf实现"
date: 2021-03-18T17:46:51+08:00
lastmod: 2021-03-18T17:46:51+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- PHP
- Redis
- SWOOLE
- Hyperf
- RedLock
description: "PHP Redis SWOOLE 基于Redis的分布式锁算法RedLock及RedLock-Hyperf实现"
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

### 前言

最近项目需要在Hyperf框架下封装Redis分布式锁，于是基于RedLock算法封装了 [RedLock-Hyperf](https://github.com/zonghay/redlock-hyperf) SDK，目前除支持简单对象调用外，也支持了在Hyperf框架下通过AOP注解来实现。
基于Redis实现一个分布式锁，相信这对你来说并不是难事。多数人会使用 **setnx + expire + del** 命令来实现一个简单的分布式锁，但这样的互斥锁真的安全吗。
本文我们一起探索一下Redis分布锁的常见实现方式以及Redis官方推荐的 [RedLcok](https://redis.io/topics/distlock) 算法如何保证锁的安全性。

> 多说一嘴，虽然目前市面上除了Redis实现的分布式锁，同样有通过zookeeper、etcd来实现的。但是，从成本上，后两者会给一个简单的分布式抢锁的动作带来一定的运维成本。从生态上看，相信目前对于绝大多数的项目没有不用Redis的。

### 单实例常见实现方式

#### 获取锁

* **setnx + expire**
    1. setnx key value：
       key存在返回false，key不存在写入返回true。
    2. expire key timeout：
       获取锁成功后，为锁设置过期时间。

  上述逻辑有个问题是，两个步骤是串行而非原子性的。无论是Redis Server还是Client在获取锁成功后发生故障并未执行expire，那么锁都将无法释放

* **set key value [EX seconds][PX milliseconds][NX|XX]**
  为了增强上述操作的原子性，我们使用Redis 2.6.12 版本后开始支持的set增强命令，这样只需要一条命令便可以完成setnx + expire，保证了原子性

```
SET key value[EX seconds][PX milliseconds][NX|XX]
```

#### 释放锁
在释放锁时，我们要验证value的值再进行删除操作，而不是简单粗暴的直接**del**（这样任何客户端都可以进行解锁操作）。同时为了保证校验value和删除操作的原子性，我们使用Lua脚本来操作
```lua
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

#### 关于value的唯一性

value必须要具有唯一性，假如value不具备唯一性，而是一个固定值，那么就可能存在下面的问题：
1. 客户端1获取锁成功
2. 客户端1在某个操作上阻塞了太长时间
3. 设置的key过期了，锁自动释放了
4. 客户端2获取到了对应同一个资源的锁
5. 客户端1从阻塞中恢复过来，因为value值一样，所以执行释放锁操作时就会释放掉客户端2持有的锁，这样就会造成问题

所以通常来说，在获取和释放锁时，我们要使用UUID或其他方式对value的唯一性进行保证。

### RedLock算法

上述常用互斥锁的使用方式是基于单点Redis Server或单个Redis Cluster，这种算法往往对Redis Crash的抵抗基本为零，锁的安全性保证不如分布式锁DLM（Distributed Lock Manager）高。
网上关于基于Redis实现的DLM有很多，今天的RedLock就是 Redis作者 antirez 在自己博客上提出的思路，[http://antirez.com/news/77](http://antirez.com/news/77)，后来社区上纷纷基于此做了各种语言的实现。

> This page is an attempt to provide a more canonical algorithm to implement distributed locks with Redis. We propose an algorithm, called Redlock, which implements a DLM which we believe to be safer than the vanilla single instance approach.

这里我们将对文章中核心进行翻译，帮助大家深入了解 RedLcok 算法。

#### 安全性和可用性保证

> **Safety and Liveness guarantees**
We are going to model our design with just three properties that, from our point of view, are the minimum guarantees needed to use distributed locks in an effective way.
Safety property: Mutual exclusion. At any given moment, only one client can hold a lock.
Liveness property A: Deadlock free. Eventually it is always possible to acquire a lock, even if the client that locked a resource crashes or gets partitioned.
Liveness property B: Fault tolerance. As long as the majority of Redis nodes are up, clients are able to acquire and release locks.

我们设计的分布式锁最少应该满足一下三点：

1. 安全性：任何时刻只能有一个client持有锁
2. 可用性A：死锁释放。即使client在持有锁阶段crash或者失联，一段时间后其他客户端也应该有能力获取到锁。
3. 可用性B：容错能力。主要多数Redis节点存活，客户端就应该能够获取和释放锁。

#### RedLock算法

在分布式版本，我们假设有N个Redis Matser节点。他们是完全独立的（注意这里的节点可以是N个Redis单master实例，也可以是N个Redis Cluster集群，但并不是有N个主节点的cluster集群）。
上面我们已经知道了在单实例上如何安全的获取释放锁，下面客户端将从N = 5的情况下按照以下步骤获取锁。

1. 获取当前时间的毫秒级时间戳
2. 依次尝试从5个实例，使用相同的key和具有唯一性的value(例如UUID)获取锁，当向Redis请求获取锁时，客户端应该设置一个网络连接和响应超时时间，这个超时时间应用小于锁的失效时间，例如你的锁自动失效时间为10s，则超时时间应该在5~50毫秒之间，这样可以避免服务器端Redis已经挂掉的情况下，客户端还在死死地等待响应结果。如果服务端没有在规定时间内响应，客户端应该尽快尝试去另外一个Redis实例请求获取锁
3. 客户端使用当前时间减去开始获取锁时间（步骤1记录的时间）就得到获取锁使用的时间，当且仅当从大多数(N/2+1，这里是3个节点)的Redis节点都取到锁，并且使用的时间小于锁失败时间时，锁才算获取成功。
4. 如果取到了锁，key的真正有效时间等于有效时间减去获取锁所使用的时间（步骤3计算的结果）
5. 如果某些原因，获取锁失败（没有在至少N/2+1个Redis实例取到锁或者取锁时间已经超过了有效时间），客户端应该在所有的Redis实例上进行解锁（即便某些Redis实例根本就没有加锁成功，防止某些节点获取到锁但是客户端没有得到响应而导致接下来的一段时间不能被重新获取锁）

#### 失败重试

当客户端未能获取到锁时，应该在一个随机延迟时间后再去尝试，目的是让在相同时间尝试获取同一个锁的其他客户端有可能争抢到锁（这里说的是如果有三个客户端都在争锁，可能出现 2，2，1 的情况谁也抢不到锁，这时候你所有客户端没有随机 delay 直接重试，很可能还是都抢不到。）client获取一个锁所花的时间越少，被其他客户端插进来的几率就越小。所以，客户端应该同时向N个实例发送SET命令。

#### 释放锁

无论客户端是否认为他获取到了锁，最后（无论是失败后立即释放，还是成功后完成逻辑操作主动释放）都应该向所有实例进行释放锁操作。

#### 安全性论证

假设一个客户端已经在多数实例中获取到了锁，key在不同实例上被设置了多次，所以他们的过期时间也不同。假设第一个被设置成功的key的时间是T1，最后一个key成功的时间是T2，那么第一个key的有效时间 MIN_VALIDITY=TTL-(T2-T1)-CLOCK_DRIFT ，其他key的有效时间都会比这个时间长，所以至少在这段时间内，所有的key同时处于被设置状态。
在这段时间内，多数实例上的key是处于被设置的状态，其他客户端是无法获取到锁的，所以如果一个锁被获取了，同一时间内他是不会再一次被获取的，因为SET NX操作不会在N/2+1个实例上成功。

#### 可用性论证

系统的可用性基于以下三个主要因素：

1. 锁的自动release（基于过期机制）：最终锁可以被再次获取
2. 客户端主动释放锁
3. 当客户端需要重试时，随机的delay时间要比在多数实例上获取到锁的时间要大，这样可以大概率的避免split brain的发生

#### 性能、故障恢复和文件同步

多数用户希望在使用Redis作为一个分布式锁服务的时候能有高性能表现，为了达到这个要求，我们可以使用多路复用的策略向N个Redis Server发送请求缩减整体延迟。
如果我们目标是要设计一个可以故障恢复的系统模型，数据持久化也是需要考虑的。假设我们的系统没有数据持久化，客户端A获取到了3/5实例中的锁，其中这三个中的一个实例发生率重启，那么在重启后另一个客户端就可以再次获取到相同的锁了，这违背了我们的互斥性原则。
如果我们开启了AOF，那么情况会好一些。我们关闭或者重启了Redis，因为expire是语义上实现的，所以过期时间还是有效流逝的，我们所有需求还是满足的。那如果是断电呢？如果Redis被设置成每秒钟做一次磁盘同步，很可能我们的key就丢失了。理论上，如果我们想要保证任何时刻锁的安全性，那么就需要把 fsync=always 打开，但是这在一定程度上又牺牲了Redis的性能。

#### 让算法更可靠：锁续租

如果客户端的工作是由很多小步骤完成的，那么我们可以将锁的有效时间设置的尽可能小，然后通过向服务端发送一个延长锁 TTL 的 Lua 脚本来延长锁的有效时间。
只有在有效时间内成功在多数实例上续租后，我们才认为锁成功续租。并且我们应该设置一个锁最多续租次数，来避免可用性A无法得到保证

### RedLock-Hyperf设计和使用

[RedLock-Hyperf](https://github.com/zonghay/redlock-hyperf) 是在RedLock算法上向 [Hyperf  ～2.1.*](https://github.com/hyperf/hyperf) 版本进行兼容改造。不仅支持简单对象调用，同样支持在Hyperf框架通过AOP注解方式使用

使用前需要现在/config/autoload/redis.php下进行Redis连接池配置作为独立实例

#### 引入

```php
composer require zonghay/redlock-hyperf
```

#### 普通对象方式
```php
    try {
        $lock = $this->container->get(RedLock::class)->setRedisPoolName()->setRetryCount(1)->lock('redlock-hyperf-test', 60000);
        if ($lock) {
            //do your code
            $this->container->get(RedLock::class)->unlock($lock);
        }
    } catch (\Throwable $throwable) {
        var_dump($throwable->getMessage());
    }
```

* setRedisPoolName方法用于指定Redlock使用哪些Redis实例作为分布式独立节点，这里需要传入索引数组，默认['default']，数组的值应该是/config/autoload/redis.php下的连接池name
  关于为什么要使用独立的Redis节点：
* setRetryCount方法用于设置获取锁的重试次数，默认2次
* setRetryDelay 用于一次获取锁失败后延迟时间后重试，默认200，单位毫秒
* lock方法，获取锁
    * resource：锁的key
    * ttl：锁过期时间，单位毫秒。
    * return：array|false
* unlock方法，释放锁
    * 参数：lock方法成功后的return
* 如果担心请求保持锁阶段进程出现重启或退出情况，建议增加以下代码

```php
//参考 RedlockHyperf\Aspect\RedLockAspect
if ($lock) {
  //to release lock when server receive exit sign
  Coroutine::create(function () use ($lock) {
  $exited = CoordinatorManager::until(Constants::WORKER_EXIT)->yield($lock['validity']);
  $exited && $this->redlock->unlock($lock);
  });
  //do your code
  $this->redlock->unlock($lock);
  return $result;
}
```

#### AOP注解方式

```php
class IndexController extends AbstractController
{
    /**
     * @RedLockAnnotation(resource="redlock-hyperf-test", poolName={"default"})
     */
    public function index() {}
}
```
SDK提供 RedlockHyperf\Annotation\RedLockAnnotation 注解，作用类于方法上，可以配置resource（必填），poolName，poolName，poolName，clockDriftFactor，ttl等参数

### 最后

关于RedLock的安全性问题，分布式系统专家Martin Kleppmann和antirez 之间就发生过一场争论，网上也有这部分争论的翻译，推荐大家可以看一下，看后对分布式场景的理解会很有收获。
[基于Redis的分布式锁到底安全吗（上）](https://cloud.tencent.com/developer/article/1418249)
[基于Redis的分布式锁到底安全吗（下）](https://cloud.tencent.com/developer/article/1418246)

### 参考资料

* http://antirez.com/news/77
* https://redis.io/topics/distlock
* https://juejin.cn/post/6844903830442737671#heading-8
* https://www.v2ex.com/t/693936
* https://www.v2ex.com/t/693676#reply15
* https://github.com/mike-marcacci/node-redlock