---
title: "Composer Basic Usage"
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

This file contains the project's dependencies and some other metadata

```json

{
  "require": {
    "monolog/monolog": "1.0.*"
  }
}

```

Version Explanation:

~ indicates that only the last segment of the version number can change (if it's ~x.y then the last segment is y, if it's ~x.y.z then the last segment is z)
~1.2.3 means 1.2.3 <= version number < 1.3.0
~1.2 means 1.2 <= version number < 2.0

^ indicates that all segments except the major version number can change
^1.2.3 means 1.2.3 <= version number < 2.0.0

Special case for versions starting with 0:

^0.3.0 equals 0.3.0 <= version number < 0.4.0 Note: Not < 1.0.0
Because: according to semantic versioning rules, a major version number starting with 0 indicates that it is an unstable version, and minor version numbers are allowed to be incompatible in an unstable state.
So, if you specify a library starting with 0, you must be careful:
Risky way: ~0.1 equals 0.1.0 <= version number < 1.0.0
Safe way: ^0.1 equals 0.1.0 <= version number < 0.2.0


For a detailed structure of composer.json, see the official documentation

[https://docs.phpcomposer.com/04-schema.html\](https://docs.phpcomposer.com/04-schema.html)

#### Composer install

This command is used to download and install dependencies from composer.json into the vendor directory, while generating a composer.lock file.

If a composer.lock file exists in your directory, then composer will download the specified versions from the lock file, ignoring the json file.

So the priority of composer install download is: composer.lock > composer.json

#### composer.lock

After installing dependencies, Composer writes the exact list of versions installed to the composer.lock file. This locks the specific versions of the project.

#### Composer update

This command downloads the latest dependencies based on the composer.json file and updates the version numbers in the composer.lock file.

If you want to install or update a single dependency, you can whitelist them:

>`composer update monolog/monolog [...]`

Note: composer update may update dependencies that you do not want to update. To avoid this, use the command composer update nothing, and composer will only update the dependencies that have changed in the composer.json file.

#### Composer require

This command is used to add new dependencies to the composer.json file in the current directory.

#### Composer search

This command is used to search for dependency packages for the current project, usually only searching for packages on packagist.org.

#### Composer show

This command is used to list all available packages.

#### Composer depends

This command can find out which packages installed in your project are being depended on by other packages and lists them.

#### Composer validate

You should always run the validate command before submitting a composer.json file and creating a tag. It will check if your composer.json file is valid.

#### Composer create-project

Composer creates a new project from an existing package. This is equivalent to executing a git clone or svn checkout command and then installing the package's dependencies into its own vendor directory.

#### Composer dump-autoload

Don't forget to optimize the autoloader when deploying code to production:

>`composer dump-autoload --optimize`

>You can also use --optimize-autoloader when installing packages. Without this option, you may notice a 20% to 25% performance loss.

In PHP-FPM mode, the autoloader can take up a large part of the time for each request (like in Laravel). Using classmaps might not be very convenient during development, but it can still offer the convenience of PSR-0/4 standards while ensuring performance.

--optimize (-o): Converts PSR-0/4 autoloading to classmap to get faster loading speeds. This is especially suitable for production environments, but it may take some time to run, so it is not the default setting yet.