---
title: libevent入门
date: 2020-01-07 22:33:56
tags: 
- 网络编程 
- IO模型 
- 高性能服务器
categories: 编程
---

## libevent简介

[libevent](https://libevent.org)是一个开源的高性能I/O框架，是基于Reactor模式实现的。什么是Reactior模式，简单地说，Reactor一般包括以下几个组件，`句柄（Handle）`，`事件多路分发器（EventDemultiplexer）`，`事件处理器（EventHandler)`。

<!-- more -->

用一个简单的unix socket编程的例子来说明：

- 句柄就是文件描述符fd，可以对其进行一些读写的操作。
- 事件就是某个文件描述符，可读，可写，或者出现错误等事件。
- 事件多路分发器，就是I/O多路复用（select, poll, epoll），再加上一个分发的逻辑。
- 事件处理器可以理解成一个回调函数，这个回调函数决定了出现I/O的时候应该做什么，也就是具体的业务。把事件处理器注册到事件多路分发器中，这样就会形成某个事件和某个事件处理器的之间的一个映射。

libevent中事件分为三种：

- I/O事件
- 信号事件
- 定时器事件

libevent的核心是事件循环，根据发生的事件，调用对应的事件处理器。

## libevent安装

libevent是跨平台的，本人使用的是linux，只介绍linux下安装的方法

1. 使用包管理工具apt或者yum等来下载，下载的版本由软件仓库提供的版本决定
2. 源码编译，[github源码](https://github.com/libevent/libevent)，按readme操作即可

头文件包含：libevent所使用的头文件均在event2目录下

`#include <event2/event.h>`
`#include <event2/xxx.h>`

编译时需要加上 `-levent` 参数链接libevent库

## libevent API

详细请参数官方[API文档](https://libevent.org/doc/files.html)，这里简单介绍几个常用Data Structures和API

| 数据结构            | 功能                                                         |
| ------------------- | ------------------------------------------------------------ |
| `struct event`      | 事件处理器，可以代表IO事件，信号事件，或者定时器事件         |
| `struct event_base` | Reactor实例，也可以理解为事件多路分发器，每个`struct event`都要加入到`struct event_base`才能生效 |



| 函数                                                         | 功能                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `event_base_new()`                                           | 创建event_base                                               |
| `event_base_free()`                                          | 销毁event_base，释放内存                                     |
| `event_new()`                                                | 创建一个事件，可以是IO、信号、定时器事件                     |
| `event_free()`                                               | 销毁一个事件，释放内存                                       |
| `event_add()`                                                | 把一个event加入到event_base的事件循环中                      |
| `event_del()`                                                | 把event从event_base的事件循环中移除                          |
| `event_base_dispatch()` or `event_base_loop()`               | 进入事件循环，一直运行直到没有待定的事件                     |
| `event_base_loopbreak()` or `event_base_loopexit()`          | 退出事件循环                                                 |
| `typedef void(*event_callback_fn)(evutil_socket_t, short, void *)` | 指明回调函数的类型，第一个参数是fd，第二个参数是event类型，第三个参数是在注册的时候填入 |



| 事件类型   |            |
| ---------- | ---------- |
| EV_TIMEOUT | 定时器超时 |
| EV_READ    | fd可读     |
| EV_WRITE   | fd可写     |
| EV_SIGNAL  | 信号产生   |
| EV_CLOSE   | fd关闭     |

## 基于event的Demo

实现一个简单echo服务器，并且捕获SIGINT信号，设定一个1秒的定时器。同时处理IO、信号、定时器三种事件

简单起见，不考虑错误处理

```c
#include <event2/event.h>
#include <event2/listener.h>
#include <signal.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <time.h>
#include <unistd.h>

#define MAX_BUF_SIZE 4096
#define MAX_LISTEN_CON 16

void ReadCallBack(int fd, short event, void *arg)
{
    struct event *readEvent = (struct event *)arg;
    if (event & EV_CLOSED)
    {
        printf("clientFd:%d close\n", fd);
        event_del(readEvent); 
        event_free(readEvent);
        close(fd);
    }
    else if (event & EV_READ)
    {
        char buf[MAX_BUF_SIZE];
        int n = recv(fd, buf, MAX_BUF_SIZE, 0);
        printf("recv %d bytes from clientFd:%d\n", n, fd);
        write(fd, buf, n);
    }
}

void AcceptCallBack(int fd, short event, void *arg)
{
    struct event_base *base = (struct event_base *)arg;
    int connFd = accept(fd, NULL, NULL);
    // 与客户端连接的fd，加入事件循环
    struct event *readEvent = event_new(base, connFd, EV_READ|EV_CLOSED|EV_PERSIST, ReadCallBack, event_self_cbarg());
    event_add(readEvent, NULL);
}

// 信号SIGINT回调函数
void SigIntCallback(int signo, short event, void *arg)
{
    struct event_base *base = (struct event_base *)arg;
    printf("server exit because SIGINT\n");
    event_base_loopexit(base, NULL); // 退出事件循环
}

struct timeval tv = {1, 0};
void TimeoutCallBack(int fd, short event, void *arg)
{
    struct event *timerEvent = (struct event *)arg;
    printf("%ld\n", time(NULL));
    event_add(timerEvent, &tv); // 再次把定时事件加入事件循环
}


int main()
{
    // 创建监听fd
    int listenFd = socket(AF_INET, SOCK_STREAM, 0);

    // 设置socket选项
    int flag = 1;
    setsockopt(listenFd, SOL_SOCKET, SO_REUSEADDR, &flag, sizeof(flag));
    
    // 填入地址端口信息
    struct sockaddr_in serverAddr = {0};
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = htonl(INADDR_ANY);
    serverAddr.sin_port = htons(8888);
    
    // fd绑定地址端口信息
    bind(listenFd, (struct sockaddr *) &serverAddr, sizeof(serverAddr));

    // 监听fd
    listen(listenFd, MAX_LISTEN_CON);

    // 创建Reactor实例，事件循环实例
    struct event_base *base = event_base_new();

    // 创建并注册IO事件处理器           
    struct event *listenEvent = event_new(base, listenFd, EV_READ|EV_PERSIST, AcceptCallBack, base);
    event_add(listenEvent, NULL);

    // 创建并注册信号事件处理器
    struct event *signalEvent = evsignal_new(base, SIGINT, SigIntCallback, base);
    event_add(signalEvent, NULL);

    // 创建并注册定时器事件处理器
    struct event *timerEvent = evtimer_new(base, TimeoutCallBack, event_self_cbarg());
    // event_add(timerEvent, &tv);

    // 事件分发循环
    event_base_dispatch(base);

    // 释放
    event_free(listenEvent);
    event_free(signalEvent);
    event_free(timerEvent);
    event_base_free(base);
}
```

## 基于bufferevent的Demo

bufferevent是对I/O事件的一种更高层的抽象。把一个bufferevent和一个句柄绑定起来，并且设置这个bufferevent的属性EV_READ|EV_WRITE，那么就可以得到一个可读写的bufferevent，并设置响应的回调函数，处理buffer时的业务，处理buffer可写时候的业务。bufferevent有个特点就是，用户不需要关系真正的读写的操作，buffervevent会在读写缓冲区可用的时候自动操作。用户只需要关心把输入缓冲区的数据取出并且经过业务处理，然后把数据放入输出缓冲区即可，底层的I/O会自动完成，实现真正的异步操作。

对于echo服务器，核心代码就一行`evbuffer_add_buffer(ouput, input)`把输入缓冲区的数据直接迁移到输出缓冲区，libevent使用`struct evbuffer`这种数据结构，还可以避免过多数据的拷贝。

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <event2/event.h>
#include <event2/bufferevent.h>
#include <event2/buffer.h>

#define MAX_BUF_SIZE 4096
#define MAX_LISTEN_CON 16

void ReadCallback(struct bufferevent *bev, void *ctx)
{
    struct evbuffer *input = bufferevent_get_input(bev);
    struct evbuffer *ouput = bufferevent_get_output(bev);
    evbuffer_add_buffer(ouput, input); // 把输入缓冲区的数据完全复制到输出缓冲区
}

void ErrorCallback(struct bufferevent *bev, short event, void *ctx)
{
    if (event & BEV_EVENT_EOF) //对端关闭
    {
        printf("clinet close\n");
    }
    else if (event & BEV_EVENT_ERROR)
    {
        printf("network error\n");
    }
    bufferevent_free(bev);
}

void AcceptCallback(int fd, short event, void *arg)
{
    struct event_base *base = (struct event_base *)arg;

    int connFd = accept(fd, NULL, NULL);

    struct bufferevent *bufev = bufferevent_socket_new(base, connFd, BEV_OPT_CLOSE_ON_FREE); //bufev free的时候自动close fd
    bufferevent_setwatermark(bufev, EV_READ, 0, MAX_BUF_SIZE); //一次读取最大不超过MAX_BUF_SIZE
    bufferevent_enable(bufev, EV_READ|EV_WRITE);
    bufferevent_setcb(bufev, ReadCallback, NULL, ErrorCallback, NULL);
}


int main()
{
    // 创建监听fd
    int listenFd = socket(AF_INET, SOCK_STREAM, 0);

    // 设置socket选项
    int flag = 1;
    setsockopt(listenFd, SOL_SOCKET, SO_REUSEADDR, &flag, sizeof(flag));
    
    // 填入地址端口信息
    struct sockaddr_in serverAddr = {0};
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = htonl(INADDR_ANY);
    serverAddr.sin_port = htons(8888);
    
    // fd绑定地址端口信息
    bind(listenFd, (struct sockaddr *) &serverAddr, sizeof(serverAddr));

    // 监听fd
    listen(listenFd, MAX_LISTEN_CON);

    // 创建Reactor实例
    struct event_base *base = event_base_new();

    // 创建并注册IO事件处理器           
    struct event *listenEvent = event_new(base, listenFd, EV_READ|EV_PERSIST, AcceptCallback, base);
    event_add(listenEvent, NULL);

    event_base_dispatch(base);

    event_free(listenEvent);
    event_base_free(base);
}
```



## 参考

官方文档是英文，但是阅读难度不高，建议阅读

http://www.wangafu.net/~nickm/libevent-book/TOC.html

https://libevent.org/doc/