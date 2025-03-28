---
title: "Hyperf/Crontab 组件源码解析"
date: 2020-06-02T17:29:05+08:00
lastmod: 2020-06-02T17:29:05+08:00
author: ["ZhangYong"]
tags:
- ALL
- PHP
- SWOOLE
- Hyperf
- 底层实现
description: "PHP SWOOLE Hyperf_Crontab_组件源码解析"
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

>前置阅读：[Hyperf/Crontab使用文档](https://hyperf.wiki/#/zh-cn/crontab)
>前置阅读：[Hyperf/Process自定义进程使用文档](https://hyperf.wiki/#/zh-cn/process)
>前置阅读：[Hyperf事件机制](https://hyperf.wiki/#/zh-cn/event)

#### 写在开头
之前做项目用到了Hyperf/Crontab组件来进行秒级的数据清洗，最近又在做定时任务的拆分，于是就打算过一遍组件源码加深理解，顺便构思一下如何在此基础上搭建Hyperf/Crontab的任务调度功能。Crontab本质上是一个随Server启动的自定义进程，所以接下来我们将从启动和执行两个阶段来进行介绍。

#### 定时器的启动
由于Crontab是一个随着Server启动的进程，分析他的生命周期肯定会设计到框架的启动。但本文不是主要介绍Hyperf框架启动源码的，所以我们会简单的过一下涉及到自定义进程启动的框架启动代码。 当我们使用命令 php bin/hyperf.php start 启动hyperf时

```php
$application = $container->get(\Hyperf\Contract\ApplicationInterface::class);
```
在入口文件 hyperf.php 框架会实例化一个 Hyperf\Framework\ApplicationFactory 实例，同时会扫描一遍所有带 @Command 注解或者定义了 commands 配置的地方，实例化一个Symfony\Component\Console\Application对象 （这是一个Symfony的Console命令类用于定义命令触发执行的任务） 。将所有被定义为command的对象类注册到$application对象中，最后在入口文件执行
```php
$application->run();
```
执行所有的Command命令，其中包括 Hyperf\Server\Command\StartServer 服务Server启动类。在这个类里面定义了如果接收到的命令参数包含 start 那么就会执行他的逻辑，即根据 config/autoload/server.php 的配置实例化Hyperf\Server\Server类，在这个过程中会触发BeforeMainServerStart事件，到这里我们即将进入自定义进程启动的核心阶段。
```php
$serverProcesses = $serverConfig['processes'] ?? [];
        $processes = $this->config->get('processes', []);
        $annotationProcesses = $this->getAnnotationProcesses();
        // Retrieve the processes have been registered.
        $processes = array_merge($serverProcesses, $processes, ProcessManager::all(), array_keys($annotationProcesses));
        foreach ($processes as $process) {
            ...
            if ($instance instanceof ProcessInterface) {
                $instance->isEnable() && $instance->bind($server);
            }
        }
```
BootProcessListener监听到BeforeMainServerStart事件的触发会拿到所有 @Process 和 定义在processes配置文件的进程类，执行他们的 isEnable 和 bind 方法。

#### 定时器的执行
在上面我们对自定义进程是如何随框架启动的进行了简单的介绍，接下来我们对本文的主角Crontab自定义进程进行解析。 Hyperf文档介绍在使用Crontab之前需要在 config/autoload/processes.php 内注册一下 Hyperf\Crontab\Process\CrontabDispatcherProcess 自定义进程。那么我们就直接来看一下这个process类里面做了什么事情。 如果你对 Hyperf/Process 组件使用熟悉的话会知道，一个process类主要执行的逻辑都在 handle() 方法 内。

```php
public function handle(): void
    {
        $this->event->dispatch(new CrontabDispatcherStarted());
        while (true) {
            $this->sleep();
            $crontabs = $this->scheduler->schedule();
            while (! $crontabs->isEmpty()) {
                $crontab = $crontabs->dequeue();
                $this->strategy->dispatch($crontab);
            }
        }
    }
```
在handle内首先触发来一个 CrontabDispatcherStarted 事件，目前这个事件无人监听。接下来就是一段时间的协程阻塞，阻塞时间第一次为距离下一次整分钟的秒数，其余的都是60s。关于为什么要使用 \Swoole\Coroutine::sleep 而不是直接 sleep() ，是因为自定义进程默认是一个协程的Server。 接下来

```php
$crontabs = $this->scheduler->schedule();
```
返回一个当前这一分钟该执行的SplQueue队列，队列中的是Crontab对象
```
object(Hyperf\Crontab\Crontab)#46164 (10) {
  ["name":protected]=>
  string(4) "Foo4"
  ["type":protected]=>
  string(8) "callback"
  ["rule":protected]=>
  string(11) "* * * * * *"
  ["singleton":protected]=>
  bool(false)
  ["mutexPool":protected]=>
  string(7) "default"
  ["mutexExpires":protected]=>
  int(3600)
  ["onOneServer":protected]=>
  bool(true)
  ["callback":protected]=>
  array(2) {
[DEBUG] Event Hyperf\Framework\Event\OnPipeMessage handled by Hyperf\Crontab\Listener\OnPipeMessageListener listener.
    [0]=>
    string(16) "App\Task\FooTask"
    [1]=>
    string(7) "execute"
  }
  ["memo":protected]=>
  NULL
  ["executeTime":protected]=>
  object(Carbon\Carbon)#46106 (3) {
    ["date"]=>
    string(26) "2020-06-02 14:15:57.000000"
    ["timezone_type"]=>
    int(3)
    ["timezone"]=>
    string(13) "Asia/Shanghai"
  }
}
```
这个对象中记录了我们目前比较关注的两个关键信息：

1. Execution Callback
2. Execution Time

关于这个crontab对象是如何生成的，这个我们在后面会介绍到。 在我们拿到crontab对象后，我们会把对象发送给在 config/autoload/dependencies.php 定义好的 StrategyInterface 的实现类来进行dispatch，默认指定的是Worker 进程执行策略。
```php
$server = $this->serverFactory->getServer()->getServer();
        if ($server instanceof Server && $crontab->getExecuteTime() instanceof Carbon) {
            $workerId = $this->getNextWorkerId($server);
            $server->sendMessage(new PipeMessage(
                'callback',
                [Executor::class, 'execute'],
                $crontab
            ), $workerId);
        }
```
除Coroutine策略外的dispatch方法是相同的，都是进程间轮训的向WorkID通过 sendMessage 发送 PipeMessage 对象，同时 sendMessage 方法会触发 OnPipeMessage 事件。该事件被 OnPipeMessageListener 监听，会根据 PipeMessage 执行对应的callback函数，即 Executor->execute' 。在该方法中会根据corntab对象的属性定义 Swoole\Timer::after ,根据corontab的executeTime属性定义多少秒后执行callback。
```php
$callback && Timer::after($diff > 0 ? $diff * 1000 : 1, $callback);
```
这样基本就完成了我们一个秒级定时任务的执行。

现在我们回到上面那个关于crontab对象是什么时候生成的问题。我们说过Cornrab本质上就是一个自定义进程，那根据Hyperf/Process的使用说明，所有的自定义进程都继承了 Hyperf\Process\AbstractProcess ，这个类在启动SwooleProcess时会触发 BeforeProcessHandle 事件，在这个事件中会扫描所有的crontab配置和注解，将这些注解进行解析生成crontab对象，存储在crontabs属性中。
```php
public function register(Crontab $crontab): bool
    {
        if (! $this->isValidCrontab($crontab)) {
            return false;
        }
        $this->crontabs[$crontab->getName()] = $crontab;
        return true;
    }
```

以上大致就是一个crontab定时任务的执行流程，当然里面还有很多执行细节和Contab个性化的定义参数由于篇幅有限我们还没来得及介绍，感兴趣的同学可以私下进行阅读，下面我也附上来Hyperf/Crontab的整体执行流程类之间的关系图，方便大家对照着阅读源码。

![](/images/hyperf_crontab.png)