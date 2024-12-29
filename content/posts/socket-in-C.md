---
title: socket in C
date: 2021-08-18 19:34:30
tags: [C]
---

一些常用的函数:

```c
int socket(int domain, int type, int protocol);
```

`domain`可选值:

+ `PF_UNIX`, `PF_LOCAL`: 本地通信
+ `AF_INET`, `PF_INET`: IPv4 Internet
+ `PF_INET6`: IPv6 Internet
+ `PF_IPX`: IPX-Novell
+ `PF_NETLINK`: 内核用户界面设备

`type`可选值:

+ `SOCK_STREAM`: 流式套接字, TCP
+ `SOCK_DGRAM`: 数据包套接字, UDP
+ `SOCK_SEQPACKET`: 序列化包
+ `SOCK_RAW`: RAW类型, 提供原始网络协议访问
+ `SOCK_RDM`: 提供可靠的数据报文, 不过可能数据有乱序
+ `SOCK_PACKET`: 专用类型, 不能再通用程序中使用

`protocol`可选值:

+ 通常协议中只有一种特定类型, 这时protocol参数只能设置为0

返回值:

+ `sockfd`: 一个套接字描述符
+ -1: 发生了错误, 全局变量`errno`被设置为错误代码

<br>


```c
int bind(int sockfd, struct sockaddr *my_addr, int addrlen);
```

`sockfd`: 一个`socket()`函数返回的套接字描述符

`my_addr`: 一个`sockaddr`指针, 包含地址信息(名称, 端口, IP地址)

`addrlen`: 可以设置为`sizeof(struct sockaddr)`

返回值: -1表示发生错误, `errno`为错误代码

<br>


```c
int listen(int sockfd, int backlog);
```

`sockfd`: 一个`socket()`返回的套接字描述符

`backlog`: 未经处理的连接请求队列的最大容量

返回值: -1表示发生了错误, `errno`为错误代码

</br>

```c
int accept(int sockfd, void *addr, int *addrlen);
```

`sockfd`: 一个`socket()`返回的套接字描述符

`addr`: 一个`struct sockaddr_in`指针, 存储着远程连接过来的计算机信息(IP, 端口)

`addrlen`: 值应该是`sizeof(struct sockaddr)`, `accept()`不会在`addr`中存储多于`addrlen`的字节, 如果`addr`中的数据不够`addrlen`, `accept()`会改变`addrlen`的值

调用情景:

+ 有人从其他地方尝试用`connect()`连接你的某个端口(已经在`listen()`)
+ `listen()`将连接请求加入`accept()`的等待队列
+ 调用`accept()`连接一个连接请求
+ `accept()`返回一个新的套接字描述符, 表示建立的连接

<br>


```c
int connect(int sockfd, struct sockaddr *serv_addr, int addrlen);
```

`sockfd`: `socket()`返回的套接字描述符

`serv_addr`: 存储着远程计算机的信息(IP, 端口)

`addrlen`: 应该是`sizeof(struct sockaddr)`, 实际上是`socklen_t`类型

<br>


```c
int recv(int sockfd, void *buf, int len, int flags);
```

`sockfd`: 要读取数据的套接字描述符

`buf`: 字符缓冲区指针, 用来放接收到的数据

`len`: 缓冲区长度

`flags`: `recv()`的一个标志, 一般为0

返回值: 实际接收到的数据长度, -1表示发生了错误, `errno`为错误代码

<br>


```c
int send(int sockfd, const void *msg, int len, int flags);
```

`sockfd`: 远程连接的套接字描述符

`msg`: 字节数据指针

`len`: 数据长度

`flags`: 发送标记, 一般为0

返回值: 实际发送的数据长度, -1表示发生了错误, `errno`为错误代码

`send()`有可能发送不完全部数据, 也就是说, 如果返回值小于`len`就需要再次发送剩下的数据

<br>


```c
key_t ftok(const char *path, int id);
```

`path`: 路径, 只要存在就行

`id`: 虽然是`int`, 但是只有8位, 随意设置

返回值: `key_t`的值, -1表示发生了错误

`ftok`通过路径提取文件信息, 再根据这些信息和`id`合成`key_t`

<br>


```c
int msgget(key_t key, int msgflg);
```

`key`: 

+ 0(`IPC_PRIVATE`)表示建立新的消息队列
+ 大于0, 根据`msgflg`来确定操作, 一般来源于`ftok()`返回的IPC键值

`msgflg`:

+ 0表示取消消息队列标识符, 如果不存在则报错
+ `IPC_CREAT`: 当`msgflg & IPC_CREAT`为真时, 如果不存在键值与`key`相等的消息队列, 就新建一个, 如果存在这样的消息队列, 返回其标识符
+ `IPC_CREAT | IPC_EXCL`: 如果不存在键值与`key`相等的消息队列就新建一个, 如果存在就报错

返回值: 一个用`key`命名的消息队列的标识符, -1表示发生错误

<br>


```c
int msgsnd(int msqid, const void *msg, size_t msgsz, int msgflg);
```

`msqid`: `msgget()`返回的消息队列标识符, 表示目标消息队列

`msg`: 要发送的消息

`msgsz`: 消息的长度

`msgflg`:

+ 0表示当目标消息队列已满的时候, 阻塞当前消息的发送, 直到消息能进入目标消息队列
+ `IPC_NOWAIT`: 当消息队列已满的时候, 不等待, 立即返回
+ `IPC_NOERROR`: 若要发送的消息过大, 则将消息截断并把多余部分丢弃, 且不通知发送进程

返回值: 0表示成功, -1表示发生了错误

