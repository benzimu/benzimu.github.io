# tornado httpserver 实现解析


## tornado httpserver 详解

tornado httpserver 封装了对 http 请求的处理，是一个非阻塞、单线程的 HTTP 服务器。一个完整的 web 服务器示例如下，启动脚本后在浏览器访问 `http://localhost:8888`，页面会显示 `Hello, world`：

```python
import tornado.httpserver
import tornado.ioloop
import tornado.options
import tornado.web

from tornado.options import define, options

define("port", default=8888, help="run on the given port", type=int)

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("Hello, world")

def main():
    # 解析启动命令
    tornado.options.parse_command_line()
    # 创建Application示例
    application = tornado.web.Application([
        (r"/", MainHandler),
    ])
    # 创建HTTPServer实例
    http_server = tornado.httpserver.HTTPServer(application)
    # 监听端口，创建服务器socket
    http_server.listen(options.port)
    # 获取IOLoop并启动
    tornado.ioloop.IOLoop.current().start()

if __name__ == "__main__":
    main()
```

从实例中可以看到一个 web 服务器同时使用了 tornado.httpserver.HTTPServer 与 tornado.ioloop.IOLoop，二者不可分离。

HTTPServer 有三个父类：tornado.tcpserver.TCPServer、tornado.util.Configurable、
tornado.httputil.HTTPServerConnectionDelegate

当创建 HTTPServer 实例，即调用 tornado.httpserver.HTTPServer(application) 时，程序首先会调用 tornado.util.Configurable.\_\_new__() 方法创建 HTTPServer 实例，对 tornado.util.Configurable 的详解可参考： [tornado 配置类 Configurable]({{< ref "tornado_util_configurable.md" >}})，实例创建完成之后，会调用 tornado.httpserver.HTTPServer.initialize() 初始化参数配置。

### tornado.httpserver.HTTPServer.initialize()

```python
def initialize(self, request_callback, no_keep_alive=False, io_loop=None,
                xheaders=False, ssl_options=None, protocol=None,
                decompress_request=False,
                chunk_size=None, max_header_size=None,
                idle_connection_timeout=None, body_timeout=None,
                max_body_size=None, max_buffer_size=None,
                trusted_downstream=None):
    # 由tornado.util.Configurable分析可知，self.request_callback为
    # tornado.web.Application实例
    self.request_callback = request_callback
    self.no_keep_alive = no_keep_alive
    self.xheaders = xheaders
    self.protocol = protocol
    self.conn_params = HTTP1ConnectionParameters(
        decompress=decompress_request,
        chunk_size=chunk_size,
        max_header_size=max_header_size,
        header_timeout=idle_connection_timeout or 3600,
        max_body_size=max_body_size,
        body_timeout=body_timeout,
        no_keep_alive=no_keep_alive)
    # 调用TCPServer的初始化方法
    TCPServer.__init__(self, io_loop=io_loop, ssl_options=ssl_options,
                        max_buffer_size=max_buffer_size,
                        read_chunk_size=chunk_size)
    self._connections = set()
    self.trusted_downstream = trusted_downstream
```

在开始的示例中实例化 HTTPServer 之后会执行：http_server.listen(options.port)，该代码会创建服务器 socket，监听 8888 端口，socket/tcp 详解可参考： [socket/tcp]({{< ref "socket_tcp/index.md" >}})，listen() 方法在 tornado.tcpserver.TCPServer 中实现。

### tornado.tcpserver.TCPServer.listen()

```python
def listen(self, port, address=""):
    # 调用bind_sockets
    sockets = bind_sockets(port, address=address)
    self.add_sockets(sockets)
```

listen() 方法中就两行代码，分别调用了两个方法：tornado.netutil.bind_sockets()、add_sockets。tornado.netutil.bind_sockets() 详解参考：[tornado_netutil#bind_sockets()]({{< ref "tornado_netutil.md#tornadonetutilbind_sockets" >}})

### tornado.tcpserver.TCPServer.add_sockets()

```python
def add_sockets(self, sockets):
    # 获取当前IOLoop对象，此时还没有start
    if self.io_loop is None:
        self.io_loop = IOLoop.current()

    for sock in sockets:
        # 保存socket到self._sockets
        self._sockets[sock.fileno()] = sock
        # 重点方法
        add_accept_handler(sock, self._handle_connection,
                            io_loop=self.io_loop)
```

add_sockets() 方法最重要的是调用 add_accept_handler() 函数，详解参考：[tornado_netutil#add_accept_handler()]({{< ref "tornado_netutil.md/#tornadonetutiladd_accept_handler" >}})，从 add_accept_handler() 函数的详解可知，客户端连接到服务器之后的数据传输，最终调用 self._handle_connection() 方法完成。

### tornado.tcpserver.TCPServer._handle_connection()

```python
def _handle_connection(self, connection, address):
    # 对ssl相关处理，可以略过
    if self.ssl_options is not None:
        assert ssl, "Python 2.6+ and OpenSSL required for SSL"
        try:
            connection = ssl_wrap_socket(connection,
                                            self.ssl_options,
                                            server_side=True,
                                            do_handshake_on_connect=False)
        except ssl.SSLError as err:
            if err.args[0] == ssl.SSL_ERROR_EOF:
                return connection.close()
            else:
                raise
        except socket.error as err:
            if errno_from_exception(err) in (errno.ECONNABORTED, errno.EINVAL):
                return connection.close()
            else:
                raise
    try:
        # 如果是ssl连接，则使用SSLIOStream处理connection，否则使用IOStream
        if self.ssl_options is not None:
            stream = SSLIOStream(connection, io_loop=self.io_loop,
                                    max_buffer_size=self.max_buffer_size,
                                    read_chunk_size=self.read_chunk_size)
        else:
            stream = IOStream(connection, io_loop=self.io_loop,
                                max_buffer_size=self.max_buffer_size,
                                read_chunk_size=self.read_chunk_size)
        # 处理stream
        future = self.handle_stream(stream, address)
        if future is not None:
            # 将future添加到IOLoop
            self.io_loop.add_future(gen.convert_yielded(future),
                                    lambda f: f.result())
    except Exception:
        app_log.error("Error in connection callback", exc_info=True)
```

以非 ssl 请求为例，tornado 会实例化 tornado.iostream.IOStream 对象，用它去处理流，该对象主要封装了对请求数据读写的一些操作。之后会调用 self.handle_stream(stream, address)，而该方法在 tornado.HTTPServer中被实现。

### tornado.httpserver.HTTPServer.handle_stream()

```python
def handle_stream(self, stream, address):
    # 将相关参数保存上下文中，以便之后获取
    context = _HTTPRequestContext(stream, address,
                                    self.protocol,
                                    self.trusted_downstream)
    # 初始化HTTP1ServerConnection实例
    conn = HTTP1ServerConnection(
        stream, self.conn_params, context)
    # 保存连接conn到self._connections
    self._connections.add(conn)
    # 开始在该conn连接上处理请求
    conn.start_serving(self)
```

该方法主要是完成了对 HTTP1ServerConnection 的初始化，以及通过调用 start_serving 开始处理请求。http1connection.HTTP1ServerConnection.start_serving() 详解参考：[tornado_http1connection#start_serving]({{< ref "tornado_http1connection.md/#tornadohttp1connectionhttp1serverconnectionstart_serving" >}})

## 相关方法解析

### tornado.httpserver.HTTPServer.start_request()

```python
def start_request(self, server_conn, request_conn):
    if isinstance(self.request_callback, httputil.HTTPServerConnectionDelegate):
        delegate = self.request_callback.start_request(server_conn, request_conn)
    else:
        delegate = _CallableAdapter(self.request_callback, request_conn)

    if self.xheaders:
        delegate = _ProxyAdapter(delegate, request_conn)

    return delegate
```

方法实现了父类 tornado.httputil.HTTPServerConnectionDelegate 中的 start_request() 方法，当新的请求开始时，这个方法会被服务器调用。

通过上面 tornado.httpserver.HTTPServer.initialize() 详解可知，self.request_callback 为 tornado.web.Application 实例，而 tornado.web.Application 刚好继承至httputil.HTTPServerConnectionDelegate，则会调用 tornado.web.Application 的 start_request() 方法。

最终返回的是继承至 httputil.HTTPMessageDelegate 的 _RoutingDelegate 对象，即 delegate 为 httputil.HTTPMessageDelegate 实例。

