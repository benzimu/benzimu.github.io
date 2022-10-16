# tornado 多进程实现解析


## tornado multiple processes

tornado 实现了多进程的执行模式。使用 tornado 多进程启动服务时，IOLoop 的初始化（IOLoop.current()）操作必须在 fork 子进程之后执行。且在多进程模式下无法使用 debug 模式，因为实现 debug 模式的 autoreload.py 在 tornado.web.Application 初始化时就已经实例化了 IOLoop。tornado 使用多进程的方式有下面两种：

```python
# 方式一
app = tornado.web.Application(urls)
sockets = tornado.netutil.bind_sockets(port, address)
tornado.process.fork_processes(0) # # 默认为系统CPU核数
http_server = tornado.httpserver.HTTPServer(app)
http_server.add_sockets(sockets)
tornado.ioloop.IOLoop.current().start()

# 方式二
app = tornado.web.Application(urls)
server = tornado.httpserver.HTTPServer(app)
server.bind(port, address)
server.start(0) # 默认为系统CPU核数
tornado.ioloop.IOLoop.current().start()
```

以方式一为例分析多进程实现源码。tornado.netutil.bind_sockets() 详解参考：[tornado_netutil#bind_sockets()]({{< ref "./tornado_netutil.md/#tornadonetutilbind_sockets" >}})

### tornado.process.fork_processes()

```python
def fork_processes(num_processes, max_restarts=100):
    global _task_id
    assert _task_id is None
    # 当传入的num_processes为None或者小于等于0时，默认使用系统CPU核数
    if num_processes is None or num_processes <= 0:
        num_processes = cpu_count()
    # 如果IOLoop已经实例化，则抛出异常
    if ioloop.IOLoop.initialized():
        raise RuntimeError("Cannot run in multiple processes: IOLoop instance "
                            "has already been initialized. You cannot call "
                            "IOLoop.instance() before calling start_processes()")
    gen_log.info("Starting %d processes", num_processes)
    # 用于保存子进程PID
    children = {}
    # fork子进程的内部函数
    def start_child(i):
        # 一次调用，两次返回
        pid = os.fork()
        # 子进程运行
        if pid == 0:
            # child process
            _reseed_random()
            global _task_id
            _task_id = i
            return i
        # 父进程运行
        else:
            children[pid] = i
            return None
    
    # 根据num_processes生成对应数量的子进程
    for i in range(num_processes):
        id = start_child(i)
        # 此时父进程还需要执行下面的操作，所以fork时必须父进程先执行，否者直接返回
        if id is not None:
            return id
    num_restarts = 0
    # 遍历所有子进程
    while children:
        try:
            # 等待子进程执行，阻塞，返回子进程的pid、子进程退出时的状态信息，0表示子进程没有出现异常
            pid, status = os.wait()
        except OSError as e:
            if errno_from_exception(e) == errno.EINTR:
                continue
            raise
        if pid not in children:
            continue
        # 将退出的进程从dict中移除
        id = children.pop(pid)
        # 处理子进程退出状态
        if os.WIFSIGNALED(status):
            gen_log.warning("child %d (pid %d) killed by signal %d, restarting",
                            id, pid, os.WTERMSIG(status))
        elif os.WEXITSTATUS(status) != 0:
            gen_log.warning("child %d (pid %d) exited with status %d, restarting",
                            id, pid, os.WEXITSTATUS(status))
        else:
            gen_log.info("child %d (pid %d) exited normally", id, pid)
            continue
        num_restarts += 1
        # 如果重启次数操作最大次数，则抛出异常
        if num_restarts > max_restarts:
            raise RuntimeError("Too many child restarts, giving up")
        # 重启子进程
        new_id = start_child(id)
        if new_id is not None:
            return new_id

    sys.exit(0)
```

os.fork() 函数调用有两次返回，当返回值为 0 时，表示此时为子进程运行中，当值不为 0 时，表示父进程运行中。因为父进程需要继续执行下面的代码，所以 fork() 子进程返回时必须先执行父进程，即返回值不为 0。调用 fork() 之后先执行哪个进程的是由 Linux 下专有文件 `/proc/sys/kernel/sched_child_runs_first` 的值来确定的(值为 0 父进程先执行，非 0 子进程先执行)。

当子进程全部 fork() 完成之后，父进程（main 函数）会调用 os.wait() 阻塞住，等待子进程执行。此时，所有的子进程会执行各自内存空间的代码段。子进程由于完全复用父进程的代码段，则都会继续执行方式一中 tornado.process.fork_processes(0) 之后的代码。只有当所有的子进程都正常退出或者重启次数超过限制之后，父进程才会退出（sys.exit(0)）。

http_server.add_sockets(sockets) 方法会完成服务器 socket 的监听，即 accept()，并将其回调函数注册到 IOLoop，已完成客户端与服务端的通信。详解请参考：[tornado_httpserver]({{< ref "./tornado_httpserver.md" >}})。

方式二的 server.start(0) 封装了子进程的fork操作，原理与方式一一样。

### 多进程模式下，如何避免同一个请求不被多次执行呢？

tornado 多进程的处理流程实现创建服务器 socket，然后在 fork 子进程，这样所有的子进程都监听同一个文件描述符，即同一个 socket。

当连接过来时，所有的子进程都会收到可读事件，这是所有的子进程都会调到 accept_handler 回调函数，尝试建立连接。

一旦其中的一个子进程成功的建立了连接，当其他子进程在尝试建立连接时就会触发EWOULDBLOCK或者EAGAIN错误，这时回调函数判断是这个错误则不做处理。详解参考：[tornado_netutil#add_accept_handler()]({{< ref "./tornado_netutil.md/#tornadonetutiladd_accept_handler" >}})。

当成功建立连接的子进程还在处理这个连接的时候有过来一个连接，则由另外一个子进程处理这个连接。

tornado 就是通过这样一种机制，利用多进程提升效率，由于一个连接只能有一个子进程成功创建，同一个请求也就不会被多个子进程处理。

IOLoop 为单例、多进程模式。

