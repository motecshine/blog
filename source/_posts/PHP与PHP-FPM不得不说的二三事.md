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
  
  请求流程：
  
  Client ---[GET /xxx?a=1]----> Web Server-------->应用程序
 
## PHP-FPM 工作原理

## NGINX与PHP-FPM