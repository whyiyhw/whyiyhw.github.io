---
title: php8.1新特性与部分使用实践
date: 2022-01-20 12:57:04
categories: php
tags:
- jit
- php8
---

#  php8.x 的实用特性与使用

## php8 中的实用特性

### 命名参数

- 1 可以指定参数传入，跳过可选参数 ； 2 指定参数是无需遵循传入顺序。

- 看例子把 解释起来 比较 费劲

```php
public function response(array $data = [],int $code = 200,string $msg = "") : array
{
	return ['data'=> $data, 'code'=> $code,'msg' => $msg];
}

// php7
$this->response([],200,"其他的提示信息");//非必须参数只能按顺序写

// php8以后
$this->response(msg: "其他的提示信息")//指定参数传入与响应
```

### `Nullsafe`  运算符

- 这个感觉是对 `??` 操作符的一个补充

```php
$user = null;

// php7
if($user){
	$order = $user->lastOrder();
    if($order){
        $discountAmount = $order->getDiscountAmount();	
    }
}
//对于参数对象可能为 null 并且会执行方法的情况 ?? 就没法去处理了

// php8 以后 
$discountAmount = $user?->lastOrder()?->getDiscountAmount();
```

### 枚举类型 & `match` 表达式

- 枚举类型 可以替换原来 常量定义码的作用，并且更加灵活，以下是一种使用的方式
- `match` 替换 `switch` 可以减少 因为弱类型比较转换，带来的 `bug`

```php
enum ResponseCode: int
{
    case SUCCESS = 200;
    case SYS_ERROR = 500;

    public static function getMessageByCode(int $code): string
    {
        $responseCode = self::tryFrom($code);
        if (!$responseCode) {
            return '未定义错误码';
        }
        return self::getMessage($responseCode);
    }

    private static function getMessage(self $value): string
    {
        return match ($value) {
            self::SUCCESS => "成功",
            self::SYS_ERROR => "系统错误",
        };
     }
}
// 响应值构造使用
public function success(array|object $data = [], string $msg = "", int|object $code = ResponseCode::SUCCESS): JsonResponse
{
    $res = new stdClass();
    $res->data = $data ?: (object)[];
    $res->code = is_object($code) ? $code->value : $code;
    $res->msg = $msg ?: ResponseCode::getMessageByCode($res->code);
    return response()->json($res);
}
```

## `php8`  的  `opcache`  与 `jit(Just In Time)`

- `php ` 的 `opcache` 对运行效率的提示是极其巨大，建议 `php7` 以上的版本 都开起来。

- `php8` 之后 加入了 `jit`  机制，

  - 在 `opcache` 的基础上，对运行时的 `opcodes` 进行分析，对热点 `opcodes` 直接转换成指令码，来减少 `VM` 的翻译执行工作，所以代码会越跑越快
  - 但是这个提升，效果没有 `opcache` 开启时起飞的感觉，实测下来 大概有 5% - 10% 左右的不稳定提升
  - 毕竟 `php` 的项目的性能瓶颈，更多的在网络跟 `IO`

- 原理简易缩略图如下

![05](/img/05.png)

- 就不多加说明了，附一份相关配置

  ```php
    [opcache]
    # 加载扩展
    zend_extension=opcache.so
    # 开启扩展
    opcache.enable=1
    # 开启 cli 模式使用 opcahce
    opcache.enable_cli=1
    # opcache 共享内存大小，以兆字节为单位。 
    opcache.memory_consumption=256
    # 用来存储预留字符串的内存大小，以兆字节为单位
    opcache.interned_strings_buffer=8
    # opcache 哈希表中可存储的脚本文件数量上限。 真实的取值是在质数集合 { 223, 463, 983, 1979, 3907, 7963, 16229, 32531, 65407, 130987 } 中找到的第一个大于等于设置值的质数。 设置值取值范围最小值是 200，最大值在 PHP 5.5.6 之前是 100000，PHP 5.5.6 及之后是 1000000。
    # laravel 日常  32531 就够用 项目很大的情况下用 65407或者130987
    opcache.max_accelerated_files=100000
    # 浪费内存的上限，以百分比计。 如果达到此上限，那么 OPcache 将产生重新启动续发事件。
    opcache.max_wasted_percentage=5
    #如果启用，OPcache 将在哈希表的脚本键之后附加改脚本的工作目录，以避免同名脚本冲突的问题。 禁用此选项可以提高性能，但是可能会导致应用崩溃。
    opcache.use_cwd=1
    # boolean 是否开启脚本文件变更验证
    opcache.validate_timestamps=0
    # 检查脚本时间戳是否有更新的周期，以秒为单位。 设置为 0 会导致针对每个请求， OPcache 都会检查脚本更新
    opcache.revalidate_freq=60
    # 不需要注解功能的 就关闭 需要注解才开启
    opcache.save_comments=0
    #开启jit的debug
    opcache.jit_debug=1
    # jit的模式 默认为 tracing
    opcache.jit=1255
    # jit缓存的尺寸; 默认值为 0, 也就是禁用 JIT
    opcache.jit_buffer_size=256M
  ```

- 以上的配置在实际应用中，需要通过实际 `opcache` 的运行情况进行监控与调整。怎么监控与调整，等以后有时间再聊。

-     `opcode.jit`的配置值有点复杂。
      
    - 它接受 `disable，on，off，trace，function`，和按顺序排列的 4 个不同标志的 4 位值。
      - disable：在启动时完全禁用JIT功能，并且在运行时无法启用。
      - off：禁用，但是可以在运行时启用JIT。
      - on：启用tracing模式。
      - tracing：细化配置 的别名 1254。
      - function：细化配置 的别名 1205。
    
- `jit` 的四位配置顺序是：`CPU`特定的优化标志、寄存器分配、`JIT` 触发器、优化级别，官方给的推荐值为1255

  - `CPU`特定的优化标志：

    - 0 没有

    - 1个 启用AVX指令生成

  - `R`-寄存器分配：

    - 0 不执行寄存器分配

    - 1 使用本地线性扫描寄存器分配器

    - 2 使用全局线性扫描寄存器分配器

  - `JIT`触发器：
    - 0 JIT在第一次脚本加载时的所有功能
    - 1 首次执行时的JIT函数
    - 2 在第一个请求时进行概要分析，并在第二个请求时编译热功能
    - 3 动态分析并编译热功能
    - 4 在文档注释中使用@jit标记编译函数
    - 5 跟踪JIT

  - `O`-优化级别：	
    - 0 不要准时
    - 1 最小JIT（调用标准VM处理程序）
    - 2 选择性VM处理程序内联
    - 3 基于单个函数的静态类型推断的优化JIT
    - 4 静态类型推断和调用树的优化JIT
    - 5 基于静态类型推断和内部过程分析的优化JIT

- 所以 1255 指的是, 启用AVX指令生成，使用本地线性扫描寄存器分配器，跟踪JIT，基于静态类型推断和内部过程分析的优化JIT
  - function 是C = 1，R = 2，T = 0，O = 5的别名。 1205
  - tracing 是C = 1，R = 2，T = 5，O = 4的别名。 1254


-   相关参考

    -   jit https://php.watch/versions/8.0/JIT
    -   opcache https://php.p2hp.com/manual/zh/opcache.configuration.php
    -   php8特性 

        -   https://www.php.net/releases/8.1/zh.php
        -   https://www.php.net/releases/8.0/zh.php
