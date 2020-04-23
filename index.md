# 浅谈Nginx与OpenResty

## Nginx

### 简介

Nginx是一款 `免费`、`高性能`、`高可靠` 的 `开源` 软件，

可作为 `WebServer`、 `LB`、 `WebCacheServer` 等软件设备满足互联网业务需求。

伴随着[C10K](http://www.kegel.com/c10k.html)问题产生了许多优秀开源软件，Nginx算是其中的出色玩家之一。

根据2019年9月的统计数据，Nginx已经服务/代理了全球25.64%的最繁忙站点，例如(Yandex,VK,Hulu,CloudFlare,Airbnb,GitHub,…)。

### 架构

1. 模块化

2. 多进程

3. 事件驱动与异步非阻塞

### 高效率基础

1. 高效算力利用(multi-processes,event-driven,asynchronous,non-blocking)

2. 高效的网络IO(kqueue, epoll, select, poll)

3. 高效的文件IO(sendfile, aio, directio)

4. 高效的内存管理(malloc, tcmalloc, jemalloc)

5. 高效的网络处理(keepalive, cache request and streaming response)

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

| head1        | head two          | three |
|:-------------|:------------------|:------|
| ok           | good swedish fish | nice  |
| out of stock | good and plenty   | nice  |
| ok           | good `oreos`      | hmm   |
| ok           | good `zoute` drop | yumm  |


一、网络IO架构
Nginx	Apache
多进程、	多进程模型(MPM):
事件驱动、	1.prefork:
异步、	        新连接对应新进程
非阻塞	2.worker:
	        新连接对应新线程，一个进程持有多个线程
	3.event:
	        同上新连接对应新线程，但针对keepalive做了优化，特定线程用于保持keepalive连接，活跃请求通过其他线程处理


二、静态与动态 内容处理
Nginx	Apache
1.天然仅支持静态内容处理	1.基于文件的模式支持静态内容处理
2.动态内容处理须调用外部模块等，并须等待返回结果	2.在工作实例中嵌入语言解释/处理器 来处理动态内容
3.连接nginx与动态内容处理模块，须通过 http,FastCGI,SCGI,uWSGI,memcache等协议	3.处理动态内容无须与其他软件模块交互
4.由此nginx可以更专注于静态内容处理，并达到可观的高效率



三、配置文件(分布式 与 统一 配置)
Nginx	Apache
1.全局统一配置	1.目录级别的配置文件 .htaccess
2.高效；对比apache失去灵活性，但提升效率，比如多级目录每层目录配置的解析等类似场景。	2.配置文件动态加载(每次请求都会重新解析配置)
3.安全；对比目录级的配置，用户未必能做到安全性的配置管理。	3.做到目录级别的权限隔离，不同目录对应不同用户
4.动态；通关nginx变量可实现动态变量设置与获取，达到动态配置的效果。



四、基于文件解析 与 基于URI解析
Nginx	Apache
1.基于URI解析	1.基于文件系统资源的方式解析


五、模块
Nginx	Apache
1.第三方模块须选定并静态编译进nginx。	1.支持动态加载或卸载 第三方模块，提供开关选项。
	2.第三方模块可以专注拓展功能，而不影响核心功能。


六、兼容性、生态、支持、文档
Nginx	Apache
1.相对apache略逊色	1.无处不在


七、搭配使用
通常用Nginx作为反向代理，Apache作为Web Server；
Nginx处理静态内容，Apache处理动态内容。

## OpenResty

### 简介
1. Free + Open-Source + High-Performance + High-Stability + …
2. WebServer + WebCacheServer + Proxy + LB + …
3. lua-nginx-module + LuaJIT
4. lualib, test framework, debugging tool

###高效率基础：
1. Lua and LuaJIT
2. ngx_lua_mode
3. asynchronous, non-blocking

###最佳实践:
1. api gateway(orange, kong)
2. 火焰图(性能分析)

systemtap
openresty-systemtap-toolkit
FlameGraph
wrk


Nginx:
简介：
1.free+open-source+high-performance+high-stability+...
2.WebServer+WebCacheServer+proxy+LB+...
3.C10K problem -> http://www.kegel.com/c10k.html
C10K manifest spurred a number of attempts to solve the problem of web server optimization to handle a large number of clients at the same time, and nginx turned out to be one of the most successful ones;
10,000 inactive HTTP keep-alive connections take about 2.5M memory;
4.served or proxied 25.64% busiest sites in September 2019(Yandex,VK,Hulu,CloudFlare,Airbnb,GitHub,…)

架构概览：
1.modular
core module,event module,phase handler,protocols,variable handlers, filters, upstream and load balancers.
2.event-driven,asynchronous,non-blocking
event collector,dispatcher,handler.
3.multi-process and single-threaded

高效率基础：
1.高效的算力利用(multi-processes,event-driven,asynchronous,non-blocking)
2.高效的网络IO(kqueue, epoll, select, poll)
3.高效的文件IO(sendfile, aio, directio)
4.高效的内存管理(malloc, tcmalloc, jemalloc)
5.高效的网络处理(keepalive, cache request and streaming response)

流量处理方式及相关优化：
1.process
accept_mutex, EPOLLEXCLUSIVE, reuseport
2.network resource
fds
port
3.network traffic
cache request
streaming response
keepalive
4.kernel params
sysctl.conf
limits.conf

OpenResty:
简介：
1.Free + Open-Source + High-Performance + High-Stability + …
2.WebServer + WebCacheServer + Proxy + LB + …
3.lua-nginx-module + LuaJIT
4.lualib, test framework, debugging tool

高效率基础：
1.Lua and LuaJIT
2.ngx_lua_mode
3.asynchronous, non-blocking

最佳实践:
1.api gateway(orange, kong)
2.火焰图(性能分析)

systemtap
openresty-systemtap-toolkit
FlameGraph
wrk