# Attack on Tornado - IO Multiplexing实现 🌪

## 前言

这一章中我们来用 `Python` 实现 IO Multiplexing 也就是IO多路复用

## IO Multiplexing

实现IO多路复用需要借助到 `selectors` 

`selectors` 中为我们提供了 `select` , `poll` , `epoll` , `kqueue` , `devpoll` 

`iom_server.py`

```python
import socket
import selectors

def server(host, port):
    sock = socket.socket()
    sock.bind((host, port))
    print('启动服务端...')

    # listen(5) 表示允许等待的最大数量
    sock.listen(5)
    # 获取IO多路复用器, 这里会根据系统选择相应的多路复用器
    selector = selectors.DefaultSelector()
    # 将socket注册进多路复用器
    selector.register(sock, selectors.EVENT_READ)
    while True:
        # 进行select调用
        ready = selector.select()
        for r in ready:
            sock = r[0][0]
            conn, addr = sock.accept()
            data = conn.recv(1024).decode('utf-8')
            print(data)
            conn.close()

    sock.close()
    print('关闭服务端...')


if __name__ == '__main__':
    server('localhost', 8888)
```

`iom_client.py`

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

## 多线程IO Multiplexing

相信从前面的几篇实现 , 你已经轻车熟路了

`iom_server.py` 

```python
import socket
import selectors
import threading

def handle_conn(conn, addr):
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
    selector = selectors.DefaultSelector()
    selector.register(sock, selectors.EVENT_READ)
    while True:
        ready = selector.select()
        for r in ready:
            sock = r[0][0]
            conn, addr = sock.accept()
            print('客户端 %s-%s 连接成功, 等待发送数据...' % (addr[0], addr[1]))
            handle = threading.Thread(target=handle_conn, args=(conn, addr))
            handle.start()
    sock.close()
    print('关闭服务端...')

if __name__ == '__main__':
    server('localhost', 8888)
```



