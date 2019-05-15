---
title: 关于composer的一些记录
date: 2019-05-15 10:33:38
tags:
categories:
---

- `composer dump-autoload -o`

```shell
Compsoer\ClassLoader 会优先查看 autoload_classmap 中所有生成的注册类。
如果在classmap 中没有发现再 fallback 到 psr-4 然后 psr-0

所以当打了 composer dump-autoload -o 之后，
composer 就会提前加载需要的类并提前返回。
这样大大减少了 IO 和深层次的 loop
```