---
title: php-fpm模式下的优化思路与实践
date: 2022-01-21 13:28:06
categories: php
tags:
- fpm
- 优化
---

# php-fpm模式下的优化思路与实践

一般来说，计算机体系中所谓的优化，无非三类，串行改并行，同步转异步，以及加缓存，减少执行。

## 串行改并行

程序执行总时间= (不可并行子模块...) 相加

在 `php-fpm` 模式下 每个请求都是单进程单线程去处理，而 `pcntl`（`php` 进程处理扩展）在 `fpm` 模式下无法使用。所以就有了各种勉强的处理方案。

### 基于`redis || 各种mq` 队列的并行操作

具体来说 就是利用 中间件 来对外发布任务，由消费者（比如用 `golang` 监听 `redis` ，开协程去处理多个 `task`）来进行并行的消费 , 然后把结果传回来。

那么你肯定也会有疑问，我直接`http`调用`go`的服务然后响应不行吗？当然可以，但是 `curl` 在 `php-fpm` 中是阻塞操作，用队列的方式更加灵活且不会堵塞。

```php
$randKey = Str::random();
$redis->lpush($key,$task1.'-'.$task2.':'.$rankKey);
// 堵塞等待消费者将 task 完成 然后把结果 塞入  $key.$rankKey 
$data = $redis->brPop($key.$randKey, 10);
......//其他逻辑
```

### 基于 `popen || proc_open || exec ` 的并行操作

```php
    for ($i=0; $i < 10; $i++) {
        $pipe[$i] = popen('php ./exec.php ', 'w');
    }
    for ($i=0; $i < 10; $i++) {
        pclose($pipe[$i]);
    }
```

这种用 `shell` 的方式虽然看起来确实是简单了，但是存在两个问题，一个是函数的平台限制，第二是安全性较低，但不得不说 shell 有些特殊场景下确实很有用。

## 同步转异步

在这类操作里面 除了串行改并行的的两种方式可以实现（需要额外的进程去处理），将执行操作异步，日常用的最多的是 `fastcgi_finish_request()` ，简单实在看下在 `laravel` 中的使用

```php
# public/index.php

$response = $kernel->handle(
    $request = Request::capture()
)->send();

$kernel->terminate($request, $response);

# vendor/symfony/http-foundation/Response.php:391
public function send()
{
    $this->sendHeaders();
    $this->sendContent();

    if (\function_exists('fastcgi_finish_request')) {
    	fastcgi_finish_request();
    } elseif (\function_exists('litespeed_finish_request')) {
    	litespeed_finish_request();
    } elseif (!\in_array(\PHP_SAPI, ['cli', 'phpdbg'], true)) {
    	static::closeOutputBuffers(0, true);
    }

    return $this;
}
```

其实 `tp` 里面也一样使用了这个函数来进行收尾，优点很明显，代码还是在同一个进程内，无需额外的配置，开箱即用。缺点的话，就是在响应之后的逻辑比较重的情况下，fpm 的进程数会随着请求量的增加而增加。也会受到脚本最大执行时间的限制，只适合一些比较轻量级的异步操作，重要的业务逻辑，建议还是以 `MQ` 为主。

## fpm 中的性能优化实践

缓存的重要性就不说了，在 `php-fpm` 的生命周期中，最重要的缓存就是 `OPcache` ，特别是 `laravel` 这种文件特别多，逻辑很重的框架 开不开 `OPcache` ，是两种体验，相关的配置 可以看我之前写过的[ 博文 ]( https://blogxy.cn/2022/01/20/php8-1%E6%96%B0%E7%89%B9%E6%80%A7%E4%B8%8E%E9%83%A8%E5%88%86%E4%BD%BF%E7%94%A8%E5%AE%9E%E8%B7%B5/#php8-%E7%9A%84-opcache-%E4%B8%8E-jit-Just-In-Time ) 

- 现在有用 `laravel` 框架写了一个登录后的用户 获取信息的 `API`, 使用 `JMeter` 进行测试

  ```shell
  # 本地 fpm 的配置
  pm = dynamic
  pm.max_children = 5
  pm.start_servers = 2
  pm.min_spare_servers = 1
  pm.max_spare_servers = 3
  
  # jmeter 配置
  线程数 : 7
  Ramp-Up时间（秒）: 1
  循环次数: 50
  
  # 接口执行的 sql
  select * from `personal_access_tokens` where `personal_access_tokens`.`id` = '16' limit 1;
  
  select * from `users` where `users`.`id` = '1' and `users`.`deleted_at` is null limit 1;
  
  update `personal_access_tokens` set `last_used_at` = '2022-01-21 17:38:06', `personal_access_tokens`.`updated_at` = '2022-01-21 17:38:06' where `id` = '16';
  
  ```

1. 不开启 `OPcache` 的情况下相关统计


| Label    | # 样本 | 平均值 | 最小值 | 最大值 | 标准偏差 | 异常 % | 吞吐量  | 接收 KB/sec | 发送 KB/sec | 平均字节数 |
| -------- | ------ | ------ | ------ | ------ | -------- | ------ | ------- | ----------- | ----------- | ---------- |
| HTTP请求 | 350    | 537    | 73     | 961    | 151.25   | 0.00%  | 12.5255 | 6.4         | 2.52        | 523        |
| 总体     | 350    | 537    | 73     | 961    | 151.25   | 0.00%  | 12.5255 | 6.4         | 2.52        | 523        |

2. 开启 `OPcache` 不开 `JIT` 的情况

| Label    | # 样本 | 平均值 | 最小值 | 最大值 | 标准偏差 | 异常 % | 吞吐量   | 接收 KB/sec | 发送 KB/sec | 平均字节数 |
| -------- | ------ | ------ | ------ | ------ | -------- | ------ | -------- | ----------- | ----------- | ---------- |
| HTTP请求 | 350    | 55     | 9      | 173    | 26.02    | 0.00%  | 92.86283 | 47.43       | 18.68       | 523        |
| 总体     | 350    | 55     | 9      | 173    | 26.02    | 0.00%  | 92.86283 | 47.43       | 18.68       | 523        |

3. 开启 `OPcache` 开启 `JIT` 的情况


| Label    | # 样本 | 平均值 | 最小值 | 最大值 | 标准偏差 | 异常 % | 吞吐量    | 接收 KB/sec | 发送 KB/sec | 平均字节数 |
| -------- | ------ | ------ | ------ | ------ | -------- | ------ | --------- | ----------- | ----------- | ---------- |
| HTTP请求 | 350    | 53     | 9      | 180    | 22.5     | 0.00%  | 100.51694 | 51.34       | 20.22       | 523        |
| 总体     | 350    | 53     | 9      | 180    | 22.5     | 0.00%  | 100.51694 | 51.34       | 20.22       | 523        |

4. 在开启 `OPcache` 开启 `JIT` 的情况下，根据 对 `sql` 执行顺序的分析，我发现更新的操作是最耗时且无用的，如果将这一步，移入到`fastcgi_finish_request()` 之后去执行,开启异步操作,效果如何呢？

| Label    | # 样本 | 平均值 | 最小值 | 最大值 | 标准偏差 | 异常 % | 吞吐量   | 接收 KB/sec | 发送 KB/sec | 平均字节数 |
| -------- | ------ | ------ | ------ | ------ | -------- | ------ | -------- | ----------- | ----------- | ---------- |
| HTTP请求 | 350    | 127    | 10     | 165    | 33.53    | 0.00%  | 47.76853 | 24.4        | 9.61        | 523        |
| 总体     | 350    | 127    | 10     | 165    | 33.53    | 0.00%  | 47.76853 | 24.4        | 9.61        | 523        |

很明显，异步操作并没有减轻系统的压力（毕竟在同一个进程内）反而，使得系统的吞吐量大减，原因为 响应提前给了 `JMeter`  但是 实际上 `fpm` 进程还在被占用，从而使得 大量的请求被堵塞。数据才是最真实的。所以对于高并发与负载的系统，异步操作应该尽可能的移出主逻辑程序。



相关对比图如下所示

![06](/img/06.jpg)

`laravel` 写接口，开不开 `OPcache` 吞吐性能差距在**10倍**左右，而开 `JIT` 比不开 提升大概有 `5%-10%` ，当然这是测试环境，正式服的话，一个是 `fpm` 的进程数要根据监控还有服务器配置进行动态的调整，`OPcahe` 的配置项也要根据监控数据进行动态的修改，`laravel` 本身的路由与配置缓存,根据实际情况再考虑加不加，单机 `fpm` + `4H16G` `linux` 基本上能满足市面上绝大多数中小型企业的日常业务需求。

如果再进一步优化，就考虑 长连接与连接池，常驻内存的框架，其它包括**GC，内存，JIT，调度**，这些就要看语言特性了，没法强求。在一般业务场景中，`opcache+jit` 已经足够使用了。目前的困境在于两点，一是 `fpm` 的简易性[语言下限跟上限都比较低]与 专攻`web`的局限性，二是缺少大公司背书与就业市场&舆论环境的双重恶化。

梳理完 `fpm` 的基础优化后，发现 408的4门基础课还是永远的神，应用层的东西，抽象的再厉害，一旦需要性能，底层知识你永远也避不开。