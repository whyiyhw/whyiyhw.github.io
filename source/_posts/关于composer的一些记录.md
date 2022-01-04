---
title: 关于composer的一些记录
date: 2019-05-15 10:33:38
categories: php
tags:
- composer
---

### 为什么 上线要使用 `composer dump-autoload -o`
- `composer dump-autoload -o` 做了什么？
	- 自动生成了 注册类的 `key=>value` 数组 按A-Z进行排序并生成了对应的索引
- `Compsoer\ClassLoader` 会优先查看 `autoload_classmap` 中所有生成的注册类。
- 如果在`classmap` 中没有发现再 `fallback` 到 `psr-4`规则去找 然后 `psr-0`规则去找

- 所以当执行 `composer dump-autoload -o` 之后，`composer` 就会提前加载需要的类并提前返回。这样大大减少了 `IO` 和深层次的 `loop`
```

### 问题点 `You made a reference to a non-existent script @php artisan package:discover`

​```shell
composer -V
Composer version 1.2.1 2016-09-12 11:27:19
```

- 解决方法，升级 composer 版本
- `composer selfupdate`

```shell

Updating to version 1.6.3 (stable channel).
    Downloading: 100%
Use composer self-update --rollback to return to version 1.2.1
```