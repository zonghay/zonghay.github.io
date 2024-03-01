---
title: "概述PHP、Swoole和Hyperf(个人理解版)"
date: 2023-06-09T17:50:47+08:00
lastmod: 2023-06-29T17:50:47+08:00
author: ["ZhangYong"]
tags: # 标签
- PHP
- SWOOLE
- Hyperf
description: "概述PHP、Swoole和Hyperf"
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

## PHP-FPM
### 1.1 传统Nginx+FPM架构
#### 1.1.1 PHP-FPM 并发模型
![图片](/images/php-fpm.png)
图1.1.1 一个HTTP请求的流转过程在网络应用场景下，PHP并没有像Golang那样实现http网络库，而是实现了FastCGI协议，然后与web服务器配合实现了http的处理。 web服务器来处理http请求，然后将解析的结果再通过FastCGI协议转发给处理程序，处理程序处理完成后将结果返回给web服务器，web服务器再返回给用户。

PHP-FPM是经典的多进程并发模型，Master/Worker模型。Master进程与Worker进程之间不会直接进行通信，Master进程只负责Fork和管理子进程，网络请求由子进程处理，一个Worker进程同时只能处理一个请求。Master通过共享内存获取worker进程的信息，比如worker进程当前状态、已处理请求数等，当master进程要杀掉一个worker进程时则通过发送信号的方式通知worker进程。

#### 1.1.2 PHP-FPM 初始化启动 及 Worker 请求处理
PHP-FPM从初始化启动到Worker请求处理大概涉及到以下步骤：
* fpm_init()  Master读取php-fpm.conf文件初始化内存配置变量、创建管道、套接字、启动事件管理
* fpm_run()  Master进程fork出子进程后阻塞，worker进程去accept请求，执行php脚本
  * (1)等待请求： worker进程阻塞在fcgi_accept_request()等待请求；
  * (2)解析请求： fastcgi请求到达后被worker接收，然后开始接收并解析请求数据，直到request数据完全到达；
  * (3)请求初始化： 执行php_request_startup()，此阶段会调用每个扩展的：PHP_RINIT_FUNCTION()；
  * (4)编译、执行： 由php_execute_script()完成PHP脚本的编译、执行；
  * (5)关闭请求： 请求完成后执行php_request_shutdown()，此阶段会调用每个扩展的：PHP_RSHUTDOWN_FUNCTION()，然后进入步骤(1)等待下一个请求。

![图片](/images/php_cycle_life.png)
图1.1.2 php生命周期生成的语法树和opcode，同一个PHP脚本每次运行的结果都是一样的，在PHP-FPM模式下，每次请求都要处理一遍，是对系统资源极大的浪费

### 1.2 Laravel框架的性能问题
laravel是fpm社区里面非常受欢迎的一款web框架，结合composer管理开发组件，可以帮助开发者在框架提供的多种优秀组件（ORM，Router，Middleware，Artisan等）和解决方案在PSR-4规范下高效的进行Web服务端开发。所以有人说，如果Wordpress让php焕发第一春，那么Laravel+Composer让PHP焕发第二春。

虽然laravel框架开发起来简单高效，中文社区也非常活跃，生态丰富。但是用laravel开发的应用性能问题却一直被反复提及。

造成Laravel性能问题的主要原因有以下几点：
* 框架代码的深度封装，导致把fpm模式下框架启动需要加载大量类和文件的性能问题放大（需要new太多类，每次new一个类都要autoloader找到类所在文件再require）
* 每一次请求都会将所有的路由和配置全部加在到内存中，即便这次请求可能用不到（这其实也要归咎于fpm一次请求结束后就会销毁内存的特性）
* 在框架启动时会调用composer autoload，按照classmap、prs-4、psr-0等规范生成所有类的引入路径

所以针对Laravel进行性能优化的方向也就是针对以上几点来进行的：
* 服务器启用 PHP OPcache 扩展缓存 opcodephp 
* artisan route:cache php artisan config:cache等缓存命令
* 通过 composer install --optimize-autoloader --no-dev 初始化项目依赖，以便加速 Composer 定位指定类对应的加载文件，同时不安装开发环境使用的依赖

### 1.3 FPM架构的局限
并发模型带来的单台服务器性能局限只能开发HTTP服务器，如果需要WebSocket，TCP等其他协议的服务器怎么办服务处理中的内存状态无法持久化，绝大多数只能开发无状态服务，有状态服务需要借助本地文件或者mmap方式来实现Swoole-PHP的高性能通信引擎扩展

## SWOOLE-PHP的高性能通信引擎扩展
### 2.1 什么是Swoole
Swoole 是一个使用 C++ 语言编写的基于异步事件驱动和协程的并行网络通信引擎，为 PHP 提供协程、高性能网络编程支持。提供了多种通信协议的网络服务器和客户端模块，可以方便快速的实现 TCP/UDP服务、高性能Web、WebSocket服务、物联网、实时通讯、游戏、微服务等，使 PHP 不再局限于传统的 Web 领域。

Swoole的出现可以打开上述FPM所面临的局限Swoole是通过cli模式启动的常驻内存服务，通过自身提供的异步/协程服务端可以提高单机服务的并发性能（即便是同步I/O的服务性能也会有较大提升）Swoole提供了多种通信协议的网络服务器，包括TCP、UDP、HTTP、WebSocket、MQTTSwoole提供该性能共享内存SwooleTable、进程间无锁计数器Atomic、进程间API等方式保证有状态服务进程间的同步底层hook原生PHP函数，使其能够更好的运行在协程server内（底层替换了ZendVM Stream的函数指针，所有使用php_stream进行socket操作均变成协程调度的异步IO）

### 2.2 如何使用Swoole
#### 2.2.1 协程风格HTTP服务端
```php
<?php
use Swoole\Coroutine\Http\Server;
use function Swoole\Coroutine\run;

run(function () {
    $server = new Server('127.0.0.1', 9502, false);
    $server->handle('/test', function ($request, $response) {
        Co::sleep(1);
        $response->end("test");
    });
    $server->handle('/stop', function ($request, $response) use ($server) {
        $response->end("<h1>Stop</h1>");
        $server->shutdown();
    });
    $server->start();
});
```

#### 2.2.2 协程风格TCP服务端
```php
<?php
use Swoole\Process;
use Swoole\Coroutine;
use Swoole\Coroutine\Server\Connection;

//多进程管理模块
$pool = new Process\Pool(2);
//让每个OnWorkerStart回调都自动创建一个协程
$pool->set(['enable_coroutine' => true]);
$pool->on('workerStart', function ($pool, $id) {
    //每个进程都监听9501端口
    $server = new Swoole\Coroutine\Server('127.0.0.1', 9501, false, true);

    //收到15信号关闭服务
    Process::signal(SIGTERM, function () use ($server) {
        $server->shutdown();
    });

    //接收到新的连接请求 并自动创建一个协程
    $server->handle(function (Connection $conn) {
        while (true) {
            //接收数据
            $data = $conn->recv(10);

            if ($data === '' || $data === false) {
                $errCode = swoole_last_error();
                $errMsg = socket_strerror($errCode);
                echo "errCode: {$errCode}, errMsg: {$errMsg}\n";
                $conn->close();
                break;
            }

            //发送数据
            $conn->send('hello');

            Coroutine::sleep(1);
        }
    });

    //开始监听端口
    $server->start();
});
$pool->start();
```
### 2.3 Swoole并发模型
![图片](/images/swoole_process_model.png)
Swoole服务端进程线程结图
#### 2.3.1 swoole_process
SWOOLE_PROCESS 模式的 Server 所有客户端的 TCP 连接都是和主进程建立的，内部实现比较复杂，用了大量的进程间通信、进程管理机制。适合业务逻辑非常复杂的场景。
Swoole 提供了完善的进程管理、内存保护机制。 在业务逻辑非常复杂的情况下，也可以长期稳定运行。
优点：
* 连接与数据请求发送是分离的，不会因为某些连接数据量大某些连接数据量小导致 Worker 进程不均衡Worker 
* 进程发生致命错误时，连接并不会被切断
* 可实现单连接并发，仅保持少量 TCP 连接，请求可以并发地在多个 Worker 进程中处理

缺点：
* 存在 2 次 IPC 的开销，master 进程与 worker 进程需要使用 unixSocket 进行通信

#### 2.3.2 swoole_base
SWOOLE_BASE 这种模式就是传统的异步非阻塞 Server。与 Nginx 和 Node.js 等程序是完全一致的。
BASE 模式下没有 Master 进程的角色，只有 Manager 进程的角色。
每个 Worker 进程同时承担了 SWOOLE_PROCESS 模式下 Reactor 线程和 Worker 进程两部分职责。

优点：
* BASE 模式没有 IPC 开销，性能更好
* BASE 模式代码更简单，不容易出错

缺点
* TCP 连接是在 Worker 进程中维持的，所以当某个 Worker 进程挂掉时，此 Worker 内的所有连接都将被关闭
* 少量 TCP 长连接无法利用到所有 Worker 进程
* TCP 连接与 Worker 是绑定的，长连接应用中某些连接的数据量大，这些连接所在的 Worker 进程负载会非常高。但某些连接数据量小，所以在 Worker 进程的负载会非常低，不同的 Worker 进程无法实现均衡。
* 如果回调函数中有阻塞操作会导致 Server 退化为同步模式，此时容易导致 TCP 的 backlog 队列塞满问题。

### 2.4 Swoole Coroutine和 Go Goroutine的区别
* swoole的协程需要再协程上下文中使用
* swoole的协程是通过hook底层php函数实现的，go是原生支持的
* swoole是单线程协程，同一时间只会调度一个协程，而go是多线程实现的

```php
<?php
$n = 4;
for ($i = 0; $i < $n; $i++) {
    go(function () use ($i) {
        // 模拟 IO 等待
        Co::sleep(1);
        echo microtime(true) . ": hello $i " . PHP_EOL;
    });
};
echo "hello main \n";
\Swoole\Event::wait();2.5 Swoole 和 Go HTTP服务器性能对比2.5.1 压测代码Swoole HTTP 代码如下使用Swoole/Server 设置 2个worker进程$http = new Swoole\Http\Server("127.0.0.1", 9502, SWOOLE_PROCESS);
$http->set(array(
    'worker_num'    => 2,     // worker process num
));
$http->on('request', function ($request, $response) {
    $response->end("test");
});
$http->start();Go HTTP 代码如下package main

import (
    "fmt"
    "time"
    "net/http"
    "log"
)

func test(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "test") //这个写入到w的是输出到客户端的
}

func main(){
    http.HandleFunc("/test", test) //设置访问的路由
    err := http.ListenAndServe(":9502", nil) //设置监听的端口
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}
```
2.5.2 压测命令 
```shell
wrk -t8 -c200 -d30s --latency  "http://127.0.0.1:9502/test"2.5.3 压测结果Running 30s test @ http://127.0.0.1:9502/test
  8 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.42ms  318.83us  11.60ms   75.70%
    Req/Sec    17.62k     1.70k   20.29k    62.75%
  Latency Distribution
     50%    1.35ms
     75%    1.58ms
     90%    1.82ms
     99%    2.29ms
  4222085 requests in 30.10s, 628.13MB read
  Socket errors: connect 0, read 60, write 0, timeout 0
Requests/sec: 140262.55
Transfer/sec:     20.87MBRunning 30s test @ http://127.0.0.1:9502/test
  8 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.29ms    0.90ms  33.35ms   86.15%
    Req/Sec    10.99k     1.74k   17.86k    71.76%
  Latency Distribution
     50%    2.36ms
     75%    2.70ms
     90%    2.97ms
     99%    3.50ms
  2632268 requests in 30.10s, 301.24MB read
  Socket errors: connect 0, read 51, write 0, timeout 0
Requests/sec:  87441.90
Transfer/sec:     10.01MB
```
注：压测结果上Swoole和Golang的差距主要体现在
go是使用单线程eventloop处理IO事件，多线程实现协程调度，执行用户层代码
swoole使用多线程eventloop处理IO事件，多进程执行用户层php代码

## Hyperf-基于Swoole的PHP协程框架
Hyperf 的出现是为了解决Swoole门槛高，上手难的问题。结合Composer组件，帮助使用者以低成本打造一套高性能的灵活的应用服务。

### 3.1 Hyperf特性
* Hyperf内置大量的协程组件，以保证请求不回退化到阻塞模式。
* Hyperf支持多端口监听，可同时启动HTTP和WebSocket端口
* 支持jsonRPC，gRPC微服务配套设施（熔断、降级、配置中心、链路追踪、Metric监控等）
* 因为是常驻内存的服务，所以引入连接池（Mysql、Redis、GuzzleHttp）
* ORM 基于Laravel ORM进行改造，从Laravel/Lumen 移植过来非常顺畅

### 3.2 Hyperf框架核心
#### 3.2.1 依赖注入
Hyperf/DI 是框架中管理对象类创建和依赖关系的组件，也是实现 注解 和 AOP 功能的关键。DI中管理的对象都是用于服务长生命周期的单例对象，短生命周期使用make()方法创建。
依据创建对象的需求不同可分为以下三种注入方式
* 简单对象注入
* 抽象对象注入
* 工厂对象注入

#### 3.2.2 注解 和 AOP
Hyperf基于AOP和注解实现了诸多框架基础功能，如
* 依赖注入 @Inject()
* 路由定义 @AutoController()
* 中间件 @Middleware(FooMiddleware::class) 
* 缓存 @Cacheable(prefix="user", ttl=9000, listener="user-update")

#### 3.2.3 事件机制 和 生命周期
Hyperf的事件机制存在于框架启动时，能够帮助用户更好的和框架配合完成逻辑的解耦，例如DBqueryLisenner。在处理每个连接时，会默认创建一个协程去处理，主要体现在 onRequest、onReceive、onConnect 事件，所以可以理解为每个请求都是一个协程，由于创建协程也是个常规操作，所以一个请求协程里面可能会包含很多个协程，同一个进程内协程之间是内存共享的，但调度顺序是非顺序的，且协程间本质上是相互独立的没有父子关系，所以对每个协程的状态处理都需要通过 协程上下文 来管理。

![图片](/images/hyperf_events.png)

## 最后
这篇文章不是不是为了比较一个高低，不是说FPM模式下的开发方式和项目就不好，而是想表达fpm可能已经不适合当下追求并发，尽可能榨取服务器CPU性能的要求了。Swoole在php原生语法简单高效的基础上，为php搭建了基于协程（异步非阻塞）的运行模式，再加上Hyperf这样组件化的成熟框架，使得上手Swoole不在困难。让大家了解到这种开发方式的可能性就可以了。
