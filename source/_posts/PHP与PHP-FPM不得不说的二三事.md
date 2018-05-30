---
title: PHP与PHP-FPM不得不说的二三事
date: 2018-04-17 21:06:06
tags: php
categories: 
- php
---

## 前言
  说起PHP，大家肯定对PHP-FPM也不陌生，因为如果做Web的话经常与它打交道，公司新建blog，我来抛砖引玉，文章有错误的地方欢迎大家指正。
  可能是一个系列将从 php-fpm讲起，会针对 ZendVM  词法编译，语法编译，粗暴的zend_mm和gc，zend_vm重要的数据结构(zend_array(php5 是hashtable)， zval)

## SAPI是什么？
  `SAPI` 是PHP框架的接口层，他是进入PHP内部的入口，其中我们使用频率比较多的几个: CLI, PHP-FPM,  ApacheHandler2(php 5.6以前经常使用,  php-ng 以后从官方的标准SAPI库中移除) ，都实现了SAPI Interface。

## PHP-FPM是什么？
  说起FPM(FastCGI Process Manager)，不得不先说说FastCGI。
  FastCGI 是Web程序和处理程序之间的一种通信协议，他是与HTTP类似的一种应用层通信的协议。注意：FastCGI 是一种协议
  PHP本身不像GO 那样实现HTTP协议(第三方库 Swoole 有实现)，而是实现了 FastCGI协议。
  FPM 就是解析和管理FastCGI Pool

## PHP-FPM 工作原理

```c
fpm.c
/*	children: return listening socket
	parent: never return */
int fpm_run(int *max_requests) /* {{{ */
{
	struct fpm_worker_pool_s *wp;

	/* create initial children in all pools */
	for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
		int is_parent;

		is_parent = fpm_children_create_initial(wp);

		if (!is_parent) {
			goto run_child;
		}

		/* handle error */
		if (is_parent == 2) {
			fpm_pctl(FPM_PCTL_STATE_TERMINATING, FPM_PCTL_ACTION_SET);
			fpm_event_loop(1);
		}
	}

	/* run event loop forever */
	fpm_event_loop(0);

run_child: /* only workers reach this point */

	fpm_cleanups_run(FPM_CLEANUP_CHILD);

	*max_requests = fpm_globals.max_requests;
	return fpm_globals.listening_socket;
}
/* }}} */

```

### 三种进程管理方式
- 静态模式:
  在启动的时候 master根据pm.max_children配置fork出相应数量地worker进程，woker的数量是固定的。

- 动态模式(dynamic)：
  这种模式应该是最常用的， fpm 启动时会根据pm.start_servers配置初始化一定数量的worker。 如果master发现空闲woker低于pm.min_spare_servers配置数则会fork出更多的woker进程，但是不会超过pm.max_spare_servers, 如果master发现了空闲的woker 大于 pm.max_spare_servers 则会杀死部分woker。

- 按需(ondemand)：
  启动时，不分配woker，请求来了 master 才会fork，但是不会超过pm.max_children。请求流程完了也不会立马kill woker，当woker的空闲时间超过pm.oricess_idle_timeout才会被杀死


### 工作流程
 FPM 是一个多进程模型，他由一个Master进程和多个Worker进程组成。Master在初始化时会建立一个socket，但是不会接受和处理请求，而是由fork出来的子进程完成这些工作。
 Master进程和Woker进程之间不会直接通信，而是通过共享内存Master知道Woker进程的信息


#### master 进程
从代码中可以看到 master在调用fpm_run()后不再返回，而是进入fpm_event_loop(),这个方法会循环处理注册的几个I/O事件。

#### multi worker 进程
woker的工作就是 争抢处理请求，争抢成功后，解析FastCGI协议，获得服务器真实的php脚本地址，然后编译执行脚本，但是woker进程是阻塞的，这样是为了简单粗暴的解决进程资源安全问题。
下面是woker执行的几个阶段:
- 等待请求：woker进程阻塞在 fcgi_accept_request()中等待请求。

- 解析请求:   fastcgi请求到达后被，woker开始接受，然后解析请求，request数据陆续到达，直到request完整开始执行编译。

- ZEND_VM初始化： 执行php_request_startup(), `此阶段会调用每个扩展的PHP_RINT_FUNCTTION()`,并且激活GC模块等等。

- 执行脚本：   由php_execute_script() 激活zend_vm ，编译执行j脚本。(不清楚 可以gdb attache fpm 一下 然后打个断点) 

- 收尾:  由php_request_shutdown()完成，并且调用每个扩展的PHP_RSHUTDOWN_FUNCTION()。

## PHP-FPM 配置与优化
* 不讲apache与php-fpm 我用的不多，没有深入了解过。

### 几个关键的配置
fpm一般有两个配置文件，fpm.conf 是针对所有的，www.conf 是针对fpm  master-woker pool的

- fpm.conf
 - error_log = log/php-fpm.log   `错误日志：超时，woker不够用，php 脚本crash都会记录在这里`
 - process.max = 128 `控制全局的woker数量`

- www.conf
  - listen = 127.0.0.1:9000 `fpm FastCGI 端口，与常用的http服务做通讯(nginx, apache, iis)`
  - listen.allowed_clients = 127.0.0.1 `允许访问的客户端，如果这个是127.0.0.1则 nginx或者apache 需要与php-fpm在一台主机上`
  
  *  下面的上文讲过
  - pm = dynamic  
  - pm.max_children = 5 
  - pm.start_servers = 2 
  - pm.min_spare_servers = 1
  - pm.max_spare_servers = 3
  - pm.process_idle_timeout = 10s;
