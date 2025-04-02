---
title: "Python多播"
date: 2025-04-02T17:39:49+08:00
tags: [python]
---
多播协议是个好东西。
服务端
```python
import socket
import struct

# 多播地址和端口
MULTICAST_GROUP = '239.0.0.110'
SERVER_PORT = 12345

# 创建 UDP 套接字
server_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)
# 允许地址重用
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
# 绑定到指定端口
server_socket.bind(('', SERVER_PORT))

# 将套接字加入多播组
group = socket.inet_aton(MULTICAST_GROUP)
mreq = struct.pack('4sL', group, socket.INADDR_ANY)
server_socket.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP, mreq)

print(f"Listening on multicast address {MULTICAST_GROUP}:{SERVER_PORT}...")

try:
    while True:
        # 接收数据
        data, client_address = server_socket.recvfrom(1024)
        message = data.decode('utf-8')
        print(f"Received message from {client_address}: {message}")

        # 向客户端发送响应
        response = "Message received successfully!"
        server_socket.sendto(response.encode('utf-8'), client_address)
except KeyboardInterrupt:
    print("Shutting down...")
finally:
    server_socket.close()

```
客户端
```python
import socket

MULTICAST_GROUP = '239.0.0.110'
SERVER_PORT = 12345

client_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
client_socket.settimeout(3)

message = "Hello, I'm a client!"

try:
    client_socket.sendto(message.encode('utf-8'), (MULTICAST_GROUP, SERVER_PORT))
    print(f"Sent message to {MULTICAST_GROUP}:{SERVER_PORT}")
    try:
        data, server_address = client_socket.recvfrom(1024)
        print(f"Received response from {server_address}: {data.decode('utf-8')}")
    except socket.timeout:
        print("No response from server. Server may be unavailable.")
except socket.error as e:
    print(f"Error: {e}")
finally:
    client_socket.close()
```