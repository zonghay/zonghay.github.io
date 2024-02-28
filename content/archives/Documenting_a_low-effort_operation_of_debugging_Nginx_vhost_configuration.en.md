---
title: "Documenting a low-effort operation of debugging an Nginx vHost configuration."
date: 2019-11-13T11:09:04+08:00
lastmod: 2019-11-13T11:09:04+08:00
author: ["zonghay"]
keywords:
-
categories: # 没有分类界面可以不填写
- 
tags: # 标签
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
    zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---
Recently, I've been studying the specifics of the Fast-CGI protocol and used TCPDUMP to capture the TCP packets sent from Nginx to PHP-FPM. So, I set up a simple environment on my MAC to run a small demo. I already had Nginx and PHP, just had to configure an Nginx vhost and it should be done, but who knew I'd spend half a day messing with the vhost. Tough times.

Since all my previous Nginx configurations were done on virtual machines provided by the company, it was just a matter of copying a conf file from a previous project, changing the root path and server_name, and then reloading.

Having read the Nginx manual, I understood the roles of the http, server, and location modules, as well as some configurations, so I decided to write one myself.

First, I wrote a test.php file in /Users/zonghay/www/, and then configured the vhost to see the effect.

My first server configuration was as follows:
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
Accessing http://localhost:8848 in the browser, I encountered a 404 error.
A 404 error means something couldn't be found. But what exactly? After thinking for a while, I realized that I wrote a test.php file, but the Nginx configuration was looking for index.html and index.htm. I quickly changed the PHP file name and updated the Nginx configuration file to this:

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
Restart Nginx and send a request.
This time it wasn't a 404, but a blank page, and instead of displaying anything, it downloaded the index.php file. Eh, why is that?
After a Bing search, I found out that when Nginx matches the index.php file, it realizes it's not an HTML resource, so it returns the file with a content-type of application/octet-stream. We can see the HTTP response in the F12 developer tools.

> HTTP/1.1 200 OK
Server: nginx/1.17.3
Date: Wed, 13 Nov 2019 13:25:25 GMT
Last-Modified: Wed, 13 Nov 2019 13:23:34 GMT
Content-Type: application/octet-stream
Content-Length: 18
ETag: "5dcc03d6-12"
Accept-Ranges: bytes

But how to solve this? Of course, you need to tell Nginx to hand over the file to someone else for processing. So, let's continue building the wall, add a location block to match PHP files, and tell Nginx to hand the file over to PHP-FPM listening on port 9000 of the local machine.

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

Continue with the restart and refresh.

With full of anticipation, I found that it was neither a 404 nor a download this time, but a blank page. I was devastated. But after calming down and analyzing it, why didn't it return anything? It could be that PHP-FPM didn't process the file or didn't process it correctly, returning empty. After carefully comparing my virtual machine's vhost configuration with this season's configuration, I realized I might have made a basic mistake. As we all know, what is the full name of PHP-FPM? It's called PHP Fast-CGI Manager, an application that implements the Fast-CGI protocol. You tell Nginx to process the file, but you don't tell it what protocol to use. It's like going to a restaurant, carrying meat and asking them to stir-fry a dish, but if you don't say what dish, they'll definitely ignore you. So, I Binged and compared the differences between the two configuration files, and wrote a final version.

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
The included fastcgi_params file contains the necessary K-V key-value pairs for the Fast-CGI protocol. As for the functions of these key-value pairs, I will write another article after I finish researching the Fast-CGI protocol. Finally, my little demo ran successfully with some scares, but seeing the words "Hello World" on the screen, I felt quite embarrassed. I've been working for a year, and I can still make such a basic mistake in Nginx configuration.