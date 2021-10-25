# Attack on Tornado - SIGIO实现 🌪














<extoc></extoc>

## 前言

这一章中我们来用 `Python` 实现 `SIGIO` , 也就是信号驱动IO (`Signal driven IO`) , 我们需要借助两个标准库 : 

- signal : 信号支持
- fcntl : 设置非阻塞文件描述符

实际上信号驱动IO对于 `TCP` 套接字的作用并不大 , 因为在 `TCP` 中 `SIGIO` 信号产生的条件有很多 : 

- 监听套接字上某个连接请求已经完成
- 某个断连请求已经发起
- 某个断连请求已经完成
- 某个连接半关闭
- 数据到达套接字
- 数据已经从套接字发送走
- 套接字上发生异步错误

当然我们还是可以通过 `SIGIO` 来监听套接字

而在 `UDP` 中 `SIGIO` 信号产生的条件仅仅 : 

- 数据到达套接字
- 套接字上发生异步错误

## UDP SIGIO

我们先来实现一个 `UDP` 的 `SIGIO` 

`udp_sigio_server.py`

```python
import os
import time
import fcntl
import signal
import socket

def server(host, port):
    sock = socket.socket(type=socket.SOCK_DGRAM)
    sock.setblocking(False)
    sock.bind((host, port))

    def receive_signal(signum, stack):
        data, addr = sock.recvfrom(1024)
        data = data.decode('utf-8')
        print(data)

    signal.signal(signal.SIGIO, receive_signal)
    fcntl.fcntl(sock.fileno(), fcntl.F_SETOWN, os.getpid())
    fcntl.fcntl(sock.fileno(), fcntl.F_SETFL, fcntl.fcntl(sock.fileno(), fcntl.F_GETFL, 0) | fcntl.FASYNC)

    while True:
        print('Waiting...')
        time.sleep(3)

if __name__ == '__main__':
    server('localhost', 8888)
```

`http_client.py.py`

```python
import json
import socket
import datetime

def client(host, port):
    sock = socket.socket(type=socket.SOCK_DGRAM)
    data = {
        'send_user': 'Lyon',
        'send_time': datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
        'send_msg': 'BIO test...'
    }
    sock.sendto(json.dumps(data).encode('utf-8'), (host, port))
    sock.close()
    print('客户端发送数据成功...')

if __name__ == '__main__':
    client('localhost', 8888)
```

## TCP SIGIO


`tcp_sigio_server.py`

```python
import os
import time
import fcntl
import socket
import signal


def server(host, port):
    sock = socket.socket()
    sock.setblocking(False)
    sock.bind((host, port))
    sock.listen()

    def receive_signal(signum, stack):
        try:
            conn, addr = sock.accept()
            data = conn.recv(1024).decode('utf-8')
            print(data)
            conn.close()
        except BlockingIOError:
            pass

    signal.signal(signal.SIGIO, receive_signal)
    fcntl.fcntl(sock.fileno(), fcntl.F_SETOWN, os.getpid())
    fcntl.fcntl(sock.fileno(), fcntl.F_SETFL, fcntl.fcntl(sock.fileno(), fcntl.F_GETFL, 0) | fcntl.FASYNC)

    while True:
        print('Waiting...')
        time.sleep(3)


if __name__ == '__main__':
    server('localhost', 8888)
```

TODO : 单线程下存在问题 , 暂时还没弄清楚

在 `Mac` 环境下 , 多线程版本暂时没发现问题 

`thread_tcp_sigio_server.py`

```python
import os
import time
import fcntl
import socket
import signal
import threading


def handle_conn(conn, addr):
    while True:
        try:
            data = conn.recv(1024).decode('utf-8')
            print('线程 %s 处理来自 %s-%s 的连接...' % (threading.currentThread().getName(), addr[0], addr[1]))
            print(data)
            conn.close()
            break
        except BlockingIOError:
            pass
    conn.close()


def server(host, port):
    with socket.socket() as sock:
        sock.setblocking(False)
        sock.bind((host, port))
        sock.listen()

        def receive_signal(signum, stack):
            print('SIGIO...')
            try:
                conn, addr = sock.accept()
                handle = threading.Thread(target=handle_conn, args=(conn, addr))
                handle.start()
            except BlockingIOError:
                pass

        signal.signal(signal.SIGIO, receive_signal)
        fcntl.fcntl(sock.fileno(), fcntl.F_SETOWN, os.getpid())
        fcntl.fcntl(sock.fileno(), fcntl.F_SETFL, fcntl.fcntl(sock.fileno(), fcntl.F_GETFL, 0) | fcntl.FASYNC)

        while True:
            print('Waiting...')
            time.sleep(3)

if __name__ == '__main__':
    server('localhost', 8888)
```

`tcp_sigio_client.py`

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

