# Attack on Tornado - IOLoop 🌪












<extoc></extoc>

## 介绍

`IOLoop` 是异步非阻塞模型的关键所在 , 我们以 `asyncio` 为例来分析 `Tornado` 的源码实现

## 开始

我们会围绕官方的一个例子来进行源码分析 , 示例如下 : 

```python
import tornado.ioloop
import tornado.web


class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("Hello, world")


if __name__ == "__main__":
    application = tornado.web.Application([
        (r"/", MainHandler),
    ])
    application.listen(8888)
    tornado.ioloop.IOLoop.current().start()
```

为了节省篇幅 , 会省略部分官方注释 , 因为本节的重点不在 `tornado.web.Application` 这个应用对象上 , 所以我们在这节中不会过多的去分析 `Application`

## Configurable

在开始之前 , 我们需要先看一下 `Configurable` 的源码 , 因为它改变了我们正常对象初始化流程 , 且在 `Tornado` 中基本大多数类都继承了 `Configurable` 

```python
# type: tornado.util.Configurable

class Configurable(object):
    __impl_class = None  # type: Optional[Type[Configurable]]
    __impl_kwargs = None  # type: Dict[str, Any]

    def __new__(cls, *args: Any, **kwargs: Any) -> Any:
        base = cls.configurable_base()
        init_kwargs = {}  # type: Dict[str, Any]
        if cls is base:
            impl = cls.configured_class()
            if base.__impl_kwargs:
                init_kwargs.update(base.__impl_kwargs)
        else:
            impl = cls
        init_kwargs.update(kwargs)
        if impl.configurable_base() is not base:
            # The impl class is itself configurable, so recurse.
            return impl(*args, **init_kwargs)
        # 调用object的__new__方法
        instance = super(Configurable, cls).__new__(impl)
        # 调用实例的 initialize 方法
        # 在Tornado中, 继承了 Configurable 的类的 __init__ 基本都是 pass 带过
        # 真正的初始化都放在 initialize 方法中, 所以如果你发现了 initialize , 那么没错, 它就是真正的 `__init__`
        instance.initialize(*args, **init_kwargs)
        return instance
```

## listen

`application.listen` 

```python
def listen(self, port: int, address: str = "", **kwargs: Any) -> HTTPServer:
    # type: tornado.httpserver.HTTPServer(TCPServer, Configurable, httputil.HTTPServerConnectionDelegate)
    # type: tornado.tcpserver.TCPServer
    server = HTTPServer(self, **kwargs)
    # type: tornado.tcpserver.TCPServer.listen
    server.listen(port, address)
    return server
```

`tornado.tcpserver.TCPServer.listen` 

```python
def listen(self, port: int, address: str = "") -> None:
    # type: tornado.netutil.bind_sockets
    # bind_sockets是用来创建套接字的, 默认会根据系统需要创建相应的套接字
	# 以Mac为例, 会创建出 AF_INET 和 AF_INET6 两个套接字
    """
    sockets : [
    <socket.socket fd=5, family=AddressFamily.AF_INET, type=SocketKind.SOCK_STREAM, proto=6, laddr=('0.0.0.0', 8888)>, 
    <socket.socket fd=6, family=AddressFamily.AF_INET6, type=SocketKind.SOCK_STREAM, proto=6, laddr=('::', 8888, 0, 0)>
    ]
    """
    sockets = bind_sockets(port, address=address)
    # type: tornado.tcpserver.TCPServer.add_sockets
    self.add_sockets(sockets)
```

`tornado.tcpserver.TCPServer.add_sockets`

```python
def add_sockets(self, sockets: Iterable[socket.socket]) -> None:
    for sock in sockets:
        # self._sockets, self._handlers 是两个dict
        # sock.fileno()会返回一个文件描述符, 然后以文件描述符为key, socket对象为value存储到server对象中
        # sock type: socket.socket() 
        self._sockets[sock.fileno()] = sock
        # type: tornado.netutil.add_accept_handler
        # 在add_accept_handler中会完成IOLoop的创建, 以及套接字绑定到事件循环上
        self._handlers[sock.fileno()] = add_accept_handler(
            sock, self._handle_connection
        )
        # slef._handle_connection 看名字我们也能知道这是处理连接的回调函数
```

## add_accept_handler

`tornado.netutil.add_accept_handler` 

```python
def add_accept_handler(
        sock: socket.socket, callback: Callable[[socket.socket, Any], None]
) -> Callable[[], None]:
    # type: tornado.platform.asyncio.AsyncIOMainLoop(BaseAsyncIOLoop)
    # type: tornado.platform.asyncio.BaseAsyncIOLoop(IOLoop)
    # type: tornado.ioloop.IOLoop
    # IOLoop.current(instance=True) => tornado.ioloop.IOLoop.current
    """
    在这里有一个关键的地方就是在 AsyncIOMainLoop(make_current=True)的时候
    def initialize(self, **kwargs: Any) -> None:  # type: ignore
        这里会使用asyncio去获取到当前的事件循环, 并作为参数, 到达BaseAsyncIOLoop.initialize
        super().initialize(asyncio.get_event_loop(), **kwargs)
    """
    io_loop = IOLoop.current() 
	# 其他部分暂时省略, 我们先看看IOLoop是怎么创建的
```

`tornado.platform.asyncio.AsyncIOMainLoop.initialize`

```python
def initialize(self, **kwargs: Any) -> None:  # type: ignore
    # 这里很关键, 因为它其实使用了asyncio去获取当前线程的事件循环, 然后将拿到的eventloop作为参数传入了BaseAsyncIOLoop.initialize
    super().initialize(asyncio.get_event_loop(), **kwargs)
```

`tornado.platform.asyncio.BaseAsyncIOLoop.initialize` 

```python
def initialize(self, asyncio_loop: asyncio.AbstractEventLoop, **kwargs: Any) -> None:
    # 这里可以看到官方的标注是asyncio.AbstractEventLoop, 实际上asyncio这里的实现要复杂得多, 我们可以放在后面的文章中
    # 如果你看过asyncio的源码, 那在这里千万别搞混了, 因为tornado.ioloop.IOLoop实际上是讲asyncio中的loop作为一个或者说两个属性来使用的, 并不是继承的方式, 所以两者存在一些差异
    self.asyncio_loop = asyncio_loop
    self.selector_loop = asyncio_loop
    if hasattr(asyncio, "ProactorEventLoop") and isinstance(
        asyncio_loop, asyncio.ProactorEventLoop  # type: ignore
    ):
        self.selector_loop = AddThreadSelectorEventLoop(asyncio_loop)
    self.handlers = {}
    self.readers = set()  # type: Set[int]
    self.writers = set()  # type: Set[int]
    self.closing = False
    for loop in list(IOLoop._ioloop_for_asyncio):
        if loop.is_closed():
            del IOLoop._ioloop_for_asyncio[loop]
    # 在这里就会讲asyncio_loop存放到IOLoop的类变量中
    # 这里要注意的, 在其他的地方可能会使用asyncio的get_event_loop方法来获取asyncio_loop
    # 这样再通过asyncio_loop就可以到类变量中拿到tornado.platform.asyncio.AsyncIOMainLoop
    # 你可以在IOLoop的current中看到这块代码
    IOLoop._ioloop_for_asyncio[asyncio_loop] = self
    self._thread_identity = 0
    super().initialize(**kwargs)
    # assign_thread_identity将会在IOLoop.start()的时候被回调
    def assign_thread_identity() -> None:
        self._thread_identity = threading.get_ident()
    # type: tornado.platform.asyncio.BaseAsyncIOLoop.add_callback
    # 这个add_callback其实使用的就是asyncio中loop的add_callback, 它的做用哪个就是添加一个回调函数到事件循环中
    self.add_callback(assign_thread_identity)
```

这种我们就大费周章的拿到了 `io_loop` , 所以我们再回到 `tornado.netutil.add_accept_handler` 

```python
def add_accept_handler(
        sock: socket.socket, callback: Callable[[socket.socket, Any], None]
) -> Callable[[], None]:
    # type: tornado.platform.asyncio.AsyncIOMainLoop(BaseAsyncIOLoop)
    # type: tornado.platform.asyncio.BaseAsyncIOLoop(IOLoop)
    # type: tornado.ioloop.IOLoop
    # IOLoop.current(instance=True) => tornado.ioloop.IOLoop.current
    io_loop = IOLoop.current() 
    removed = [False]
	# 这里我们先省略accept_handler, 等到真正使用的时候再分析
    def accept_handler(fd: socket.socket, events: int) -> None:
        pass

    def remove_handler() -> None:
        io_loop.remove_handler(sock)
        removed[0] = True
	# type: tornado.platform.asyncio.BaseAsyncIOLoop.add_handler
    # 在这里注册了一个读事件
    io_loop.add_handler(sock, accept_handler, IOLoop.READ)
    return remove_handler
```

## add_handler

`tornado.platform.asyncio.BaseAasyncIOLoop.add_handler`

```python
def add_handler(
    self, fd: Union[int, _Selectable], handler: Callable[..., None], events: int
) -> None:
    fd, fileobj = self.split_fd(fd)
    if fd in self.handlers:
        raise ValueError("fd %s added twice" % fd)
    # 这里self.handlers中存储的是以文件描述符为key, 套接字对象和 accept_handler 组成的元祖为value的dict
    self.handlers[fd] = (fileobj, handler)
    if events & IOLoop.READ:
        # type: asyncio.selector_evnets.add_reader
        # 添加一个读事件的回调
        self.selector_loop.add_reader(fd, self._handle_events, fd, IOLoop.READ)
        self.readers.add(fd)
    if events & IOLoop.WRITE:
        self.selector_loop.add_writer(fd, self._handle_events, fd, IOLoop.WRITE)
        self.writers.add(fd)
```

`asyncio.selector_events.add_reader`

```python
def add_reader(self, fd, callback, *args):
    self._ensure_fd_no_transport(fd)
    # callback: tornado.platform.asyncio.BaseAsyncIOLoop._handle_events
    # type: asyncio.selector_events.BaseSelectorEventLoop._add_reader
    return self._add_reader(fd, callback, *args)
```

`asyncio.selector_events.BaseSelectorEventLoop._add_reader`

```python
def _add_reader(self, fd, callback, *args):
    self._check_closed()
    handle = events.Handle(callback, args, self)
    try:
        key = self._selector.get_key(fd)
        except KeyError:
            # self._selector是一个多路复用器, 因为我们主要是在Linux上运行
            # 所以 type: selectors.EpollSelector
            # 在Mac上使用的是 selectors.KqueueSelector
            # 这里会将一个读事件注册到IO多路复用器中
            self._selector.register(fd, selectors.EVENT_READ,
                                    (handle, None))
        else:
            mask, (reader, writer) = key.events, key.data
            self._selector.modify(fd, mask | selectors.EVENT_READ,
                                      (handle, writer))
            if reader is not None:
                reader.cancel()
```

实际上 , 到这里 , 我们的初始化流程就已经完成了

然后我们调用 `tornado.ioloop.IOLoop.current().start()` 

如果光看示例 , 你可以会疑惑 , 我什么现在又调用 `IOLoop.current()` , 且没有明显的将事件循环和 `Tornado` 的应用绑定起来 , 所以到这里你应该就没有疑惑了 , 因为在 `listen` 中 , 做了太多的事情了

```python
tornado.ioloop.current().start()
# type: tornado.platform.asyncio.BaseAsyncIOLoop.start
def start(self) -> None:
    try:
        old_loop = asyncio.get_event_loop()
    except (RuntimeError, AssertionError):
        old_loop = None  # type: ignore
    try:
        self._setup_logging()
        # type: asyncio.base_events.BaseEventLoop.run_forever
        asyncio.set_event_loop(self.asyncio_loop)
        self.asyncio_loop.run_forever()
    finally:
        asyncio.set_event_loop(old_loop)
```

`asyncio.base_events.BaseEventLoop.run_forever`

```python
def run_forever(self):
    """Run until stop() is called."""
    self._check_closed()
    if self.is_running():
        raise RuntimeError('This event loop is already running')
    if events._get_running_loop() is not None:
        raise RuntimeError(
            'Cannot run the event loop while another loop is running')
    self._set_coroutine_wrapper(self._debug)
    self._thread_id = threading.get_ident()
    if self._asyncgens is not None:
        old_agen_hooks = sys.get_asyncgen_hooks()
        sys.set_asyncgen_hooks(firstiter=self._asyncgen_firstiter_hook,
                               finalizer=self._asyncgen_finalizer_hook)
    try:
        events._set_running_loop(self)
        while True:
            # 从这里整个线程就进入了死循环当中
            # type: asyncio.base_events.BaseEventLoop._run_once
            # 每当有新的事件产生时, _run_once就会被循环一次
            self._run_once()
            if self._stopping:
                break
    finally:
        self._stopping = False
        self._thread_id = None
        events._set_running_loop(None)
        self._set_coroutine_wrapper(False)
        if self._asyncgens is not None:
            sys.set_asyncgen_hooks(*old_agen_hooks)
```

## _handle_events

在我们分析 `_run_once` 之前 , 我们需要先看一下 `_handle_events` , 因为它这才是事件发生时的回调函数 

`tornado.platform.asyncio.BaseAsyncIOLoop._handle_events` 

```python
def _handle_events(self, fd: int, events: int) -> None:
    # 其实这里的操作很简单, 就是根据监听的文件描述符, 到handlers中获取socket和handler_func
    # 这里的handler_func就是accept_handler, 也就是我们在调用add_accept_handler的时候里面调用的io_loop.add_handler(sock, accept_handler, IOLoop.READ) 这里放进去的
    # 所以我们到这里可以去看accept_handler了
    fileobj, handler_func = self.handlers[fd]
    handler_func(fileobj, events)
```

`accept_handler`

```python
# type: tornado.tcpserver.TCPServer._handle_connection
callback = self._handle_connection
def accept_handler(fd: socket.socket, events: int) -> None:
    for i in range(_DEFAULT_BACKLOG):
        if removed[0]:
            # The socket was probably closed
            return
        try:
            # 其实就是当有连接过来时, 就拿到conn和address, 然后进行回调
            # 回调就是在add_sockets的时候设置的 self._handle_connection
            connection, address = sock.accept()
        except BlockingIOError:
            # EWOULDBLOCK indicates we have accepted every
            # connection that is available.
            return
        except ConnectionAbortedError:
            # ECONNABORTED indicates that there was a connection
            # but it was closed while still in the accept queue.
            # (observed on FreeBSD).
            continue
        callback(connection, address)
```

## _handle_connection

`tornado.tcpserver.TCPServer._handle_connection` 

```python
def _handle_connection(self, connection: socket.socket, address: Any) -> None:
    # 我们把ssl部分直接略过
    if self.ssl_options is not None:
        pass
    try:
        if self.ssl_options is not None:
            stream = SSLIOStream(
                connection,
                max_buffer_size=self.max_buffer_size,
                read_chunk_size=self.read_chunk_size,
            )  # type: IOStream
        else:
            # type: tornado.iostream.IOStream
            stream = IOStream(
                connection,
                max_buffer_size=self.max_buffer_size,
                read_chunk_size=self.read_chunk_size,
            )
		# type: tornado.httpserver.HTTPServer.handle_stream
        future = self.handle_stream(stream, address)
        if future is not None:
            IOLoop.current().add_future(
                gen.convert_yielded(future), lambda f: f.result()
            )
    except Exception:
        app_log.error("Error in connection callback", exc_info=True)
```

`tornado.httpserver.HTTPServer.handle_stream` 

```python
def handle_stream(self, stream: iostream.IOStream, address: Tuple) -> None:
    context = _HTTPRequestContext(
        stream, address, self.protocol, self.trusted_downstream
    )
    # 创建连接对象
    conn = HTTP1ServerConnection(stream, self.conn_params, context)
    self._connections.add(conn)
    # type: tornado.http1connection.HTTP1ServerConnection.start_serving
    # self: tornado.httpserver.HTTPSever 继承了 tornado.httputil.HTTPServerConnectionDelegate
    conn.start_serving(self)
```

`tornado.http1connection.HTTP1ServerConnection.start_serving`

```python
def start_serving(self, delegate: httputil.HTTPServerConnectionDelegate) -> None:
    assert isinstance(delegate, httputil.HTTPServerConnectionDelegate)
    # 构造future
    # delegate <=> HTTPServer
    # type: tornado.http1connection.HTTP1ServerConnection._server_request_loop
    # 将 coroutine 包装成 Task
    # 实际上到这里一个请求的流程就已经完成了, 在self._server_request_loop中主要是处理请求的细节, 我们放到后面的章节中再分析
    # 要注意的是, 这些Future都还没有真正执行
    fut = gen.convert_yielded(self._server_request_loop(delegate))
    self._serving_future = fut
    # self.stream.io_loop type: tornado.ioloop.IOLoop.add_future
    # 在add_future会调用Future.add_done_callback type: asyncio.futures.Future
    # add_done_callback方法会根据当前Future的状态来判断是否执行, 而最终执行还是使用的asyncio.base_events.BaseEventLoop.call_soon
    # 最后呢, call_soon再调用_call_soon, 将回调包装成events.Handle, 最后append到_ready队列中
    self.stream.io_loop.add_future(fut, lambda f: f.result())
```

`tornado.http1connection.HTTP1ServerConnection._server_request` 

```python
async def _server_request_loop(
    self, delegate: httputil.HTTPServerConnectionDelegate
) -> None:
    try:
        while True:
            conn = HTTP1Connection(self.stream, False, self.params, self.context)
            # delegate.start_request type: tornado.httpserver.HTTPServer.start_request
            request_delegate = delegate.start_request(self, conn)
            try:
                ret = await conn.read_response(request_delegate)
            except (
                iostream.StreamClosedError,
                iostream.UnsatisfiableReadError,
                asyncio.CancelledError,
            ):
                return
            except _QuietException:
                # This exception was already logged.
                conn.close()
                return
            except Exception:
                gen_log.error("Uncaught exception", exc_info=True)
                conn.close()
                return
            if not ret:
                return
            # 只要没有完成, 那么每次都会切换
            await asyncio.sleep(0)
    finally:
        delegate.on_close(self)
```

所以到这里 , 基本上我们就可以知道 , 所有的请求最终都会达到 `_ready` 中 , 现在我们就可以看看最核心的 `_run_once` 到底做了什么了

## _run_once

`asyncio.base_events.BaseEventLoop._run_once` 

```python
def _run_once(self):
    sched_count = len(self._scheduled)
    # self._scheduled 是关于定时任务的一些实现
    # 定时任务基本上是通过 asyncio.base_events.BaseEventLoop.call_at 去做的
    # 而在call_at中, 会有一个timer定时器来控制
    if (sched_count > _MIN_SCHEDULED_TIMER_HANDLES and
        self._timer_cancelled_count / sched_count >
            _MIN_CANCELLED_TIMER_HANDLES_FRACTION):
        # Remove delayed calls that were cancelled if their number
        # is too high
        new_scheduled = []
        for handle in self._scheduled:
            if handle._cancelled:
                handle._scheduled = False
            else:
                new_scheduled.append(handle)

        heapq.heapify(new_scheduled)
        self._scheduled = new_scheduled
        self._timer_cancelled_count = 0
    else:
        # Remove delayed calls that were cancelled from head of queue.
        while self._scheduled and self._scheduled[0]._cancelled:
            self._timer_cancelled_count -= 1
            handle = heapq.heappop(self._scheduled)
            handle._scheduled = False

    timeout = None
    # 刚启动进来self._ready其实是有东西的, 就是在listen的时候注册的 assign_thread_identity 函数
    # 当准备好的任务中有任务时, 就直接执行
    if self._ready or self._stopping:
        timeout = 0
    # 如果现在都没有, 
    elif self._scheduled:
        # Compute the desired timeout.
        when = self._scheduled[0]._when
        timeout = min(max(0, when - self.time()), MAXIMUM_SELECT_TIMEOUT)

    if self._debug and timeout != 0:
        t0 = self.time()
        event_list = self._selector.select(timeout)
        dt = self.time() - t0
        if dt >= 1.0:
            level = logging.INFO
        else:
            level = logging.DEBUG
        nevent = len(event_list)
        if timeout is None:
            logger.log(level, 'poll took %.3f ms: %s events',
                       dt * 1e3, nevent)
        elif nevent:
            logger.log(level,
                       'poll %.3f ms took %.3f ms: %s events',
                       timeout * 1e3, dt * 1e3, nevent)
        elif dt >= 1.0:
            logger.log(level,
                       'poll %.3f ms took %.3f ms: timeout',
                       timeout * 1e3, dt * 1e3)
    else:
        event_list = self._selector.select(timeout)
    # 这个时候event_list只有[(SelectorKey(fileobj=8, fd=8, events=1, data=(<Handle BaseSelectorEventLoop._read_from_self()>, None)), 1)]
    # 它是在BaseSelectorEventLoop初始化的时候调用的 _make_self_pipe方法放进去的
    # 而_read_from_self会一直等待套接字传输数据, 并读取数据
    # 你可能会想, 我们在listen的时候也注册了两个套接字, 为什么这个event_list没有它们
    # 这是因为现在还没有连接进来, 所以两个事件都还没有准备好
    # type: asyncio.selector_events.BaseSelectorEventLoop._process_events
    self._process_events(event_list)
    # 我们先省略下面的部分, 看看_process_events做了什么事情
```

`asyncio.selector_events.BaseSelectorEventLoop._process_events` 

```python
def _process_events(self, event_list):
    for key, mask in event_list:
        fileobj, (reader, writer) = key.fileobj, key.data
        # 这里的 key.data 就是我们在上面_add_reader方法中所注册的
        # 读事件 self._selector.register(fd, selectors.EVENT_READ, (handle, None))
        # 写事件 self._selector.modify(fd, mask | selectors.EVENT_READ, (handle, writer))
        # 其中 handle 是 events.Handle type: asyncio.events.Handle
        if mask & selectors.EVENT_READ and reader is not None:
            if reader._cancelled:
                self._remove_reader(fileobj)
            else:
                # 这个时候并不会真正的执行而是将这个回调函数, 放入 _reday 中
                # type: asyncio.base_events.BaseEventLoop._add_callback
                self._add_callback(reader)
        if mask & selectors.EVENT_WRITE and writer is not None:
            if writer._cancelled:
                self._remove_writer(fileobj)
            else:
                self._add_callback(writer)
```

`asyncio.base_events.BaseEventLoop._add_callback` 

```python
def _add_callback(self, handle):
    """Add a Handle to _scheduled (TimerHandle) or _ready."""
    assert isinstance(handle, events.Handle), 'A Handle is required here'
    if handle._cancelled:
        return
    assert not isinstance(handle, events.TimerHandle)
    # 也就是我所说的, 会加入到 _reday 队列中
    # type: asyncio.base_events.BaseEventLoop._ready
    # _ready : type: collections.queue
    self._ready.append(handle)
```

接下来我们再往 `_process_events` 下面看

```python
    end_time = self.time() + self._clock_resolution
    # 当有定时任务, 且定时任务到了执行时间时, 就会将它放入到self._ready中去执行
    # 实际上这里处理的任务, 基本都是TimerHandler
    while self._scheduled:
        handle = self._scheduled[0]
        if handle._when >= end_time:
            break
        handle = heapq.heappop(self._scheduled)
        handle._scheduled = False
        self._ready.append(handle)

    ntodo = len(self._ready)
    for i in range(ntodo):
        handle = self._ready.popleft()
        if handle._cancelled:
            continue
        # 无论我们是不是出于DEBUG模式, 都将执行handle._run()
        # handler type: asyncio.events.Handler._run() 
        if self._debug:
            try:
                self._current_handle = handle
                t0 = self.time()
                handle._run()
                dt = self.time() - t0
                if dt >= self.slow_callback_duration:
                    logger.warning('Executing %s took %.3f seconds',
                                   _format_handle(handle), dt)
            finally:
                self._current_handle = None
        else:
            handle._run()
    handle = None  # Needed to break cycles when an exception occurs.
```

`asyncio.events.Handle._run` 

```python
def _run(self):
    try:
        # 那么这里_callback就不用在说了, 就是去执行我们_ready队列中的所有任务了
        self._callback(*self._args)
    except Exception as exc:
        cb = _format_callback_source(self._callback, self._args)
        msg = 'Exception in callback {}'.format(cb)
        context = {
            'message': msg,
            'exception': exc,
            'handle': self,
        }
        if self._source_traceback:
            context['source_traceback'] = self._source_traceback
        # 如果捕捉到异常了, 同样的, 还是加入到事件循环中去执行
        # type: asyncio.base_events.BaseEventLoop.call_exception_handler
        self._loop.call_exception_handler(context)
    self = None  # Needed to break cycles when an exception occurs.

```

因为本文的核心重点并不在 `asyncio` 上 , 所以关于 `asyncio` 并没有过多解释

通过这一节源码分析 , 我们就可以知道 , 在 `Tornado` 或者说 `asyncio` 中 , 异步非阻塞模型的实现 , 实际上只是应用层的异步非阻塞 , 在系统层还是使用的IO多路复用也就是同步IO去实现的

且 `callback` 如果计算过久 , 同样也会造成整个线程阻塞 , 所以如果计算过长的函数 , 可以分成多个函数 , 或者利用线程池来处理

