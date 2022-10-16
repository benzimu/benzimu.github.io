# tornado concurrent 实现解析


## tornado concurrent 实现解析

用于处理线程和 Futures 的工具。Futures 是 python3.2 中 concurrent.futures 包引入的并发编程模式。在本模块中，定义了一个兼容的 Future 类，它被设计用于协同工作，还有一些用于与 concurrent.futures 包交互的实用函数。

Future 的设计目标是作为协程（coroutine）和 IOLoop 的媒介，从而将协程和 IOLoop 关联起来。

Future 是异步操作结果的占位符，用于等待结果返回。通常作为函数 IOLoop.add_future() 的参数或 gen.coroutine 协程中 yield 的返回值。

等到结果返回时，外部可以通过调用 set_result() 设置真正的结果，将结果保存在 Future 内存中，然后调用所有回调函数，恢复协程的执行，最后通过 result() 获取结果。

Future 类通过 self.\_done 的值来判断本次 Future 操作是否结束，默认为 False，当调用 set_result() 设置结果、或者调用 set_exc_info() 设置异常时会调用 _set_done() 修改其值，即表示本次操作已经完成。

### tornado.concurrent.Future.set_result()

```python
def set_result(self, result):
    # 将结果保存到 self._result
    self._result = result
    # 修改 self._done，并调用所有回调函数
    self._set_done()
```

### tornado.concurrent.Future._set_done()

```python
def _set_done(self):
    # 修改 self._done 值，表示本次操作已完成
    self._done = True
    # 循环执行 self._callbacks 中的回调函数，通过 add_done_callback() 定义
    for cb in self._callbacks:
        try:
            cb(self)
        except Exception:
            app_log.exception('Exception in callback %r for %r',
                                cb, self)
    self._callbacks = None
```

### tornado.concurrent.Future.add_done_callback()

```python
def add_done_callback(self, fn):
    # 添加本次 Future 操作完成时的回调函数，如果 Future 还没结束，则将回调函数加入
    # self._callbacks，以便结束时调用，如果已经结束，则直接运行回调函数
    if self._done:
        fn(self)
    else:
        self._callbacks.append(fn)
```

### tornado.concurrent.Future.result()

```python
def result(self, timeout=None):
    # 清理日志
    self._clear_tb_log()
    # 判断 self._result 结果是否为None，通过 set_result() 设置
    if self._result is not None:
        return self._result
    # 判断是否有异常信息，通过 set_exc_info() 设置
    if self._exc_info is not None:
        try:
            raise_exc_info(self._exc_info)
        finally:
            self = None
    # 检查是否结束
    self._check_done()
    return self._result
```

### tornado.concurrent.chain_future()

```python
def chain_future(a, b):
    # 将两个Future对象关联在一起，一个完成，另一个也完成
    # “a”的结果（成功或失败）将被复制到“b”，除非在“a”结束之前，“b”已经完成或被取消。
    def copy(future):
        assert future is a
        # 如果b已经完成或者取消，则直接返回
        if b.done():
            return
        if (isinstance(a, TracebackFuture) and
                isinstance(b, TracebackFuture) and
                a.exc_info() is not None):
            b.set_exc_info(a.exc_info())
        elif a.exception() is not None:
            b.set_exception(a.exception())
        else:
            b.set_result(a.result())
    # 将 copy()添加到 a 的回调列表中
    a.add_done_callback(copy)
```

首先会调用 a.add_done_callback(copy)，若 a 已经完成则直接运行 copy 函数，否则加入到回调列表中等到 a 完成时再运行。copy 函数将两个 Future 对象联系到了一起，用作结果返回、超时处理。

