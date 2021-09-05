# Attack on Tornado - BIO实现 🌪




<extoc></extoc>

## 前言

这一章中我们来用 `Python` 实现 BIO (Blocking IO) 也就是阻塞IO

## BIO

实际上 , 默认的 `socket` 就是阻塞的 , 所以我们可以直接这样来实现

`bio_server.py` 

```python
import socket

def server(host, port):
    sock = socket.socket()
    sock.bind((host, port))
    print('启动服务端...')
    # listen(5) 表示允许等待的最大数量
    sock.listen(5)
    conn, addr = sock.accept()
    # 真实场景下当然不会只接收1024个字节的数据就关闭掉, 而是会根据包的大小来进行处理
    data = conn.recv(1024).decode('utf-8')
    print(data)
    conn.close()
    sock.close()
    print('关闭服务端...')

if __name__ == '__main__':
    server('localhost', 8888)
```

`bio_client.py`

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

接下里我们让服务端可以一直不停的跑

##  循环BIO

我们可以直接加一个死循环 , 让服务端可以一直不断的处理 , 至于客户端我们并不需要更改

`bio_server.py`

```python
import socket

def server(host, port):
    sock = socket.socket()
    sock.bind((host, port))
    print('启动服务端...')
    # listen(5) 表示允许等待的最大数量
    sock.listen(5)
    while True:
        conn, addr = sock.accept()
        # 真实场景下当然不会只接收1024个字节的数据就关闭掉, 而是会根据包的大小来进行处理
        data = conn.recv(1024).decode('utf-8')
        print(data)
        conn.close()
    sock.close()
    print('关闭服务端...')


if __name__ == '__main__':
    server('localhost', 8888)
```

这样就可以不断的进行处理了 , 但是如果我们想要同时处理多个客户端连接这是不支持的

所以我们可以再进阶一下 , 每一个连接都用一个线程去进行处理 , 来实现同时处理多个连接

## 多线程BIO

`bio_server.py` 

```python
import socket
import threading

def handle_conn(conn, addr):
    # 真实场景下当然不会只接收1024个字节的数据就关闭掉, 而是会根据包的大小来进行处理
    data = conn.recv(1024).decode('utf-8')
    print('线程 %s 处理来自 %s-%s 的连接...' % (threading.currentThread().getName(), addr[0], addr[1]))
    print(data)
    conn.close()

def server(host, port):
    sock = socket.socket()
    sock.bind((host, port))
    print('启动服务端...')
    # listen(5) 表示允许等待的最大数量
    sock.listen(5)
    while True:
        conn, addr = sock.accept()
        print('客户端 %s-%s 连接成功, 等待发送数据...' % (addr[0], addr[1]))
        handle = threading.Thread(target=handle_conn, args=(conn, addr))
        handle.start()

    sock.close()
    print('关闭服务端...')

if __name__ == '__main__':
    server('localhost', 8888)
```



