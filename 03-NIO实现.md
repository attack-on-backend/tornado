# Attack on Tornado - NIO实现 🌪

## 前言

这一章中我们来用 `Python` 实现 NIO (Non-blocking IO) 也就是非阻塞IO

## NIO

在IO模型的讲解中 , 我们已经知道 , `recvfrom` 系统调用之后 , 进程并没有阻塞 , 内核马上返回给进程 , 如果数据还没准备好 , 此时会返回一个 `error`

而进程在返回之后 , 可以做别的事情 , 然后再发起 `recvfrom` 系统调用 , 一直重复直到拿到数据为止 , 所以我们需要不断的进行轮询

`nio_server.py` 

```python
import socket

def server(host, port):
    sock = socket.socket()
    # 将socket设置为非阻塞
    sock.setblocking(False)
    sock.bind((host, port))
    print('启动服务端...')
    
    # listen(5) 表示允许等待的最大数量
    sock.listen(5)

    conn_queue = []
    delete_conn = []
    while True:
        try:
            # 因为socket被设置成了非阻塞, 所以这里不会阻塞
            # 且当没有连接进来时, 这里也会一直error
            conn, addr = sock.accept()
            conn_queue.append((conn, addr))
        except BlockingIOError as e:
            for c in conn_queue:
                conn, addr = c
                try:
                    # 真实场景下当然不会只接收1024个字节的数据就关闭掉, 而是会根据包的大小来进行处理
                    # recv同样是非阻塞的, 所以也需要捕捉一下BlockingIOError
                    data = conn.recv(1024).decode('utf-8')
                    print(data)
                    delete_conn.append(c)
                    conn.close()
                except BlockingIOError:
                    pass
            for c in delete_conn:
                conn_queue.remove(c)
            delete_conn = []

    sock.close()
    print('关闭服务端...')

if __name__ == '__main__':
    server('localhost', 8888)
```

`nio_client.py`

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

当然关于连接队列 , 我们还可以进行优化 , 这里只是为了实现 , 就不做优化了

这样我们就实现了一个 NIO 模型 , 但是在服务端中 , 只是等待请求和准备数据是非阻塞的而已 , 而在处理请求的时候还是阻塞处理的 , 这样的话就导致在处理连接时 , 就无法再接受连接了 , 所以我们可以和 BIO 一样利用多线程来处理连接

## 多线程NIO

`nio_server.py` 

```python
import socket
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
    sock = socket.socket()
    # 将socket设置为非阻塞
    sock.setblocking(False)
    sock.bind((host, port))
    print('启动服务端...')

    # listen(5) 表示允许等待的最大数量
    sock.listen(5)

    while True:
        try:
            # 因为socket被设置成了非阻塞, 所以这里不会阻塞
            # 且当没有连接进来时, 这里也会一直error
            conn, addr = sock.accept()
            print('客户端 %s-%s 连接成功, 等待发送数据...' % (addr[0], addr[1]))
            handle = threading.Thread(target=handle_conn, args=(conn, addr))
            handle.start()
        except BlockingIOError as e:
            pass

    sock.close()
    print('关闭服务端...')

if __name__ == '__main__':
    server('localhost', 8888)
```



