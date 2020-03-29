---
title: Linux/Unix 信号处理signal()函数的坑——为什么僵尸进程总是清理不掉？
date: 2019-12-07 00:28:52
tags: Linux服务器编程
categories: 编程
---

## 重点

先说重点，`signal()`不是标准函数，行为在不同地方不一样，不要直接使用`signal()`！！！实在有需求的，自定义一个接口一样的函数来使用。<!-- more -->

> The  behavior  of  signal()  varies across UNIX versions, and has also varied historically across different versions of Linux. ——man signal （Linux man 手册）

## 背景

最近看Unix网络编程卷1：套接字联网API。第五章提到一个例子，服务器端利用`fork()`创建子进程来处理多个客户端的请求。如果子进程完成处理退出的时候，父进程不调用`wait()`或`waitpid()`处理子进程，子进程就会处于**僵死**状态，成为僵尸进程。如果僵死进程过多，很可能会耗尽系统的资源。

Linux/Unix系统保证，在一个进程终止或者停止时，它会发送`SIGCHLD`信号给其父进程。所以我们可以得到一种处理僵尸进程的方法：父进程设置`SIGCHID`的信号处理函数，并在信号处理函数中`wait()`或者`waitpid()`子进程。这就引出了这次讨论的重点信号处理和`signal()`函数。

## 信号与signal()函数

Linux/Unix系统定义了一组信号，用于处理异步事件。信号的例子有很多，终端用户输入了 Ctrl+c 来中断程序(`SIGINT `)，或者使用kill来杀死进程(`SIGKILL`)，又或者子进程结束(`SIGCHLD`)。对于大多数的信号（`SIGKILL`和`SIGSTOP`不能被捕获），可以捕获并设置对应的自定义处理函数。

自定义信号处理函数的原型长这样：

```c
void sig_handler(int signo); // 参数就是信号，信号都用宏表示，是一个大于0的值
```

`signal()`是一个定义在标准库<signal.h>的函数，来用设置对应信号的处理函数，就是这个函数坑了一天，后面再详细说。函数原型是这样：

```c
#include <signal.h>
typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);
```

第一个参数`signum`就是要捕获的信号的值，第二个参数`sighandler_t`是自定义的信号处理函数。返回值和第二个参数类型一样，含义是前一次设置的信号处理函数，如果是第一次设置的话就返回默认处理函数(`SIG_DFL`)。

看起来很简单是不是？就根据信号设置一个回调函数就可以了，真方便。网上很多介绍`signal()`的，都是介绍一下用法，随便写几个样例就完了。但是有很重要的一点忽略了，**signal()这个函数并不是标准函数**，可能不同的机器，不同Linux，Unix内核版本，甚至在不同的`gcc`编译宏或者编译选项下会有不同的表现。

> signal是早于POSIX出现的历史悠久的函数。调用它时，不同的实现提供不同的信号语义环境以达成后向兼容——《Unix网络编程卷1：套接字联网API》 第五章p104

接下来可以好好说说这个`signal()`的行为不确定是怎么影响的处理僵尸进程的。

## 为什么僵尸进程总是清理不掉？

先贴一下代码，是一个多进程的echo回射的服务器，客户端发什么东西过去，原样返回。

```c
#include "comm.h" // 里面包含了很多socket unix相关的头文件，这个也很关键，后面解释

//里面有些大写的函数例如Socket是封装了系统的socket加了错误处理，功能不改变。Read,Write,Bind等等同理

void ServerProc(int acceptFd)
{
    int n;
    char buf[MAX_BUF_SIZE];
    while ((n = Read(acceptFd, buf, MAX_BUF_SIZE)) > 0)
    {
        Writen(acceptFd, buf, n);
    }
    Close(acceptFd);
}

// 处理结束的子进程
void HandleSigchild(int sig)
{
    int pid;
    int stat;
    while ((pid = waitpid(-1, &stat, WNOHANG)) > 0)
    {
        // IO不安全，仅调试
        printf("child process %d terminated\n", pid);
    }
}

int main()
{
    int listenFd = Socket(AF_INET, SOCK_STREAM, 0);
    
    struct sockaddr_in serverAddr = {0};
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_addr.s_addr = htonl(INADDR_ANY);
    serverAddr.sin_port = htons(ECHO_PORT - ECHO_PORT + 9877);

    Bind(listenFd, (struct sockaddr *)&serverAddr, sizeof(serverAddr));
    Listen(listenFd, MAX_LISTEN_CON);

    signal(SIGCHLD, HandleSigchild); //最重要的地方

    while (1)
    {
        // accept可能会被信号处理函数返回的时候打断
        int acceptFd = accept(listenFd, NULL, NULL);
        if (acceptFd < 0)
        {
            if (errno == EINTR)
            {
                continue;
            }
            else
            {
                err_sys("accept error");
            }
            
        }
        int pid = Fork();
        if (pid == 0)
        {
            // 子进程关闭listenFd
            Close(listenFd);
            ServerProc(acceptFd);
            exit(0);// 因为这里exit，子进程的作用范围只到这里，考虑处理僵尸进程
        }
        //父进程关闭acceptFd
        Close(acceptFd);
    }
    return 0;
}
```

编译命令：`gcc -g -std=c11 -Wall -I../comm -L../comm/build echo_server.c -lcomm -o echo_server`  用到了自己的静态库`libcomm.a`

### 存在的问题

当建立了n个子进程的时候，发现只有一个子进程被`waitpid()`处理了，其余的都变成了僵尸进程。似乎`HandleSigchild()`只被回调了一次，此时我考虑是否信号丢失了或者在哪里发生错误，只传递了一次过来。

### 另外一个例子

为了搞清楚`signal()`的用法，我又写了一个样例。

```c
#include <stdio.h>
#include <unistd.h>
#include <signal.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/wait.h>

void wait4children(int signo)
{
    while (waitpid(-1, NULL, WNOHANG) > 0);
}

int main()
{
    int i;
    pid_t pid;

    signal(SIGCHLD, wait4children);

    for (i = 0; i < 100; i++)
    {
        pid = fork();
        if (pid == 0)
        {
            break;
        }
    }

    if (pid > 0)
    {
        printf("press Enter to exit...");
        getchar();
    }

    return 0;
}
```

编译命令：`gcc -g test.c -o test`  

运行发现确实每一个子进程的SIGCHLD信号都捕获到了并且都进行了`waitpid()`处理，没有出现僵尸进程。

### 疑惑

为什么我写了`socket`相关的代码，子进程的`SIGCHID`信号就没法处理了呢？想到Unix编程里面对`signal()`的描述，我开始在google上搜索signal相关资料。发现有不少人说：**signal()设置信号的回调处理函数，是一次性的，当回调函数处理完一次，这个信号的处理函数会被设置回默认(SIG_DFL)，如果要处理多次信号，要重新调用signal()**。仔细一想，也不对啊，那为什么我的`echo_server.c`的`signal()`是一次性的，我的`test.c`的`signal()`就是永久的啊？这个时候，我开始怀疑链接的时候，signal()这个符号链接到了不同的函数。

用`gdb`开始调试，发现`echo_server`的`signal`是`libc.so.6`里面的`sysv_signal`，而`test`的是`libc.so.6`的`bsd_signal`。同时我还用`objdump`搜索了一下两个可执行文件中`signal`的符号，发现确实不一样，证明了这两个`signal()`是两个函数。

命令：`objdump -tT echo_server | grep signal`

似乎有了一点点眉目，就这能解释为什么两份代码里面同一个`signal()`，但是真正执行的代码不一样。一个疑惑解决了，但是为什么`gcc`编译的时候会出现这样的问题？还有这个`sysv_signal`和`bsd_signal`到底是什么？这是时候只能继续google了，功夫不负有心人，终于在Linux编程手册上找到了解答。

## Linux Programmer's Manual 的解答

详细信息点击[signal](http://man7.org/linux/man-pages/man2/signal.2.html)

> The behavior of signal() varies across UNIX versions, and has also varied historically across different versions of Linux.  Avoid its use: use sigaction(2) instead.  See Portability below.

大体意思就是说，`signal() `这东西有历史包袱，会在不同的Unix和Linux上表现不一致。接着往下看

> System V also provides these semantics for signal().  This was bad because the signal might be delivered again before the handler had a chance to reestablish itself.  Furthermore, rapid deliveries of the same signal could result in recursive invocations of the handler.
>
> BSD improved on this situation, but unfortunately also changed the semantics of the existing signal() interface while doing so.  On BSD, when a signal handler is invoked, the signal disposition is not reset, and further instances of the signal are blocked from being delivered while the handler is executing.  Furthermore, certain blocking system calls are automatically restarted if interrupted by a signal handler (see signal(7)).  
>

这里说有两种`signal()`，一种`System V`的，一种`BSD`的，BSD的`signal()`改善了System V的，在BSD，`signal()`**不会reset**信号处理函数，多个信号同时到达的时候会阻塞，阻塞的系统调用被中断的时候会自动重启。这不就正好对应上面提到的`sysv_signal()`（sysv就是System V的缩写）和`bsd_signal()`！这时，echo_server为什么只能处理一个子进程就能解释了，因为它真正调用的是`sysv_signal()`，由System V实现的，**会reset**信号处理函数。同样的道理test真正调用的是`bsd_signal()`，由BSD实现，**不会reset**信号处理函数，所以可以一直处理所有子进程。好的，关于`sysv_signal()`和`bsd_signal()`是什么的问题解决了。点击了解更多关于[sysv_signal()]( http://man7.org/linux/man-pages/man3/sysv_signal.3.html )和[bsd_signal()](http://man7.org/linux/man-pages/man3/bsd_signal.3.html)，里面有更详细的说明。

剩下的问题就是为什么一个`signal()`编译的时候会变成两个不同的函数？其实手册中也有说明，这里我使用的是本机（ubuntu 16.04 LTS)的man signal得到的。

> The situation on Linux is as follows:
>
> The kernel's signal() system call provides System V semantics.
>
> By default, in glibc 2 and later, the signal() wrapper function does not invoke the kernel system call.  Instead, it calls sigaction(2) using flags that supply BSD  semantics. This  default  behavior  is  provided  as  long as the _BSD_SOURCE feature test macro is defined.  By default, _BSD_SOURCE is defined; it  is  also  implicitly  defined  if  one defines _GNU_SOURCE, and can of course be explicitly defined.
>
> On  glibc  2  and later, if the _BSD_SOURCE feature test macro is not defined, then signal() provides System V semantics.  (The default implicit definition of  _BSD_SOURCE  is not  provided  if one invokes gcc(1) in one of its standard modes (-std=xxx or -ansi) or defines various other feature test  macros  such  as  _POSIX_SOURCE,  _XOPEN_SOURCE,  or _SVID_SOURCE; see feature_test_macros(7).)

总结就是：定义了`_BSD_SOURCE `这个宏，`signal()`就会编译成BSD的版本，也就是不reset信号处理函数，阻塞相同信号，自动唤醒中断的系统调用，然后`_BSD_SOURCE `是默认定义的。**但是**如果你使用了gcc  -std=xxx or -ansi **编译选项**，或者使用了`_POSIX_SOURCE`,  `_XOPEN_SOURCE`,  or `_SVID_SOURCE`这些宏，那么`_BSD_SOURCE `这个宏就**不再自动定义**了，`signal()`就变成System V的版本。这就是我前面提到了gcc的编译选项和编译宏会影响signal()的真正的行为的解释。

好了真相大白了，在echo_server中我确实使用了`std=c11`的编译选项，然后我去掉std=c11这个编译选项，再用objdump查看符号，发现还是不对。这个时候我怀疑肯定是socket编程有关的头文件里面的宏影响了。然后我导出了编译时的宏定义，一看果然有问题

命令：`gcc -I../comm -E -dM echo_server.c > echo_server.macro`使用`-E` 和`-dM`参数就可以导出编译宏，发现里面确实含有`_POSIX_SOURCE`。终于一切都水落石出了。

### 说明

简单说一下Linux Programmer's Manual中括号()里的数字是什么含义

1. 一般命令
2. 系统调用
3. 库函数，涵盖C标准函数库
4. 特殊文件（通常是/dev中的设备）和驱动程序
5. 文件格式和约定
6. 游戏和屏保
7. 杂项
8. 系统管理命令和守护进程

例如signal(2)就是系统调用。

## 解决方案

既然知道问题是由于`signal()`行为不确定导致的，那么自己写一个确定的signal函数就完事了啊。用`sigaction()`来代替signal。关于`sigaction()`的用法网上有很多，我就不细说了。

```c
#include <signal.h>
typedef	void	Sigfunc(int);
//Signal首字母大写区分signal
Sigfunc * Signal(int signo, Sigfunc *func)
{
	struct sigaction act, oldact;
	act.sa_handler = func;
	sigemptyset(&act.sa_mask);
	act.sa_flags = 0;
	act.sa_flags |= SA_RESTART;		//重启因为信号处理被中断的慢系统调用，例如accept,read
	if (sigaction(signo, &act, &oldact) < 0)
	{
		return SIG_ERR;
	}
	return oldact.sa_handler;
}
```

## 写在最后

本来就是想照着书本实现一下简单的子进程回收，防止出现僵尸进程，没有想到最后搞出那么多花样。花了我整整一天的时间，除了吃饭就在干这个。Linxu系统编程真的不是一件简单的事情，一个signal()居然牵扯到那么多编译相关的问题，还遇上了signal()这种历史遗留问题，可是网上关于signal()的资料都是讲用法，很少有提到signal()这个函数应该避免使用。其实很多东西都已经写在文档里面，可是大多数人都没有耐心去看文档，而且也是全英文，不友好。我自己也不看文档，有问题都是直接google。但是这种**最容易得到的知识，往往价值是相对比较低的，而且可能是错误的。真正有价值的东西，需要花时间去深挖，去积累。**要养成看文档的好习惯。

虽然花了一天时间，但还是很开心，因为问题解决了，困惑消除了，很有成就感。学习编程那么久，也碰到过不少"神奇"的问题，多半都没有解决，这次算是一个里程碑吧。以后遇到问题，还有继续刨根问底！

最后一点感悟就是，C/C++的工程里，随便改一个构建系统的一个参数或者一个编译宏，都可能产生巨大的影响。不熟悉的东西，最好不要乱试。

## 参考

 http://zheming.wang/blog/2011/02/17/65C9E90A-8D99-46DA-AC8D-F4D4EC825CD8/ 

 http://man7.org/linux/man-pages/man2/signal.2.html 

 https://blog.csdn.net/Move_now/article/details/60591018 

 https://www.infoq.cn/article/linux-signal-system 