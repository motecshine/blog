---
title: PHP与PHP-FPM不得不说的二三事
date: 2018-04-17 21:06:06
tags: php
---

## 前言
    说起PHP，大家肯定对PHP-FPM也不陌生，因为如果做Web的话经常与它打交道，公司新建blog，我来抛砖引玉，文章有错误的地方欢迎大家指正。

## SAPI是什么？
  `SAPI` 是PHP框架的接口层，他是进入PHP内部的入口，其中我们使用频率比较多的几个: CLI, PHP-FPM,  ApacheHandler2(php 5.6以前经常使用,  php-ng 以后从官方的标准SAPI库中移除) ，都实现了SAPI Interface。

## PHP-FPM是什么？
  说起FPM(FastCGI Process Manager)，不得不先说说FastCGI。
  FastCGI 是Web程序和处理程序之间的一种通信协议，他是与HTTP类似的一种应用层通信的协议。注意：FastCGI 是一种协议
  PHP本身不像GO 那样实现HTTP协议(第三方库 Swoole 有实现)，而是实现了 FastCGI协议。
  FPM 就是解析和管理FastCGI Pool

## PHP-FPM 工作原理


### 三种进程管理方式
- 静态模式:

- 动态模式：

- 按需：

### 工作流程
 FPM 是一个多进程模型，他由一个Master进程和多个Worker进程组成。Master在初始化时会建立一个socket，但是不会接受和处理请求，而是由fork出来的子进程完成这些工作。
 Master进程和Woker进程之间不会直接通信，而是通过共享内存Master知道Woker进程的信息
 
 
 Master主要负责管理Worker进程。如图所示：



#### master 进程
master在调用fpm_run()后不再返回，而是进入fpm_event_loop(),这个方法会循环处理注册的几个I/O事件。

#### multi worker 进程
woker的工作就是 争抢处理请求，争抢成功后，解析FastCGI协议，获得服务器真实的php脚本地址，然后编译执行脚本，但是woker进程是阻塞的，这样是为了简单粗暴的解决进程资源安全问题。
下面是woker执行的几个阶段:
- 等待请求：woker进程阻塞在 fcgi_accept_request()中等待请求。

- 解析请求:   fastcgi请求到达后被，woker开始接受，然后解析请求，request数据陆续到达，直到request完整开始执行编译。

- ZEND_VM初始化： 执行php_request_startup(), `此阶段会调用每个扩展的PHP_RINT_FUNCTTION()`,并且激活GC模块等等。

- 执行脚本：   由php_execute_script() 激活zend_vm ，编译执行j脚本。(不清楚 可以gdb attache fpm 一下 然后打个断点) 

- 收尾:  由php_request_shutdown()完成，并且调用每个扩展的PHP_RSHUTDOWN_FUNCTION()。

## NGINX与PHP-FPM