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

    def handle_conn(conn, mask):
        data = conn.recv(1024).decode('utf-8')
        print(data)
        conn.close()
        selector.unregister(conn)

    def accept(sock, mask):
        conn, addr = sock.accept()
        selector.register(conn, EVENT_READ, handle_conn)

    selector.register(sock, EVENT_READ, accept)
    print('启动服务端...')
    # 循环处理已就绪事件
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
