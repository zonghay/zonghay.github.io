---
title: "基于Redis的分布式锁算法RedLock及RedLock-Hyperf实现"
date: 2021-03-18T17:46:51+08:00
lastmod: 2021-03-18T17:46:51+08:00
author: ["ZhangYong"]
tags: # 标签
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
---

### Introduction

Recently, the project required encapsulating a Redis distributed lock under the Hyperf framework, so I encapsulated the [RedLock-Hyperf](https://github.com/zonghay/redlock-hyperf) SDK based on the RedLock algorithm. Currently, in addition to supporting simple object calls, it also supports implementation through AOP annotations within the Hyperf framework.
Implementing a distributed lock with Redis is probably not a difficult task for you. Most people use the setnx + expire + del commands to implement a simple distributed lock, but is such a mutex really secure?
In this article, we will explore together the common implementations of Redis distributed locks and how the officially recommended  [RedLcok](https://redis.io/topics/distlock) algorithm by Redis ensures the security of the lock.

> As a side note, although there are distributed locks implemented with Zookeeper and etcd besides Redis, these latter two options bring certain operational costs for a simple action of distributed lock contention. From an ecosystem perspective, I believe that most projects nowadays cannot do without Redis.

### Common Implementation Methods for Single Instance

#### Acquire Lock

* **setnx + expire**
    1. setnx key value：
       if key exist return false，otherwise set key a value and return true。
    2. expire key timeout：
        after acquiring lock successfully, set key with an expiring time.

  The logic mentioned above has an issue in that the two steps are sequential rather than atomic. Whether it's the Redis Server or the Client that encounters a failure after successfully acquiring the lock but before executing expire, the lock will not be released.

* **set key value [EX seconds][PX milliseconds][NX|XX]**
  To enhance the atomicity of the above operation, we use the enhanced SET command supported by Redis starting from version 2.6.12. With this, we can complete SETNX + EXPIRE with just one command, ensuring atomicity.

```
SET key value[EX seconds][PX milliseconds][NX|XX]
```

#### Release Lock
When releasing the lock, we need to verify the value of value before performing the delete operation, rather than simply and crudely using **DEL** directly (as this would allow any client to unlock). To ensure the atomicity of the value verification and delete operations, we use a Lua script to handle it.
```lua
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

#### About Uniqueness of value

The value must be unique. If the value is not unique and instead a fixed value, then the following problems may arise:
1. Client 1 acquires the lock successfully.
2. Client 1 gets blocked for too long on a certain operation.
3. The key expires and the lock is automatically released.
4. Client 2 acquires the lock for the same resource.
5. Client 1 recovers from being blocked and, because the value is the same, it releases the lock during the unlock operation, which then inadvertently releases the lock held by Client 2, causing issues.

Therefore, it is generally advisable to ensure the uniqueness of the value when acquiring and releasing locks by using UUIDs or other methods.

### RedLock Algorithm

The commonly used mutex lock approach described above is based on a standalone Redis Server or a single Redis Cluster. This algorithm often has virtually no resistance to Redis crashes, and the guarantee of lock safety is not as high as that of a Distributed Lock Manager (DLM).

There are many Distributed Lock Managers (DLMs) based on Redis available online. RedLock is an idea proposed by Redis creator antirez on his blog，[http://antirez.com/news/77](http://antirez.com/news/77)，Subsequently, the community has developed various implementations in different programming languages based on this concept.

> This page is an attempt to provide a more canonical algorithm to implement distributed locks with Redis. We propose an algorithm, called Redlock, which implements a DLM which we believe to be safer than the vanilla single instance approach.

#### Security and availability guarantees

> **Safety and Liveness guarantees**
We are going to model our design with just three properties that, from our point of view, are the minimum guarantees needed to use distributed locks in an effective way.
Safety property: Mutual exclusion. At any given moment, only one client can hold a lock.
Liveness property A: Deadlock free. Eventually it is always possible to acquire a lock, even if the client that locked a resource crashes or gets partitioned.
Liveness property B: Fault tolerance. As long as the majority of Redis nodes are up, clients are able to acquire and release locks.

#### RedLock算法

> In the distributed version of the algorithm we assume to have N Redis masters. Those nodes are totally independent, so we don’t use replication or any other implicit coordination system. We already described how to acquire and release the lock safely in a single instance. We give for granted that the algorithm will use this method to acquire and release the lock in a single instance. In our examples we set N=5, which is a reasonable value, so we need to run 5 Redis masters in different computers or virtual machines in order to ensure that they’ll fail in a mostly independent way.
>
> In order to acquire the lock, the client performs the following operations:
>
> Step 1) It gets the current time in milliseconds.
>
> Step 2) It tries to acquire the lock in all the N instances sequentially, using the same key name and random value in all the instances.
>
> During the step 2, when setting the lock in each instance, the client uses a timeout which is small compared to the total lock auto-release time in order to acquire it.
For example if the auto-release time is 10 seconds, the timeout could be in the ~ 5-50 milliseconds range.
This prevents the client to remain blocked for a long time trying to talk with a Redis node which is down: if an instance is not available, we should try to talk with the next instance ASAP.
>
> Step 3) The client computes how much time elapsed in order to acquire the lock, by subtracting to the current time the timestamp obtained in step 1.
If and only if the client was able to acquire the lock in the majority of the instances (at least 3), and the total time elapsed to acquire the lock is less than lock validity time, the lock is considered to be acquired.
>
> Step 4) If the lock was acquired, its validity time is considered to be the initial validity time minus the 
>
> Step 5) If the client failed to acquire the lock for some reason (either it was not able to lock N/2+1 instances or the validity time is negative), it will try to unlock all the instances (even the instances it believe it was not able to lock).

### RedLock-Hyperf Design And Import

[RedLock-Hyperf](https://github.com/zonghay/redlock-hyperf) It involves adapting the RedLock algorithm for compatibility with [Hyperf  ～2.1.*](https://github.com/hyperf/hyperf) versions. It supports not only simple object calls but also the use of AOP annotations within the Hyperf framework.

Before using it, you need to configure a Redis connection pool as a separate instance in /config/autoload/redis.php.

#### Import

```php
composer require zonghay/redlock-hyperf
```

#### Simple Object Invocation
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

* The setRedisPoolName method is used to specify which Redis instances Redlock should use as distributed independent nodes. Here, an index array must be passed in, with the default being ['default']. The values in the array should correspond to the connection pool names in /config/autoload/redis.php

  Reasons for using independent Redis nodes include:
* The setRetryCount method is used to set the number of retries for acquiring a lock, with the default being 2 attempts.
* The setRetryDelay method is for setting the delay time before retrying after a failed lock acquisition attempt, with the default being 200 milliseconds.
* The lock method is for acquiring a lock.
    * resource: The key for the lock.
    * ttl: The expiration time of the lock, in milliseconds.
    * return: array|false
* The unlock method is for releasing a lock.
    * Parameter: The return value from a successful lock method call.
* If there are concerns about the process restarting or exiting during the phase where the request holds the lock, it is recommended to add the following code:

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

#### AOP

```php
class IndexController extends AbstractController
{
    /**
     * @RedLockAnnotation(resource="redlock-hyperf-test", poolName={"default"})
     */
    public function index() {}
}
```
The SDK provides the RedlockHyperf\Annotation\RedLockAnnotation annotation, which can be applied to methods. It allows configuration of parameters such as resource (required), poolName, clockDriftFactor, ttl, and others.

### Lastly

Regarding the security issues of RedLock, there has been a debate between distributed systems expert Martin Kleppmann and antirez. Translations of this debate are available online, and it is recommended that everyone take a look. After reading, you will gain a lot of insight into distributed systems scenarios.
[基于Redis的分布式锁到底安全吗（上）](https://cloud.tencent.com/developer/article/1418249)
[基于Redis的分布式锁到底安全吗（下）](https://cloud.tencent.com/developer/article/1418246)

### References

* http://antirez.com/news/77
* https://redis.io/topics/distlock
* https://juejin.cn/post/6844903830442737671#heading-8
* https://www.v2ex.com/t/693936
* https://www.v2ex.com/t/693676#reply15
* https://github.com/mike-marcacci/node-redlock