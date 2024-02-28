---
title: "记一次调试nginx vhost的低能操作"
date: 2019-11-13T11:09:04+08:00
lastmod: 2019-11-13T11:09:04+08:00
author: ["zonghay"]
tags:
- Nginx
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
hideSummary: false
ShowWordCount: true
disableShare: true # 底部不显示分享栏
ShowBreadCrumbs: false #顶部显示路径
cover:
    image: ""
    zoom: 10%
---
最近在学习Fast-CGI的协议具体内容，用到TCPDUMP来抓一下nginx发给php-fpm的tcp包。于是自己就要在MAC上搭一个简单的环境，跑个小demo。nginx，php咱都有，回来配个nginx的vhost就成了，谁知道最后在vhost上折腾了半天。难顶。

由于之前的nginx配置都是在公司提供的虚拟机上操作，无非就是把之前项目的conf文件拷贝一份，改一下root路径、server_name，最后reload一下。

由于有看过nginx手册，明白http、server、location各模块的作用以及一些配置，于是我决定自己手写一份。

首先，我现在/Users/zonghay/www/写了一个test.php文件，然后再配置vhost想先看看效果。

我写的第一个server如下：
```
server {
        listen       8848;
        server_name  localhost;
       
        location / {
            root  /Users/zonghay/www/;
            index  index.html index.htm;
        }
}
```
浏览器访问 http://localhost:8848 ,发现报404错误。
404嘛，肯定是找不到，那到底找不到啥呢？想一会儿，我擦，自己写的是test.php文件，nginx配置却是去找index.html和index.htm。赶紧吧php的文件名改了，nginx的配置文件也改成了下面这个

```
server {
        listen       8848;
        server_name  localhost;

        location / {
        	root  /Users/zonghay/www/;
        	index  index.html index.htm index.php;
        }
}
```
重启nginx，发请求
这次不是404了，这次变成白板了，啥也没输出反而倒是把index.php这个文件下载下来了，诶，这是为啥呢？
bing了一下问题，原来是nginx匹配到index.php这个文件后，发现它不是html资源啊，于是nginx就会以application/octet-stream的content-type返回。我们F12看一下http的response即可。

> HTTP/1.1 200 OK
Server: nginx/1.17.3
Date: Wed, 13 Nov 2019 13:25:25 GMT
Last-Modified: Wed, 13 Nov 2019 13:23:34 GMT
Content-Type: application/octet-stream
Content-Length: 18
ETag: "5dcc03d6-12"
Accept-Ranges: bytes

但是要怎么解决的，当然是告诉nginx你要把这个文件交由谁处理啦。那好，咱继续补墙，添加匹配PHP文件的location块，告诉nginx把文件交由监听本机9000端口的php-fpm处理。

```
server {
        listen       8848;
        server_name  localhost;

        location / {
        	root  /Users/zonghay/www/;
        	index  index.html index.htm index.php;
        }
       
        location ~ .*\.php$ {
            fastcgi_pass 127.0.0.1:9000;
        }
}
```

继续重启、刷新

满怀期待发现，这次既不是404，也不是下载了，变成了白板，我这个崩溃啊。但冷静下来分析一下，为什么啥也没返回呢？有可能是php-fpm没处理这个文件或者没处理正确，返回空。经过仔细比对我的虚拟机vhost配置和本季的配置，发现我可能是犯了一个低级的错误。众所周知，php-fpm全称叫啥呢，叫做PHP fast-cgi manager啊，他是一个实现了fast-cgi协议的应用。你nginx告诉人家处理这个文件，但是没告诉人家用啥协议处理啊。这就好比你到了饭店，拎着肉让人家给你炒个菜，但是你没说炒什么才呀，人家肯定不理你呀。于是我边bing，边对比两个配置文件的不同，写了一个最终版

```
server {
        listen       8848;
        server_name  localhost;
        root  /Users/zonghay/www/;
        index  index.html index.htm index.php;

        location ~ .*\.php$ {
            include fastcgi_params;
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_index index.php;
        }
}
```
include的fastcgi_params这个文件里面包含了fast-cgi协议必须的参数K-V键值对，至于这些键值对都有什么作用，还是等我后续研究完fast-cgi协议再来写一篇吧。最后我的小demo是有惊无险的跑通了，看着屏幕上的Hello World的字样，我真是倍感丢人啊，都工作一年了，还能在nginx配置上犯这么低级的错误我也是服了自己。