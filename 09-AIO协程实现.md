# Attack on Tornado - AIO协程实现 🌪


<extoc></extoc>

## 前言

虽然在上一篇中我们通过协程解决了回调地狱问题 , 但是回调注册方式还是不够简单 , 因为作为用户来讲 , 我们不应该去关心注册回调事件

我们希望有一个回调控制器可以自动帮我们注册回调 , 我们只关心业务逻辑 , 而对我们而言业务逻辑必然处于协程中 , 那么这个控制器肯定是从事件循环入手了

## AIO协程进阶

我们可以定义一个基类 `Event` , 其中定义两个 `API` 分别控制 : 注册回调事件与处理回调事件 , 随后每种事件都继承基类 `Event` , 在使用时 , 直接将事件推入事件循环 , 统一由事件循环进行调度

```python
class Event:
    def _yield(self, loop, task):
        """用于注册回调"""
        raise NotImplementedError

    def _resume(self, loop, task):
        """处理回调, 并将回调函数放入准备队列"""
        raise NotImplementedError
```

我们定义两种事件 , `Accept` 和 `Read` 事件 , 用于接收请求并接收请求中的数据

```python
class ReadEvent(Event):
    def __init__(self, sock):
        self.sock = sock

    def _yield(self, loop, task):
        loop._read_wait(self.sock.fileno(), self, task)

    def _resume(self, loop, task):
        data = self.sock.recv(1024)
        loop._ready.append((task, data))


class AcceptEvent(Event):
    def __init__(self, sock):
        self.sock = sock

    def _yield(self, loop, task):
        loop._read_wait(self.sock.fileno(), self, task)

    def _resume(self, loop, task):
        r = self.sock.accept()
        loop._ready.append((task, r))
```

事件循环除了维护事件队列之外 , 还维护一个就绪任务队列用于执行跟事件无关的其他业务逻辑

```python
import select
from collections import deque

class EventLoop:
    def __init__(self):
        self._num_tasks = 0  # Total num of tasks
        self._ready = deque()  # Tasks ready to run
        self._read_waiting = {}  # Tasks waiting to read
        self._write_waiting = {}  # Tasks waiting to write

    def _io_poll(self):
        """
        Poll for I/O events and restart waiting tasks
        """
        if -1 in self._read_waiting:
            self._read_waiting.pop(-1)

        # 这里只是为了方便调试所以使用的select
        r_set, w_set, e_set = select.select(self._read_waiting, self._write_waiting, [])

        for r in r_set:
            event, task = self._read_waiting.pop(r)
            event._resume(self, task)

    def call_soon(self, task):
        self._ready.append((task, None))
        self._num_tasks += 1

    def _read_wait(self, fileno, event, task):
        """
        Add a event to the reading set
        """
        self._read_waiting[fileno] = (event, task)

    def run_forever(self):
        '''
        Run the task eventloop until there are no tasks
        '''
        while self._num_tasks:
            if not self._ready:
                self._io_poll()
            task, data = self._ready.popleft()
            try:
                # Run the coroutine to the next yield
                r = task.send(data)
                if isinstance(r, Event):
                    r._yield(self, task)
                else:
                    raise RuntimeError('unrecognized yield event')
            except StopIteration:
                self._num_tasks -= 1
```

因为我们需要循环不断的接收请求 , 所以我们还需要编写一个接收函数以及处理函数

```python
if __name__ == '__main__':
    import socket
    def handle_events(conn):
        while True:
            line = yield ReadEvent(conn)
            print(line.decode('utf-8'))
            conn.close()


    def server_loop(loop, host, port):
        sock = socket.socket()
        sock.bind((host, port))
        sock.listen(5)
        sock.setblocking(False)
        print('启动服务端...')
        while True:
            conn, a = yield AcceptEvent(sock)
            print('Got connection from ', a)
            loop.call_soon(handle_events(conn))


    loop = EventLoop()
    loop.call_soon(server_loop(loop, 'localhost', 8888))
    loop.run_forever()
```

当然还有 `client` 

```python
import time
import json
import socket
import datetime

def client(host, port):
    sock = socket.socket()
    sock.connect((host, port))
    data = {
        'send_user': 'Lyon',
        'send_time': datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
        'send_msg': 'BIO test...'
    }
    sock.send(json.dumps(data).encode('utf-8'))

    time.sleep(5)

    sock.send(json.dumps(data).encode('utf-8'))
    sock.close()
    print('客户端发送数据成功...')

if __name__ == '__main__':
    client('localhost', 8888)
```

就这样我们的优化就完成了 , 我们可以把其他跟事件无关的任务通过 `call_soon` 方法添加到 `_ready` 就绪队列中 , 就像 `server_loop` 

现在已经很接近 `asyncio` 了 , 只不过我们要达到像 `asyncio` 支持的那样 , 还需要做不少工作 , 我们的目的是去理解这种编程方式 , 所以下一篇我们直接介绍 `asyncio`

**参考资料**

- 《Python Cookbook》
