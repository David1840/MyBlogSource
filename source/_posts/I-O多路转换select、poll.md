---
title: I/O多路转换select、poll、epoll
date: 2018-07-21 09:47:22
categories: 
- Linux系统
tags:
- Linux
- 系统调用
---


构造一张我们感兴趣的描述符的列表，然后调用一个函数，直到这些描述符中的一个已经准备好进行I/O操作时，该函数才返回。

在Linux系统中可以使用select、poll、pselect等函数执行I/O多路转换，在从这些函数返回时，进程会被告知哪些描述符已经准备好可以进行I/O。


# select

系统调用select()会一直阻塞，直到一个或者多个文件描述符集合成为就绪态。

```
int select(int nfds, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);
```

select函数的第一个参数nfds取值为最大描述符加1，意思是在后面三个读、写、异常描述符集参数中找到最大描述符，然后加1。

readfds、writefds、exceptfds是指向描述符集的指针，这三个描述符集说明了我们关心的可读、可写或处于异常条件的各个描述符，每个描述符集存放在一个fd_set数据结构中，为每一可能的描述符保持了一位。

传向select的参数告诉内核：我们所关心的描述符；对于每个描述符我们所关心的状态，读、写和异常；愿意等待多长时间，不等待、等待若干时间或无限等待。

从select返回时，内核告诉我们：已准备好的描述符的数量；对于读、写或异常这三个状态中的一个，哪些描述符已准备好。使用这些返回信息，就可调用相应的IO函数，如read/write，并且确知该函数不会阻塞。select出错返回-1，并设置对应的errno，timeout超时返回0。


# poll

poll()和select()很相似。两者的主要区别在于我们要如何指定待检查的文件描述符。

```
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

struct pollfd {
      int   fd;         /* file descriptor */
      short events;     /* requested events */
      short revents;    /* returned events */
};
```

poll关心的描述符通过pollfd数组设置，pollfd个数为nfds，其中fd为待处理的描述符，events为我们想要描述符处理的事件，包括读、写事件，如POLLIN表示是否有数据可读，由用户通过按位或操作符指定，revents是poll函数的执行结果，告诉我们响应了哪些事件，可通过按位与操作符检查。timeout单位为秒。


# select、poll的缺点
poll和select同样存在一个缺点就是，包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。

# epoll

epoll是对select和poll的改进。相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。epoll机制是Linux最高效的I/O复用机制，在一处等待多个文件句柄的I/O事件。

epoll提供了三个函数，epoll_create,epoll_ctl和epoll_wait。

```
#include <sys/epoll.h>
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

### (1) int epoll_create(int size)

创建一个epoll的句柄(实例)，size指定了我们想要通过epoll实例来检查的文件描述符个数。该参数不是一个上限，而是告诉内核应该如何为内部数据结构划分厨师大小。

需要注意的是，当创建好epoll句柄后，它就是会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。

### (2) int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)

epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。

第一个参数是epoll_create()的返回值

第二个参数表示动作，用三个宏来表示：

	EPOLL_CTL_ADD：注册新的fd到epfd中；
	EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
	EPOLL_CTL_DEL：从epfd中删除一个fd；

第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事，struct epoll_event结构如下：

```
struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};
```

events可以是以下几个宏的集合：

	EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
	EPOLLOUT：表示对应的文件描述符可以写；
	EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
	EPOLLERR：表示对应的文件描述符发生错误；
	EPOLLHUP：表示对应的文件描述符被挂断；
	EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
	EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
	

### (3) int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout)

等待事件的产生，类似于select()调用。参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时。


### 工作模式

epoll对文件描述符的操作有两种模式：LT（level trigger）和ET（edge trigger）。LT模式是默认模式，LT模式与ET模式的区别如下：

　　LT模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件。

　　ET模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。

　　ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。