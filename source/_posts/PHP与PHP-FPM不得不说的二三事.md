---
title: PHP与PHP-FPM不得不说的二三事
date: 2018-04-17 21:06:06
tags: php
---

## 前言
    说起PHP，大家肯定对PHP-FPM也不陌生，因为如果做Web的话经常与它打交道，公司新建blog，我来抛砖引玉，文章有错误的地方欢迎大家指正。

## SAPI是什么？
  `SAPI` 是PHP框架的接口层，他是进入PHP内部的入口。
   其中我们使用频率比较多的几个: CLI, PHP-FPM,  ApacheHandler2(php 5.6以前经常使用,  php-ng 以后从官方的标准SAPI库中移除) ，都实现了SAPI Interface。
## PHP-FPM是什么？
  说起FPM(FastCGI Process Manager)，不得不先说说FastCGI。
 
  FastCGI 是Web程序和处理程序之间的一种通信协议，他是与HTTP类似的一种应用层通信的协议。注意：FastCGI 是一种协议
 
  PHP本身不像golang 那样实现HTTP协议(第三方库 Swoole 有实现)，而是实现了 FastCGI协议。
  
  请求流程， 如图所示：
  
  Client ---[GET /xxx?a=1]----> Web Server-------->应用程序
 
## PHP-FPM 工作原理

### 工作流程
 FPM 是一个多进程模型，他由一个Master进程和多个Worker进程组成。Master在初始化时会建立一个socket，但是不会接受和处理请求，而是由fork出来的子进程完成这些工作。
 Master主要负责管理Worker进程。如图所示：
#### master 进程

#### multiWorker 进程


## NGINX与PHP-FPM