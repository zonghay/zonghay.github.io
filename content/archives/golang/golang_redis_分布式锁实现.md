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
	RenewInterval         = DefaultExpirationTime / 2 // 续租间隔为锁过期时间的一半
	LuaCheckAndDelKey     = `
	if(redis.call('get',KEYS[1])==ARGV[1]) then
		return redis.call('del',KEYS[1])
	else
		return 0
	end
`
	LuaCheckAndRenewKey = `
	if(redis.call('get',KEYS[1])==ARGV[1]) then
		return redis.call('expire',KEYS[1],ARGV[2])
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
	script     *redis.Script  // 用于解锁的 Lua 脚本
	renewScript *redis.Script // 用于续租的 Lua 脚本
	stopChan   chan struct{}  // 用于停止续租 goroutine
}

func NewRedisLock(key string) *RedisLock {
	val := uuid.New().String()

	return &RedisLock{
		key:        key,
		val:        val,
		expiration: DefaultExpirationTime,
		cli:        cache.RedisV8Client,
		script:     redis.NewScript(LuaCheckAndDelKey),
		renewScript: redis.NewScript(LuaCheckAndRenewKey),
		stopChan:   make(chan struct{}),
	}
}

func (r *RedisLock) Lock(ctx context.Context) (bool, error) {
	success, err := r.cli.SetNX(ctx, r.key, r.val, r.expiration).Result()
	if err != nil {
		return false, err
	}
	if success {
		go r.startRenewal(ctx) // 启动续租 goroutine
	}
	return success, nil
}

func (r *RedisLock) Unlock(ctx context.Context) error {
	close(r.stopChan) // 停止续租 goroutine

	res, err := r.script.Run(ctx, r.cli, []string{r.key}, r.val).Int64()
	if err != nil {
		return err
	}
	if res != 1 {
		return errors.New("unlock failed: the lock has been lost or the value does not match")
	}
	return nil
}

// 启动续租 goroutine
func (r *RedisLock) startRenewal(ctx context.Context) {
	ticker := time.NewTicker(RenewInterval)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			// 使用 Lua 脚本原子化检查锁是否持有并续租
			res, err := r.renewScript.Run(ctx, r.cli, []string{r.key}, r.val, int(r.expiration/time.Second)).Result()
			if err != nil || res == 0 {
				return // 锁已丢失或续租失败，停止续租，打印错误日志
			}
		case <-r.stopChan:
			return // 收到停止信号，退出续租 goroutine
		}
	}
}
```