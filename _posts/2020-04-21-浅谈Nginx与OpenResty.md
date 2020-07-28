---
layout: post
author: oy
tag: Nginx OpenResty WebServer Proxy
---
## Nginx

### 简介

Nginx是一款 `免费`、`高性能`、`高可靠` 的 `开源` 软件，
可作为 `WebServer`、 `LB`、 `WebCacheServer` 等软件设备满足互联网业务需求。

伴随着[C10K](http://www.kegel.com/c10k.html)问题产生了许多优秀开源软件，Nginx算是其中的出色玩家之一。

根据2019年9月的统计数据，Nginx已经服务/代理了全球25.64%的最繁忙站点，例如(Yandex,VK,Hulu,CloudFlare,Airbnb,GitHub,…)。

### 基础架构

1. 模块化

	1). 高度抽象的模块接口

	2). 核心模块接口的简单化

	3). 多层次、多类别的模块设计

2. 多进程(Master-Slave多进程模型)

	1). 充分利用多核并发处理能力

	2). 通过IPC实现负载均衡

	3). Master进程负责对Worker进程的管理、监控提升服务的可靠性 

3. 事件驱动与异步非阻塞

	1). EPOLL的使用(内核事件驱动模块复用，内部基于EPOLL实现的高效timer)

	2). 请求的多阶段划分与处理(实现异步非阻塞的基础，阻塞耗时操作的分解) 

### 高效率基础

1. 高效算力利用(multi-processes,event-driven,asynchronous,non-blocking)

2. 高效的网络IO(kqueue, epoll, select, poll)

3. 高效的文件IO(sendfile, aio, directio)

4. 高效的内存管理(malloc, tcmalloc, jemalloc)

5. 高效的网络处理(keepalive, cache request and streaming response)

6. 高效的数据结构(内部实现了基础的数据结构如双向链表，动态数组，单向链表，rbtree, hash_table等) 

### 流量处理方式及优化思路
1. process
	* accept_mutex
	* EPOLLEXCLUSIVE
	* reuseport
2. network resource
	* fds
	* port
3. network traffic
	* cache request
	* streaming response
	* keepalive
4. kernel params
	* sysctl.conf
	* limits.conf


### Nginx VS Apache

#### 一、网络IO架构

**Nginx**  
1. 多进程  
2. 事件驱动  
3. 基于多请求阶段划分的异步非阻塞处理方式  

**Apache**  
1. prefork:新连接对应新进程  
2. worker:新连接对应新线程，一个进程持有多个线程  
3. event:同上新连接对应新线程，但针对keepalive做了优化，特定线程用于保持keepalive连接，活跃请求通过其他线程处理  
	        
#### 二、静态与动态 内容处理

**Nginx**  
1. 天然仅支持静态内容处理  
2. 动态内容处理须调用外部模块等，并须等待返回结果  
3. 连接nginx与动态内容处理模块，须通过 http,FastCGI,SCGI,uWSGI,memcache等协议  
4. 由此nginx可以更专注于静态内容处理，并达到可观的高效率

**Apache**  
1. 基于文件的模式支持静态内容处理  
2. 在工作实例中嵌入语言解释/处理器 来处理动态内容  
3. 处理动态内容无须与其他软件模块交互  
4. 由此nginx可以更专注于静态内容处理，并达到可观的高效率

#### 三、配置文件

**Nginx**  
1. 全局统一配置  
2. 高效；对比apache失去灵活性，但提升效率，比如多级目录每层目录配置的解析等类似场景  
3. 安全；对比目录级的配置，用户未必能做到安全性的配置管理  
4. 动态；通关nginx变量可实现动态变量设置与获取，达到动态配置的效果

**Apache**  
1. 目录级别的配置文件 .htaccess  
2. 配置文件动态加载(每次请求都会重新解析配置)  
3. 做到目录级别的权限隔离，不同目录对应不同用户

#### 四、解析

**Nginx**	 
1. 基于URI解析

**Apache**  
1. 基于文件系统资源的方式解析

#### 五、兼容性、生态、支持、文档

**Nginx**  
1. 相对apache略逊色

**Apache**  
1. 无处不在


#### 六、搭配使用  

1. 通常用Nginx作为反向代理，Apache作为Web Server；  
2. Nginx处理静态内容，Apache处理动态内容。
<br><br><br>

## OpenResty

### 简介
1. Free + Open-Source + High-Performance + High-Stability + …
2. WebServer + WebCacheServer + Proxy + LB + …
3. lua-nginx-module + LuaJIT
4. lualib, test framework, debugging tool

### 高效率基础：
1. Lua and LuaJIT

2. ngx\_lua\_mode
	- 请求响应的多阶段处理
	- 利用协程

		协程是非抢占式的，正在运行的协程只有在显式调用 yield 函数后才会被挂起;因此，同一时间内，只有一个协程在处理（因为 worker 是单线程的）。  
		lua 协程还有一个特性，就是子协程优先运行，只有当子协程都被挂起或运行结束才会继续运行父协程。在balancer_by_lua*, header_filter, body_filter, log阶段中openresty直接在主协程中执行代码，所以这些阶段是需要尽量避免阻塞操作的;  
		而在access，content等其他几个阶段中，会创建一个新的协程去执行此阶段的lua代码。  
		表现在API层面，两者的区别就是能否执行ngx.sleep, ngx.socket, ngx.thread这几个命令。 

3. asynchronous, non-blocking

### 最佳实践:
1. api gateway(orange, kong)

2. 火焰图(性能分析)
	- systemtap
	- openresty-systemtap-toolkit
	- FlameGraph
	- wrk
