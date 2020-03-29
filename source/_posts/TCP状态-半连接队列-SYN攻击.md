---
title: 浅谈TCP状态 半连接队列 SYN攻击
date: 2020-03-27 10:00:53
tags: 
- TCP 
- 三次握手 
- 半连接队列 
- SYN攻击 
categories: 编程
---

## 简介

TCP建立连接，发送数据，断开连接。

<!-- more -->

<img src="https://raw.githubusercontent.com/licovery/licovery.github.io/pic/img/image-20200327100435256.png" style="zoom:70%;" />


### TCP建立连接是三次握手

1. 首先客户端发送`SYN`分节，里面含有随机序列号seq，进入`SYN_SEND`状态。
2. 服务器收到`SYN`分节，进入`SYN_RECV`状态，并且回复`SYN+ACK`分节，里面含有随机序列号seq。
3. 客户端收到`SYN+ACK`分节，回复服务器`ACK`分节，进入`ESTABLISHED`状态。（此时客户端已经建立好连接。可以发送数据。但服务器可能还没有收到ACK，ACK从客户端发送与服务器收到有一个时间差，极端的情况如果此时客户端发送数据，由于网络原因比ACK更快到达服务器，并不会引起RST，服务器忽略该数据，客户端会重传。最后有两种结果，一服务器收到ACK了，连接建立完成，收到重传的数据，二服务器没有收到ACK，TCP连接建立失败）
4. 服务器收到`ACK`分节，也进入`ESTABLISHED`状态，服务器的连接建立完成，至此，整个TCP连接建立完成。

### TCP断开连接是四次挥手

为什么挥手断开连接需要四次？

- 因为TCP是全双工的，主动方断开连接，不再发送数据，不能确保不再接收数据。主动方必须等被动方把要发的数据发完。


为什么最后的ACK要等两个MSL？（为什么需要TIME_WAIT？）

- 2MSL确保有足够的时间让被动方收到了ACK或主动方收到了被动方发超时重传的FIN。即，如果被动方没有收到ACK，就会触发被动方重传FIN。

- TIME_WAIT状态的连接收到重传的FIN后，重传ACK，再等待2MSL时间。确保有足够的时间让“迷途的重复分组”过期丢弃。这只需要1个MSL即可，超过MSL的分组将被丢弃，否则很容易同新连接的数据混在一起。

## TCP状态转移图

关于TCP状态转移，网络上和书籍上（Unix网络编程卷1）已经有很多解析，不再赘述。

![](https://raw.githubusercontent.com/licovery/licovery.github.io/pic/img/image-20200327100606724.png)

另外一种形式

![](https://raw.githubusercontent.com/licovery/licovery.github.io/pic/img/image-20200327100629998.png)

## 半连接

半连接：在三次握手过程中，服务器发送`SYN+ACK`分节之后，收到客户端的ACK之前的TCP连接称为`半连接`。此时服务器处于`SYN_RECV`状态。当收到`ACK`后，服务器转`ESTABLISHED`状态，TCP连接变为`全连接`。

![](https://raw.githubusercontent.com/licovery/licovery.github.io/pic/img/tcp-sync-queue-and-accept-queue-small-1024x747.jpg)

如上图所示，这里有两个队列：`syns queue(半连接队列）`；`accept queue（全连接队列）`

### 全连接队列

已经完成三次握手的连接，调用`accept()`返回一个可用连接。如果全连接队列为空，阻塞IO的情况下下，调用`accept()`将阻塞，非阻塞阻塞IO情况下，调用`accept()`将返回失败。

### 半连接队列

在TCP建立连接过程中，服务器维护一个半连接队列，存放半连接。该队列为每个客户端的`SYN`分节开设一个条目，该条目表明服务器已收到`SYN`分节，并向客户发出`SYN+ACK`分节，正在等待客户的`ACK`分节。这些条目所标识的连接在服务器处于`SYN_RECV`状态，当服务器收到客户的`ACK`分节时，删除该条目，服务器进入`ESTABLISHED`状态，半连接转化为全连接。

### 半连接存活时间

是指半连接队列的条目存活的最长时间，也即服务器从收到`SYN`包到这个报文失效、连接失效的最长时间，该时间值是所有重传请求包的最长等待时间总和。有时我们也称半连接存活时间为`SYN_RECV`存活时间。

### SYN+ACK 重传

服务器发送完`SYN+ACK`包，服务器处于`SYN_RECV`状态，如果未收到客户`ACK`，服务器进行首次重传，等待一段时间仍未收到客户ACK，进行第二次重传，如果重传次数超过系统规定的最大重传次数，系统将该连接信息从半连接队列中删除。注意，每次重传等待的时间不一定相同。　

一种Linux下的实现：Linux下，默认重试5次，加上第一次最多共发送6次；重试间隔从1s开始翻倍增长（一种指数回退策略，Exponential Backoff），5次的重试时间分别为1s, 2s, 4s, 8s, 16s，第5次发出后还要等待32s才能判断第5次也超时。所以，至多共发送6次，经过1s + 2s + 4s+ 8s+ 16s + 32s = 2^6 -1 = 63s，TCP才会认为SYN超时断开这个连接。



## SYN Flood攻击

可以针对半连接队列发起`SYN Flood`攻击——给服务器发一个`SYN`就立即下线，或者填写错误的源端IP地址，在服务器中创建一个半连接。发`SYN`的速度是很快的，这样攻击者很容易将服务器的SYN队列资源耗尽，使服务器无法处理正常的新连接。

### 防御方法

针对该问题，Linux提供了一个`tcp_syncookies`参数解决这个问题。当`SYN队列`满了后，如果收到客户端的`SYN`分节的建立连接请求，服务器不再把连接放入`SYN队列`，服务器会通过源地址、端口、目标地址、端口和时间戳等构造一个特别的`Sequence Number+ACK`发回去，称为`SYN Cookie`。如果是攻击者则不会有响应，如果是正常连接，服务器则会收到客户端的`ACK`响应，然后可以通过确认号中减去 1 以便还原向客户端发送的原始 `SYN Cookie`。至于`SYN队列`中的连接，则不做处理直至超时关闭。请注意，不要用`tcp_syncookies`参数来处理正常的大负载连接情况，因为SYN Cookie本质上也破坏了建连接的SYN超时机制，是妥协版的TCP协议。

对于正常的连接请求，有另外三个参数可供选择：

`tcp_synack_retries`参数设置SYN超时重试次数

`tcp_max_syn_backlog`参数设置最大SYN连接数（SYN队列容量）

`tcp_abort_on_overflow`参数使SYN请求处理不过来的时候拒绝连接

另外一种防护的思路是使用防火墙限制某些IP

## 参考

[浅谈TCP（1）：状态机与重传机制](https://monkeysayhi.github.io/2018/03/07/%E6%B5%85%E8%B0%88TCP%EF%BC%881%EF%BC%89%EF%BC%9A%E7%8A%B6%E6%80%81%E6%9C%BA%E4%B8%8E%E9%87%8D%E4%BC%A0%E6%9C%BA%E5%88%B6/)

[深入探索 Linux listen() 函数 backlog 的含义](https://blog.csdn.net/yangbodong22011/article/details/60399728 ) 

[TCP半连接与SYN攻击](https://blog.csdn.net/u011080472/article/details/51209130)  

[关于TCP 半连接队列和全连接队列](http://jm.taobao.org/2017/05/25/525-1/) 

[维基百科 SYN flood](https://zh.wikipedia.org/zh-cn/SYN_flood) 

[维基百科 SYN cookie](https://zh.wikipedia.org/wiki/SYN_cookie) 