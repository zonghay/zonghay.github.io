---
title: "学习开发一款PHP拓展"
date: 2023-07-13T15:30:11+08:00
lastmod: 2024-03-01T15:30:11+08:00
author: ["ZhangYong"]
tags: # 标签
- ALL
- PHP
- PHP-Extension
- Zend Engine
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
---

## 前置学习

如果我问你，PHP拓展的执行效率比PHP原生代码更高吗？
你会不假思索的回答我。是的。
但是，为什么呢？为什么用C/C++开发的拓展效率就一定高呢？

如果你问Chat-GPT4，它会回答:

> 编译型语言：PHP扩展通常使用C或C++编写，这些都是编译型语言，编译成机器码后执行效率高，可以直接操作硬件和内存。        
直接操作内核数据结构：PHP扩展可以直接访问和操作PHP内核的数据结构和API，避免了PHP脚本中一些额外的运行时开销，如变量查找、类型判断、内存分配等。       
优化的算法和数据结构：使用C或C++编写的PHP扩展可以利用更优化的算法和数据结构，例如使用高效的哈希表、树、图等数据结构，或者使用并行计算、矢量化计算等优化的算法。     
系统级操作：PHP扩展可以直接调用操作系统的API进行系统级的操作，比如文件操作、网络操作、进程管理等，这些操作通常比PHP脚本中的相应操作要高效。

个人总结：程序执行阶段，拓展比应用层语言更贴近操作系统。
PHP可以理解成是一款用C/C++开发的应用，这款应用语法简单，编程效率高，对应的代价便是运行效率低。
为了深入理解这个问题，让我们一起来先来深入学习一下PHP这门编程语言的体系架构。

拓展的优点：
* 优化性能，尤其是计算密集型的逻辑
* 代码保护

### PHP System

![图片](/images/php_system.png)

* Zend Engine：它是PHP的核心，负责解析和执行PHP脚本。所有的PHP代码最终都会被Zend Engine解析成字节码（opcode），然后执行。
* PHP Extension：这是通过Zend API编写的模块，它们可以直接访问和操作Zend Engine的内部数据结构和函数，提供各种PHP函数和类给PHP脚本使用。例如，PDO、mysqli等数据库扩展，GD图形处理扩展等。 注意⚠️：PHP Extension 和 Zend Extension 不是一个东西
* Zend API 和 Zend Extension API 都是Zend Engine提供的接口，专门用于PHP编程语言和PHP Extension调用。 通过它，外部模块可以访问和操作Zend Engine的内部数据结构和函数。
* PHP SAPI（Server API）：这是PHP和外部环境（如Web服务器）之间的接口。不同的SAPI实现可以让PHP运行在不同的环境中，例如Apache模块（mod_php）、CGI、FastCGI、CLI（命令行接口）等。

### PHP Execution Flow

![图片](/images/php_execution.png)

* Scanning（词法分析）：PHP源代码首先会被分解为一系列的标记（token）。这一过程就是词法分析。PHP源代码被读入并转化为一系列的标记，比如变量名、函数名、关键字（如if, else等）、操作符等。 Lex生成的，源文件在 Zend/zend_language_sanner.l
* Parsing（语法分析）：接下来，这些标记会被组织成一种结构化的形式，通常是一棵抽象语法树（AST）。这一过程就是语法分析。在这个阶段，PHP会检查代码的语法是否正确，比如括号是否匹配，语句是否完整等。 yacc生成, 源文件在 Zend/zend_language_parser.y
* Compilation（编译）：紧接着，抽象语法树会被转化为opcode。这一过程就是编译。编译过程中，PHP会进行一些优化，比如常量折叠（constant folding）、死代码消除（dead code elimination）等。 opcode定义的源文件在zend_vm_opcodes.h
* Execution（执行）：最后，opcode会以op array的形式被Zend Engine顺序执行，完成实际的运算和操作。在这个阶段，PHP会进行函数调用、变量查找、表达式求值等操作。

在PHP 8中增加的JIT编译器是作为Opcache的一部分提供的，JIT在Opcache优化后的基础上，结合运行时信息将热点代码（频繁执行的代码）编译成机器码，从而进一步提高执行速度。

### Token

```php
<?php
var_dump(token_get_all('<?php
$name = "PHP";
echo "$name is the best language in the world.";
echo PHP_EOL;'));
```

### AST

```shell
pecl install ast
```

```PHP
<?php
var_dump(ast\parse_code('<?php
$name = "PHP";
echo "$name is the best language in the world.";
echo PHP_EOL;', 70));
```

### Opcodes

```shell
pecl install vld
```

```php
<?php
$name = "PHP";
echo "$name is the best language in the world.";
echo PHP_EOL;
```

```shell
php -dvld.active=1 ./test.php
Finding entry points
Branch analysis from position: 0
1 jumps found. (Code = 62) Position 1 = -2
filename:       /Users/admin/www/swoole-test/vld_test.php
function name:  (null)
number of ops:  6
compiled vars:  !0 = $name
line      #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
    2     0  E >   ASSIGN                                                   !0, 'PHP'
    3     1        NOP
          2        FAST_CONCAT                                      ~2      !0, '+is+the+best+language+in+the+world.'
          3        ECHO                                                     ~2
    4     4        ECHO                                                     '%0A'
   24     5      > RETURN                                                   1

branch: #  0; line:     2-   24; sop:     0; eop:     5; out0:  -2
path #1: 0,
PHP is the best language in the world.
```
我们先来详细的看一下opcode表格里每一列的内容具体代表什么

> * line：这是源代码中的行号。例如，第一个 opcode 对应的源代码在第 2 行。
> * #*：这是 opcode 的序号。例如，ASSIGN 是第 0 个 opcode。
> * E 和 I、O：表示 opcode 的执行入口和跳转出口。
> * op：这是 opcode 的类型。例如，ASSIGN、NOP、FAST_CONCAT、ECHO 和 RETURN。
> * fetch 和 ext：这两列提供了 opcode 的额外信息。在这个例子中，这些列为空。
> * return：这列表示 opcode 的返回值。例如，FAST_CONCAT 的返回值是 ~2。
> * operands：这是 opcode 的操作数。例如，ASSIGN 的操作数是 !0 和 'PHP'，FAST_CONCAT 的操作数是 !0 和 '+is+the+best+language+in+the+world.'。
> * 具体到每个 opcode：
> * 第 0 个 opcode ASSIGN：将字符串 'PHP' 赋值给变量 !0（即 $name）。
> * 第 1 个 opcode NOP：不进行任何操作，通常用于占位或者标记。
> * 第 2 个 opcode FAST_CONCAT：将变量 !0 和字符串 '+is+the+best+language+in+the+world.' 进行连接，结果保存在 ~2 中。
> * 第 3 个 opcode ECHO：输出 FAST_CONCAT 的结果 ~2。
> * 第 4 个 opcode ECHO：输出字符串 '%0A'（即换行符）。
> * 第 5 个 opcode RETURN：返回 1，表示脚本执行成功。

**（以上来自Chat-GPT4的回答）**

## 学习开发一款PHP拓展

下面我们使用两种方式来开发一款PHP拓展，这个拓展的功能就是很简单，就实现一个字符串反转函数。

```shell
php -v
PHP 7.3.33 (cli) (built: Jun 17 2023 07:00:43) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.3.33, Copyright (c) 1998-2018 Zend Technologies
with Zend OPcache v7.3.33, Copyright (c) 1999-2018, by Zend Technologies
```

### 使用C开发拓展

#### 使用PHP官方提供的脚本生成拓展骨架

```shell
cd php-src/ext
php ./ext_skel.php --ext funplus
```

#### 声明 && 实现函数方法

我们准备在funplus拓展里实现一个简单的字符串翻转方法，函数参数是一个字符串，返回值是翻转后的字符串。
在php_funplus.h声明函数方法

```c
PHP_FUNCTION(funplus_reverse_string);
```

在funplus.c注册方法、实现方法逻辑

```c
/* {{{ arginfo
 */
ZEND_BEGIN_ARG_INFO(arginfo_funplus_reverse_string, 0)
ZEND_ARG_INFO(0, str)
ZEND_END_ARG_INFO()
/* }}} */

/* {{{ funplus_functions[]
 */
static const zend_function_entry funplus_functions[] = {
    PHP_FE(funplus_test1, arginfo_funplus_test1)
        PHP_FE(funplus_test2, arginfo_funplus_test2)
            PHP_FE(funplus_reverse_string, arginfo_funplus_reverse_string)
                PHP_FE_END};
/* }}} */

// arginfo_funplus_test1 和 arginfo_funplus_test2 是骨架自带的


/* {{{ string funplus_reverse_string( [ string $var ] )
 */
PHP_FUNCTION(funplus_reverse_string)
{
    char *var;
    size_t var_len;
    zend_string *retval;

    ZEND_PARSE_PARAMETERS_START(1, 1)
    Z_PARAM_STRING(var, var_len)
    ZEND_PARSE_PARAMETERS_END();

    retval = zend_string_alloc(var_len, 0);
    for (size_t i = 0; i < var_len; i++)
    {
        ZSTR_VAL(retval)
        [i] = var[var_len - i - 1];
    }
    ZSTR_VAL(retval)
    [var_len] = '\0';

    RETURN_STR(retval);
}
/* }}}*/
```

#### 编译 && 安装拓展

```shell
// funplus拓展源码路径
phpize
./configure
sudo make && sudo make install
```

#### 和PHP原生代码的性能对比

```php
<?php
const IntervalTimes = 1000000;
$var = "";

$start = microtime(true);
for ($i=0 ;$i<IntervalTimes; $i++) {
$var = funplus_reverse_string("42tPJzmsqSbHvVRUGTrapCi4A3mmaohK");
}
var_dump("Extension Total time:" . (microtime(true) - $start));


$start = microtime(true);
for ($i=0 ;$i<IntervalTimes; $i++) {
$var = reverseString("42tPJzmsqSbHvVRUGTrapCi4A3mmaohK");
}
var_dump("PHP Total time:" . (microtime(true) - $start));


function reverseString($str): string
{
$reversed = "";
for ($i = strlen($str) - 1; $i >= 0; $i--) {
$reversed .= $str[$i];
}
return $reversed;
}

//string(38) "Extension Total time:0.038073062896729"
//string(30) "PHP Total time:1.0140709877014"
```

### 使用Zephir开发拓展

> With Zephir, you can implement object-oriented libraries/frameworks/applications that can be used from PHP, gaining important seconds that can make your application faster while improving the user experience.

#### 安装

```shell
pecl install zephir_parser
```
OR

```shell
Bash composer global require "phalcon/zephir:0.17.0"
```
注：需要把zephir的可执行程序加载到$PATH中
验证是否安装成功

生成拓展骨架
```shell
zephir init foo
```

#### 编写拓展代码

```shell
// foo/foo/str.zep
namespace Foo;

class Str
{
public static function bar(string! text) -> string
{
var reversedText, i, len;
let reversedText = "";
let len = strlen(text);

for i in range(len - 1, 0, -1) {
let reversedText .= chr(text[intval(len - i - 1)]);
}

return reversedText;
}
}
```

#### 编译 && 测试

```shell
zephir build
```

```php
<?php

const IntervalTimes = 1000000;
$var = "";

$start = microtime(true);
for ($i = 0; $i < IntervalTimes; $i++) {
$var = Foo\Str::bar("42tPJzmsqSbHvVRUGTrapCi4A3mmaohK");
}
var_dump("Foo Extension Total time:" . (microtime(true) - $start));


$start = microtime(true);
for ($i = 0; $i < IntervalTimes; $i++) {
$var = reverseString("42tPJzmsqSbHvVRUGTrapCi4A3mmaohK");
}
var_dump("PHP Total time:" . (microtime(true) - $start));


function reverseString($str): string
{
$reversed = "";
for ($i = strlen($str) - 1; $i >= 0; $i--) {
$reversed .= $str[$i];
}
return $reversed;
}

// string(42) "Foo Extension Total time:0.072835922241211"
// string(31) "PHP Total time:0.91381096839905"
```

### 使用PHP-CPP开发拓展

The PHP-CPP library is a C++ library for developing PHP extensions. It offers a collection of well documented and easy-to-use classes that can be used and extended to build native extensions for PHP. The full documentation can be found on http://www.php-cpp.com .

在我本机MacOS的支持不好，make编译不通过，卒。

### 使用PHP-X开发拓展 (By 韩天峰)

```shell
git clone https://github.com/swoole/PHP-X.git
cd console && \
echo "composer update" && \
composer update && \
cd ../ && \
php -d phar.readonly=off script/pack.php --disable-gz
sudo cp bin/phpx /usr/local/bin
phpx create cpp_ext
```
也卒😅

### PHP-FFI - 一种使用PHP代码调用C库的方式

For PHP, FFI opens a way to write PHP extensions and bindings to C libraries in pure PHP.
是的，FFI提供了高级语言直接的互相调用，而对于PHP来说，FFI让我们可以方便的调用C语言写的各种库。
其实现有大量的PHP扩展是对一些已有的C库的包装，比如常用的mysqli, curl, gettext等，PECL中也有大量的类似扩展。而有了FFI以后，我们就可以直接在PHP脚本中调用C语言写的库中的函数了。

```php
<?php
const CURLOPT_URL = 10002;
const CURLOPT_SSL_VERIFYPEER = 64;

$libcurl = FFI::cdef(<<<CTYPE
void *curl_easy_init();
int curl_easy_setopt(void *curl, int option, ...);
int curl_easy_perform(void *curl);
void curl_easy_cleanup(void *handle);
CTYPE
, "libcurl.so"
);

$url = "https://www.laruence.com/2020/03/11/5475.html";

$ch = $libcurl->curl_easy_init();
$libcurl->curl_easy_setopt($ch, CURLOPT_URL, $url);
$libcurl->curl_easy_setopt($ch, CURLOPT_SSL_VERIFYPEER, 0);

$libcurl->curl_easy_perform($ch);

$libcurl->curl_easy_cleanup($ch);
```

## 参考

[PHP 7 Virtual Machine](https://www.npopov.com/2017/04/14/PHP-7-Virtual-machine.html)      
[PHP 运行机制与原理 – 怼码人生](https://blog.duicode.com/2803.html)     
[Table Of Contents — PHP Internals Book](https://www.phpinternalsbook.com/index.html#)     
[深入浅出PHP(Exploring PHP) - 风雪之隅](https://www.laruence.com/2008/08/11/147.html)       
[深入理解PHP原理之Opcodes - 风雪之隅](https://www.laruence.com/2008/06/18/221.html)
[解析PHP原生扩展开发 - 掘金](https://juejin.cn/post/7036991318991749128)
[Registering and using PHP functions — PHP Internals Book](https://www.phpinternalsbook.com/php7/extensions_design/php_functions.html)
[如何使用C++开发PHP扩展(下) - WiFeng的博客](https://521-wf.com/archives/245.html)
[用C++开发PHP扩展](https://segmentfault.com/a/1190000005016605?utm_source=sf-similar-article)
[Zephir Documentation](https://docs.zephir-lang.com/0.12/zh-cn/motivation)