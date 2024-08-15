---
title: "Golang Redis分布式锁实现"
date: 2024-08-14T17:26:59+08:00
lastmod: 2024-08-14T17:26:59+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- Redis
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

```go
package lock

import (
	"context"
	"errors"
	"cache"
	"time"

	"github.com/go-redis/redis/v8"
	"github.com/google/uuid"
)

const (
	DefaultExpirationTime = 5 * time.Second
	LuaCheckAndDelKey     = `
	if(redis.call('get',KEYS[1])==ARGV[1]) then
		return redis.call('del',KEYS[1])
	else
		return 0
	end
`
)

type RedisLock struct {
	key        string
	val        string
	expiration time.Duration
	cli        *redis.Client
	script     *redis.Script
}

func NewRedisLock(key string) *RedisLock {
	val := uuid.New().String()

	return &RedisLock{
		key:        key,
		val:        val,
		expiration: DefaultExpirationTime,
		cli:        cache.RedisV8Client,
		script:     redis.NewScript(LuaCheckAndDelKey),
	}
}

func (r *RedisLock) Lock(ctx context.Context) (bool, error) {
	return r.cli.SetNX(ctx, r.key, r.val, r.expiration).Result()
}

func (r *RedisLock) Unlock(ctx context.Context) error {
	res, err := r.script.Run(context.Background(), r.cli, []string{r.key}, r.val).Int64()
	if err != nil {
		return err
	}
	if res != 1 {
		return errors.New("unlock failed: the lock has been lost or the value does not match")
	}
	return nil
}
```