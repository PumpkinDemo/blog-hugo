---
title: Python Socket Programming
date: 2021-12-07 19:47:44
tags: [python]
---

How to build socket server and socket client by python?

The first think to decide is which protocol to use, TCP or UDP?

The main difference is that TCP is connection-oriented while UDP is connectionless.

In detail, the server or the client must maintain a connection if they use TCP.

How is this reflected in socket programming?

### For UDP

For example, if we want to send a message using UDP

```python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
# s.bind('0.0.0.0', 2333)
address = ('123.123.123.123', 2333)
s.sendto(b'hi~', address)
```

Only the target address is required.

And the receiver (server) only need to specify its address using `bind` as a unique identifier on the internet so that the message sender could know where it is.

```python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.bind('0.0.0.0', 2333)
bufsize = 8192
while 1:
    message, address = s.recvfrom(bufsize)
    print(message)
    s.sendto(b'hello', address)
```

The main characteristic of UDP is that every sending of message is isolated and context-free.

Just `sendto(message, address)` no matter the sender is the server or the client, the boundary between the server and the client is vague and the distinguishment of server and client is unimportant.

### For TCP

But when it comes to TCP, things becomes difficult.

For the message sender:

```python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
# s.bind('0.0.0.0', 2333)
address = ('123.123.123.123', 2333)
s.connect(address)
bufsize = 8192
while 1:
    s.send(b'hi~')
    msg = s.recv(bufsize)
    print(msg)
s.close()
```

Before sending the message, it should do `connect` first and close the connection after communication.

The target address would be sepcify only when the connection is created.

For the message receiver (server):

```python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.bind('0.0.0.0', 2333)
max_conn = 5
s.listen(max_conn)
while 1:
    conn, address = s.accept()
    bufsize = 1024
    while 1:
        try:
            msg = conn.recv(bufsize)
            print(msg)
            conn.send(b'hello')
        except:
            print('connection break')
            break
    conn.close()
```

The server should wait for connections and maintain them until ending.

In fact, a connection is also a `socket.socket` object (a file discriptor).

### Summary

for UDP:

1. init
2. `bind` address if is receiver (server)
3. `sendto` or `recvfrom`

for TCP client:

1. init
2. `connect` to a server
3. `send` and `recv` messages
4. `close` the connection

for TCP server:
1. init
2. `bind` an address
3. `listen` continuously
4. `accept` a connection
5. `send` and `recv` messages
6. `close` the connection