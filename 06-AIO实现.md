# Attack on Tornado - AIO实现 🌪






<extoc></extoc>

## 前言

在上一章中 , 我们已经知道信号驱动IO对于我们的 `TCP` 应用来说 , 支持并不是很好

且不幸的是在 `Linux` 上的异步IO支持并不成熟 , 虽然相对来说 `Windows` 上可能要成熟一些 , 但是我们的服务更多的是在 `Linux` 环境下运行 

在第4章的 `IO-Multiplexing实现` 中 , 我们通过 `selectors` 实现了基于事件的处理方式 , 实际上 `selectors` 还支持注册回调函数 , 以此来实现我们应用层的异步IO

我们将每一个IO操作都在回调函数中完成 , 当一个IO事件发生时 , 通过调用回调函数 , 达到异步执行的效果 , 但是注意这个回调并不是由操作系统完成的 , 而是由我们的应用程序来完成的 , 毕竟我们并不是基于操作系统的异步IO来实现的

## 回调函数

`aio_server.py`

```python
import socket
from selectors import DefaultSelector, EVENT_READ


def server(host, port):
    selector = DefaultSelector()
    sock = socket.socket()
    sock.bind((host, port))
    sock.listen(5)
    sock.setblocking(False)

    # 定义请求回调
    def accept(sock, mask):
        conn, addr = sock.accept()

        # 定义链接回调
        def handle_conn(conn, mask):
            data = conn.recv(1024).decode('utf-8')
            print(data)
            conn.close()
            selector.unregister(conn)

        selector.register(conn, EVENT_READ, handle_conn)

    selector.register(sock, EVENT_READ, accept)
    print('启动服务端...')
    while True:
        ready = selector.select()
        for event, mask in ready:
            callable = event.data
            callable(event.fileobj, mask)


if __name__ == '__main__':
    server('localhost', 8888)
```

`http_client.py` 

```python
import socket
import json
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
    sock.close()
    print('客户端发送数据成功...')

if __name__ == '__main__':
    client('localhost', 8888)
```

我们通过回调的方式 , 让IO操作都变成了异步操作 , 但是 `recv` 也就是数据从内核拷贝到程序中还是同步的

## 回调地狱

虽然通过回调函数 + IO多路复用实现了异步IO , 但是这种方式实际上存在 `回调地狱(callback hell)` 问题

在上面的代码中 , 我们的 `accept` 和 `handle_conn` 已经出现了两层回调 , 我们可以加入一些代码来看看

`aio_callback_server.py`

```python
import socket
from selectors import DefaultSelector, EVENT_READ, EVENT_WRITE

def server(host, port):
    selector = DefaultSelector()
    sock = socket.socket()
    sock.bind((host, port))
    sock.listen(5)
    sock.setblocking(False)

    def accept(sock, mask):
        print('Accept ...')
        conn, addr = sock.accept()

        def connect(conn, mask):
            print('Connect ...')
            data = conn.recv(1024).decode('utf-8')

            def write(conn, mask):
                print('Write ...')

                def success(conn, mask):
                    print('Success ...')
                    selector.unregister(conn)
                    conn.close()

                selector.modify(conn, EVENT_READ, success)

            selector.modify(conn, EVENT_WRITE, write)

            conn.send(b'ok')

        selector.register(conn, EVENT_READ, connect)

    selector.register(sock, EVENT_READ, accept)
    print('启动服务端...')
    while True:
        ready = selector.select()
        for event, mask in ready:
            callable = event.data
            callable(event.fileobj, mask)

if __name__ == '__main__':
    server('localhost', 8888)
```

再调整一下客户端

`aio_callback_client.py`

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

我们只不过是在一个请求中多添加了几次读写 , 代码就变得有意思了 , 我们省略一些来看看服务端是个什么样子的 

`aio_callback_server.py`

```python
    def accept(sock, mask):
        print('Accept ...')
        def connect(conn, mask):
            print('Connect ...')
            def write(conn, mask):
                print('Write ...')
                def success(conn, mask):
                    print('Success ...')
                selector.modify(conn, EVENT_READ, success)
            selector.modify(conn, EVENT_WRITE, write)
        selector.register(conn, EVENT_READ, connect)
    selector.register(sock, EVENT_READ, accept)
```

这就清晰了 , 当层次一旦变多 , 那么回调地狱就会突显 , 我们的代码即将往非人类的方向发展 , 而实际情况就是我们的业务代码通常伴随着大量的网络IO与磁盘IO , 基本上无法避免多层次情况

所以回调实现的基础上 , 衍生出了基于协程的解决方案