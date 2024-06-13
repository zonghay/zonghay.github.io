---
title: "Composer 基本使用"
date: 2019-12-19T11:09:04+08:00
lastmod: 2019-12-19T11:09:04+08:00
author: ["ZhangYong"]
tags:
- ALL
- Composer
- PHP
description: "composer PHP"
weight:
slug: ""
draft: false # 是否为草稿
comments: true # 本页面是否显示评论
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hideMeta: false # 是否隐藏文章的元信息，如发布日期、作者等
hideSummary: false
ShowWordCount: true
disableShare: true # 底部不显示分享栏
ShowBreadCrumbs: false #顶部显示路径
---

#### composer.json

该文件包含了项目的依赖和其他一些源数据。

```json

{
  "require": {
    "monolog/monolog": "1.0.*"
  }
}

```

版本说明：

~表示版本号只能改变最末尾那段（如果是 ~x.y 末尾就是 y，如果是 ~x.y.z 末尾就是 z）
~1.2.3 代表 1.2.3 <= 版本号 < 1.3.0
~1.2   代表  1.2 <= 版本号 <2.0

^表示除了大版本号以外，小版本号和补丁版本号都可以变
^1.2.3 代表 1.2.3 <= 版本号 < 2.0.0

特殊情况0开头的版本号：
^0.3.0 等于 0.3.0 <= 版本号 <0.4.0  注意：不是 <1.0.0
因为：semantic versioning 的规定是，大版本号以 0 开头表示这是一个非稳定版本（unstable），如果处于非稳定状态，小版本号是允许不向下兼容的，
所以如果你要指定 0 开头的库那一定要注意：
危险写法：~0.1 等于 0.1.0 <= 版本号 <1.0.0
保险写法：^0.1 等于 0.1.0 <= 版本号 <0.2.0



composer.json 结构详细见官方说明

[https://docs.phpcomposer.com/04-schema.html\](https://docs.phpcomposer.com/04-schema.html)

#### Composer install

改名了用于将composer.json中的依赖下载安装到vendor目录下，同时生成一个composer.lock文件。

如果你的目录存在composer.lock文件，那么composer将会根据lock文件下载指定版本，忽略json文件。

于是composer install下载的优先级：composer.lock > composer.json

#### composer.lock

在安装依赖后，Composer 将把安装时确切的版本号列表写入 composer.lock 文件。这将锁定改项目的特定版本。

#### Composer update

该命令根据composer.json文件匹配最新的依赖进行下载，并将版本号更新到composer.lock文件

如果只想安装或更新一个依赖，你可以白名单它们：

>`composer update monolog/monolog [...]`

注：composer update 可能会将你并不想更新的依赖一同更新。这事使用composer update nothing命令，composer只会把composer.json文件改动的依赖进行update。

#### Composer require

改名了用于增加新的依赖到当前目录的composer.json文件

#### Composer search

该命令用于为当前项目搜索依赖包，通常它只搜索 packagist.org 上的包

#### Composer show

该命令用于列出所有可用的软件包

#### Composer depends

该命令可以查出已安装在你项目中的某个包，是否正在被其它的包所依赖，并列出他们。

#### Composer validate

在提交 composer.json 文件，和创建 tag 前，你应该始终运行 validate 命令。它将检测你的 composer.json 文件是否是有效的

#### Composer create-project

Composer 从现有的包中创建一个新的项目。这相当于执行了一个 git clone 或 svn checkout 命令后将这个包的依赖安装到它自己的 vendor 目录。

#### Composer dump-autoload

在部署代码到生产环境的时候，别忘了优化一下自动加载：

>`composer dump-autoload --optimize`

>安装包的时候可以同样使用--optimize-autoloader。不加这一选项，你可能会发现20%到25%的性能损失。

在PHP-FPM模式下，autoloader 会占用每个请求的很大一部分时间（比如laravel），使用 classmaps 或许在开发时不太方便，但它在保证性能的前提下，仍然可以获得 PSR-0/4 规范带来的便利。

--optimize (-o): 转换 PSR-0/4 autoloading 到 classmap 获得更快的载入速度。这特别适用于生产环境，但可能需要一些时间来运行，因此它目前不是默认设置