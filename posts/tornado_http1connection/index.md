# tornado http1connection 实现解析


## tornado http1connection 解析

tornado http1connection 主要是对 http 协议进行了封装。

### tornado.http1connection.HTTP1ServerConnection.start_serving()

```python
def start_serving(self, delegate):
    # 这里断言delegate是httputil.HTTPServerConnectionDelegate的实例
    assert isinstance(delegate, httputil.HTTPServerConnectionDelegate)
    self._serving_future = self._server_request_loop(delegate)
    self.stream.io_loop.add_future(self._serving_future,
                                    lambda f: f.result())
```

开始处理这个连接上的请求，在 tornado.HTTPServer.handle_stream() 中调用此方法，传入的参数为 HTTPServer 的实例，即 delegate 为 HTTPServer 实例，由于 HTTPServer 继承至 httputil.HTTPServerConnectionDelegate，所以断言成功，程序开始执行 self._server_request_loop()。

### tornado.http1connection.HTTP1ServerConnection._server_request_loop()

```python
@gen.coroutine
def _server_request_loop(self, delegate):
    try:
        while True:
            # 初始化HTTP1Connection实例
            conn = HTTP1Connection(self.stream, False,
                                    self.params, self.context)
            # 调用delegate的start_request处理连接请求
            request_delegate = delegate.start_request(self, conn)
            try:
                # 读取http响应
                ret = yield conn.read_response(request_delegate)
            except (iostream.StreamClosedError,
                    iostream.UnsatisfiableReadError):
                return
            except _QuietException:
                # This exception was already logged.
                conn.close()
                return
            except Exception:
                gen_log.error("Uncaught exception", exc_info=True)
                conn.close()
                return
            # 如果ret为false，则表示读取了完整的http响应
            if not ret:
                return
            yield gen.moment
    finally:
        delegate.on_close(self)v
```

方法中调用了 delegate.start_request(self, conn)，即 [tornado.httpserver.HTTPServer.start_request()]({{< ref "tornado_httpserver.md/#tornadohttpserverhttpserverstart_request" >}})，得到一个 httputil.HTTPMessageDelegate 实例，即 request_delegate。接下来开始读取响应数据。

### tornado.http1connection.HTTP1Connection.read_response()

```python
def read_response(self, delegate):
    # 判断是否需要解压缩数据
    if self.params.decompress:
        delegate = _GzipMessageDelegate(delegate, self.params.chunk_size)
    return self._read_message(delegate)
```

### tornado.http1connection.HTTP1Connection._read_message()

```python
@gen.coroutine
def _read_message(self, delegate):
    need_delegate_close = False
    try:
        # 读取请求header数据，返回Future对象
        header_future = self.stream.read_until_regex(
            b"\r?\n\r?\n",
            max_bytes=self.params.max_header_size)
        # 判断header_timeout是否为None，如果为None，则直接读取header_future数据
        if self.params.header_timeout is None:
            header_data = yield header_future
        else:
            try:
                # header超时实现
                header_data = yield gen.with_timeout(
                    self.stream.io_loop.time() + self.params.header_timeout,
                    header_future,
                    io_loop=self.stream.io_loop,
                    quiet_exceptions=iostream.StreamClosedError)
            except gen.TimeoutError:
                self.close()
                raise gen.Return(False)
        # 解析header信息，HTTP协议分为request-line、request-header、
        # request-body三个部分的。在查看各浏览器、服务器配置的时候往往将
        # request-line、request-header归为一类了
        start_line, headers = self._parse_headers(header_data)
        # 判断是否作为客户端
        if self.is_client:
            # 如果作为客户端，则解析服务端响应起始行
            start_line = httputil.parse_response_start_line(start_line)
            self._response_start_line = start_line
        else:
            # 如果作为服务端，则解析客户端请求起始行
            start_line = httputil.parse_request_start_line(start_line)
            self._request_start_line = start_line
            self._request_headers = headers
        # 判断是否能保持长链接，即Connection: keep_alive
        self._disconnect_on_finish = not self._can_keep_alive(
            start_line, headers)
        need_delegate_close = True
        with _ExceptionLoggingContext(app_log):
            header_future = delegate.headers_received(start_line, headers)
            if header_future is not None:
                yield header_future
        if self.stream is None:
            # We've been detached.
            need_delegate_close = False
            raise gen.Return(False)
        skip_body = False
        if self.is_client:
            if (self._request_start_line is not None and
                    self._request_start_line.method == 'HEAD'):
                skip_body = True
            code = start_line.code
            if code == 304:
                # 304 responses may include the content-length header
                # but do not actually have a body.
                # http://tools.ietf.org/html/rfc7230#section-3.3
                skip_body = True
            if code >= 100 and code < 200:
                # 1xx responses should never indicate the presence of
                # a body.
                if ('Content-Length' in headers or
                        'Transfer-Encoding' in headers):
                    raise httputil.HTTPInputError(
                        "Response code %d cannot have body" % code)
                # TODO: client delegates will get headers_received twice
                # in the case of a 100-continue.  Document or change?
                yield self._read_message(delegate)
        else:
            if (headers.get("Expect") == "100-continue" and
                    not self._write_finished):
                self.stream.write(b"HTTP/1.1 100 (Continue)\r\n\r\n")
        if not skip_body:
            body_future = self._read_body(
                start_line.code if self.is_client else 0, headers, delegate)
            if body_future is not None:
                if self._body_timeout is None:
                    yield body_future
                else:
                    try:
                        yield gen.with_timeout(
                            self.stream.io_loop.time() + self._body_timeout,
                            body_future, self.stream.io_loop,
                            quiet_exceptions=iostream.StreamClosedError)
                    except gen.TimeoutError:
                        gen_log.info("Timeout reading body from %s",
                                        self.context)
                        self.stream.close()
                        raise gen.Return(False)
        self._read_finished = True
        if not self._write_finished or self.is_client:
            need_delegate_close = False
            with _ExceptionLoggingContext(app_log):
                delegate.finish()
        # If we're waiting for the application to produce an asynchronous
        # response, and we're not detached, register a close callback
        # on the stream (we didn't need one while we were reading)
        if (not self._finish_future.done() and
                self.stream is not None and
                not self.stream.closed()):
            self.stream.set_close_callback(self._on_connection_close)
            yield self._finish_future
        if self.is_client and self._disconnect_on_finish:
            self.close()
        if self.stream is None:
            raise gen.Return(False)
    except httputil.HTTPInputError as e:
        gen_log.info("Malformed HTTP message from %s: %s",
                        self.context, e)
        self.close()
        raise gen.Return(False)
    finally:
        if need_delegate_close:
            with _ExceptionLoggingContext(app_log):
                delegate.on_connection_close()
        header_future = None
        self._clear_callbacks()
    raise gen.Return(True)
```

重要方法，首先会调用 stream 的 read_until_regex 方法开始读取客户端请求传入的头数据（http 协议的header），返回一个 Future 对象，即 header_future，详解参考：[tornado_iostream#read_until_regex()]({{< ref "tornado_iostream.md/#tornadoiostreambaseiostreamread_until_regex" >}})；接着会判断是否有设置 header_timeout，而在该方法中，默认是 3600 秒，则开始请求头超时机制设计，详解参考：[tornado_gen#with_timeout()]({{< ref "tornado_gen.md/#tornadogenwith_timeout" >}})；

