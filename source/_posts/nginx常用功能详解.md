---
title: nginx常用功能详解
categories: nginx
tags:
  - php-fpm
  - nginx
abbrlink: 144ead03
date: 2022-03-03 13:51:56
---

# nginx 常用功能详解

## nginx与php-fpm的两种通信方式详解

一般来说，我们配置 `nginx` 与 `php-fpm` 的通信会有两种设置，以 `unix socket` 与 `tcp/ip socket` 的方式 通过 fast-cgi 协议进行通信

比如

```nginx
# tcp socket 的方式
fastcgi_pass  127.0.0.1:9000
# unix socket 的方式
fastcgi_pass  /usr/run/php-fpm.sock 
```

## 对比 UNIX Domain Socket 与 TCP/IP Socket

`socket api` 原本是为 网络通讯设计的，后来在socket 上发展出一种 `IPC(Inter-Process Communication 进程间通信)` 机制， 就是`unix socket` ，当然 `socket` 也可以进行 本机通信（通过 `loopback` 地址`127.0.0.1`），`unix socket`对比 `tcp socket` 效率更高，不需要经过网络协议栈，不需要打包 拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。`IPC` 机制本质上是可靠的通讯，而网络协议是为不可靠的通讯设计的。

- `UNIX Socket` （同一服务器上不同进程间的通信机制）
  - 不需要网络协议栈、打包拆包,减少了 tcp 开销，理论效率比 tcp socket 更高
  - 缺点也是因为缺少tcp的检查与重试机制，在高并发时稳定性欠佳（需要优化内核与应用相关参数）
    - listen.backlog   等待连接数，当其被用完时直接 502
    - pm.max_children 进程都在执行等待时处理过慢 504
    - <https://shipengliang.com/software-exp/nginx%E5%92%8Cphp-fpm%E4%BD%BF%E7%94%A8tcp-socket%E8%BF%98%E6%98%AFunix-socket.html>
    - <https://blog.csdn.net/weixin_42123737/article/details/81367074>
- `TCP/IP Socket` （多台服务器上不同进程间的通信机制）
  - 扩展方便 , 适合 nginx 与 php-fpm 在不同服务器的情况
  - 相关优化方案比较多  <https://blog.51cto.com/u_15322220/3286743>
- 相关衍生问题

  - 协议 CGI 与 FAST-CGI
    - cgi 与 fast-cgi 协议 对比？
    - fast-cgi 协议相关？
      - <https://www.cnblogs.com/itbsl/p/9828776.html#%E6%B7%B1%E5%85%A5cgi%E5%8D%8F%E8%AE%AE>
      - 协议头 `version type request_id content_length`
      - 协议体 `data`
  - fpm backlog 配置优化
    - <https://www.cnxct.com/something-about-phpfpm-s-backlog/>

  - 进程间通信 IPC 的7种方式？
    - 匿名管道 pipe
    - 有名管道 fifo
    - 信号 Signal
    - 消息队列 Message
    - 共享内存 share memory
    - 信号量 semaphore
    - 嵌套字 socket
