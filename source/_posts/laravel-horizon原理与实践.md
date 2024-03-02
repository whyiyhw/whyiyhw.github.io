---
title: laravel-horizon原理与实践
abbrlink: 3139d2fc
date: 2023-12-01 23:26:17
tags:
  - laravel
  - horizon
categories: laravel
index_img: /img/post/ComfyUI_00008_.png
---

> 基于 `laravel10`，`laravel-horizon 5.21.4` 与 `php8.1`  得到的结论,（2023-11-30）
## 安装与配置
### 安装与发布
- 前置条件 
> 需要 `pcntl,posix` 扩展
- 安装
```shell
composer require laravel/horizon
```
- 发布
```shell
php artisan horizon:install
```
- 重新发布 
```shell
php artisan horizon:publish
```
### 配置修改
> `config/horizon.php`
> balance 改为 false ，正常 job 逻辑就行
### 定时器修改
```php
// horizon 统计数据生成  
$schedule->command('horizon:snapshot')->everyFiveMinutes();
```
### 运行
> `php artisan horizon`
> 查看 `http://{your host}/horizon`
### supervisor 部署
```ini
[program:laravel-horizon]
process_name=%(program_name)s
command = php artisan horizon
directory = /project/path
autostart = true
startsecs = 5
autorestart = true
startretries = 3
user = root
redirect_stderr = true
stdout_logfile_maxbytes = 10MB
stdout_logfile_backups = 5
stdout_logfile = /var/log/laravel-horizon.log
```

## 原理解析

### horizon 如何去 hook 队列的？

- 在应用启动时，服务注册 去注入 QueueManager 的 connector 关联闭包
```php
//vendor/laravel/horizon/src/HorizonServiceProvider.php:141
\Laravel\Horizon\HorizonServiceProvider::register()

//vendor/laravel/horizon/src/HorizonServiceProvider.php:153
$this->registerQueueConnectors();

//vendor/laravel/horizon/src/HorizonServiceProvider.php:189
protected function registerQueueConnectors()  
{  
//vendor/laravel/framework/src/Illuminate/Queue/QueueManager.php
    $this->callAfterResolving(QueueManager::class, function ($manager) {  
        $manager->addConnector('redis', function () {  
        //vendor/laravel/horizon/src/Connectors/RedisConnector.php 
            return new RedisConnector($this->app['redis']);  
        });    
    });
}

```

- 当 `dispatch` 被调用的时候
```php
//vendor/laravel/framework/src/Illuminate/Foundation/Bus/PendingDispatch.php::__destruct():193
app(Dispatcher::class)->dispatch($this->job);

//vendor/laravel/framework/src/Illuminate/Bus/Dispatcher.php::dispatch():76
$this->dispatchToQueue($command)

//vendor/laravel/framework/src/Illuminate/Bus/Dispatcher.php::dispatchToQueue():229
$this->pushCommandToQueue($queue, $command);

//vendor/laravel/framework/src/Illuminate/Bus/Dispatcher.php::pushCommandToQueue():253
$queue->push($command);

// 这里的 $queue 变成了 
// vendor/laravel/horizon/src/RedisQueue.php::push():42
$this->enqueueUsing(  
    $job,  
    $this->createPayload($job, $this->getQueue($queue), $data),  
    $queue,  
    null,  
    function ($payload, $queue) use ($job) {
        $this->lastPushed = $job;  
        return $this->pushRaw($payload, $queue);  
    });

//其重载了 
//vendor/laravel/framework/src/Illuminate/Queue/RedisQueue.php::push():132
$this->enqueueUsing(  
    $job,  
    $this->createPayload($job, $this->getQueue($queue), $data),  
    $queue,  
    null,  
    function ($payload, $queue) {  
        return $this->pushRaw($payload, $queue);  
    });
```

- 对比下 两个方法
```php
//vendor/laravel/framework/src/Illuminate/Queue/RedisQueue.php::pushRaw():153
public function pushRaw($payload, $queue = null, array $options = [])  
{  
    $this->getConnection()->eval(  
        LuaScripts::push(), 2, $this->getQueue($queue),  
        $this->getQueue($queue).':notify', $payload  
    );  
  
    return json_decode($payload, true)['id'] ?? null;  
}

// vendor/laravel/horizon/src/RedisQueue.php::pushRaw():65
public function pushRaw($payload, $queue = null, array $options = [])  
{  
    $payload = (new JobPayload($payload))->prepare($this->lastPushed);  
  
    parent::pushRaw($payload->value, $queue, $options);  
  
    $this->event($this->getQueue($queue), new JobPushed($payload->value));  
  
    return $payload->id();  
}
```

> 很明显,就是装饰器模式，在方法前/后加入了 事件机制

> 也就是说，核心的原理就是在 queue 数据存入 redis 前后，会触发定义的 laravel-horizon 事件, 从而去记录 队列的信息

### `php artisan horizon` 做了什么？

```php
//vendor/laravel/horizon/src/Console/HorizonCommand.php::handle():56
$master->monitor();

//vendor/laravel/horizon/src/MasterSupervisor.php::monitor():211
public function monitor()  
{  
    $this->ensureNoOtherMasterSupervisors();  
  
    $this->listenForSignals();  
  
    $this->persist();  
  
    while (true) {  
        sleep(1);  
        $this->loop();  
    }
}
```
- 很显然，核心在于 while 循环 中的 loop
```php
//vendor/laravel/horizon/src/MasterSupervisor.php::monitor():245
public function loop()  
{  
    try {
	    // 处理 等待中 的信号
        $this->processPendingSignals();  
        // 处理 等待中 的命令
        $this->processPendingCommands();  
		
		// 运行中
        if ($this->working) {
            $this->monitorSupervisors();  
        }  
        // 更新 监控信息
        $this->persist();  
		
		// 触发 监控事件
        event(new MasterSupervisorLooped($this));  
    } catch (Throwable $e) {  
        app(ExceptionHandler::class)->report($e);  
    }
}

// vendor/laravel/horizon/src/EventMap.php:50
Events\MasterSupervisorLooped::class => [  
    Listeners\TrimRecentJobs::class,  
    Listeners\TrimFailedJobs::class,  
    Listeners\TrimMonitoredJobs::class,  
    Listeners\ExpireSupervisors::class,  
    Listeners\MonitorMasterSupervisorMemory::class,  
]
```
> 总结下就是一个一秒的定时器，去持续的监控 Supervisor 的状态
## QA
### `laravel-horizon` 会影响 job 消费吗？
> 根据原理，我们知道 其核心原理为 替换了 队列操作 `redis` 的方法，其本身不会影响 job 的功能，  `php artisan horizon`  也不会 影响到  `php artisan queue:work` 

### auth 与 sanctum 如何结合起来？
> 我们知道 sanctum 就是通过 header 中 传入 token 来实现的认证
> `app/Providers/HorizonServiceProvider.php` 中存在 gate 方法 内部的闭包函数返回 true 就代表响应成功，
> 又因为 api 与前端使用了同一个域名，那么久简单了，登录后将 token 信息存入 cookie, 在闭包内读取

```php
// 获取 cookie 中的 token 信息  
$token = request()?->cookie('token') ?? "";  
if (empty($token) || !str_contains($token, 'Bearer')) {  
    return false;  
}  
$token = str_replace('Bearer ', '', $token);  
// 通过 sanctum token 获取 user 信息
$userId = PersonalAccessToken::findToken($token)->tokenable_id ?? 0;  
$user = AdminUsers::query()->find($userId);
```

> 再指定用户可访问就好

### 缺点有哪些？
> 扩展性不够好，想要改 js 跟 blade 模版很困难，特别是 balde 内部链接默认走 http ，导致我只能在生成缓存文件后，再去替换，确实很难受。

> 通知机制一般，不支持自定义的 webhook 