# tornado 源码解析


## tornado 源码解析

### 简介

此项目主要是针对 python web 框架 — tornado 源码相关模块进行解析，加深对 web 开发的理解。在详解某一模块时会引入相关基础知识概念。包括以下几个知识点：

- [Socket、TCP 详解]({{< ref "./socket_tcp/index.md" >}})
- [IO 多路复用（select、epoll、kqueue）]({{< ref "./io_multiplexing.md" >}})
- [tornado.ioloop 实现解析]({{< ref "./tornado_ioloop.md" >}})
- [tornado posix 实现解析]({{< ref "./tornado_platform_posix/index.md" >}})
- [tornado 配置类 Configurable 实现解析]({{< ref "./tornado_util_configurable.md" >}})
- [tornado.httpserver 实现解析]({{< ref "./tornado_httpserver.md" >}})
- [tornado netutil 实现解析]({{< ref "./tornado_netutil.md" >}})
- [tornado.http1connection 实现解析]({{< ref "./tornado_http1connection.md" >}})
- [tornado.iostream 实现解析]({{< ref "./tornado_iostream.md" >}})
- [tornado.gen 实现解析]({{< ref "./tornado_gen.md" >}})
- [tornado 定时器实现解析]({{< ref "./tornado_ioloop_PeriodicCallback.md" >}})
- [tornado.concurrent 实现解析]({{< ref "./tornado_concurrent.md" >}})
- [tornado 多进程实现解析]({{< ref "./tornado_multi_processes.md" >}})

### 环境

- python 2.7
- tornado 4.5.1
- Ubuntu 16.04

