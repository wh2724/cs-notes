## I/O多路复用之select、poll和epoll

I/O 模型一个输入操作大概就是分两步：应用程序请求并等待数据到达；从内核向进程复制数据。而对于套接字上的输入操作，第一步类似，就是等待数据从网络中到达，**数据到达以后为了防止数据丢失，会先复制到内核中的某个缓冲区；第二步就是将内核缓冲区中的数据复制到应用程序缓冲区。**

Unix 下有五种 I/O 模型：

- 阻塞式 I/O

- 非阻塞式 I/O

- I/O 复用(select poll)

- 信号驱动式 I/O(SIGIO)

- 异步 I/O(AIO)

**描述符**：内核（kernel）利用文件描述符（file descriptor）来访问文件。 文件描述符是非负整数。 打开现存文件或新建文件时，内核会返回一个文件描述符。 读写文件也需要使用文件描述符来指定待读写的文件。描述符在Windows系统中又称句柄。

select，poll，epoll都是IO多路复用的机制。I/O多路复用就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。

### select

select 可以用来监视多个文件句柄的状态变化，程序会阻塞在 select 直到被监视的FD有一个或多个发生了状态改变，然后通知应用程序，应用程序轮询所有的FD集合，判断监控的FD是否有事件发生，并作相应的处理。

```c
int select(int maxfd, fd_set * readfds, fd_set * writefds, fd_set * exceptfds, struct timeval * timeout);

FD_CLR(inr fd, fd_set* set); // 清除描述词组set中相关fd的位

FD_ISSET(int fd, fd_set *set); // 用来测试描述词组set中相关fd的位是否为真

FD_SET(int fd, fd_set*set); // 用来设置描述词组set中相关fd的位

FD_ZERO(fd_set *set); // 用来清除描述词组set的全部位

struct timeval {

time_t tv_sec;

time_t tv_usec;

};

// ----- 调用流程 -------

fd_set readfso; // 1.1 定义需要监视的描述符

FD_ZERO(&readfso); // 1.2 清空

FD_SET(fd, &readfso); // 1.3 设置需要监控的描述符，如fd=1

while(1) {

readfs = readfso;

ret = select(maxfd+1, &readfds, NULL, NULL, timeout); // 2.1 监听所关注的事件，在此为监听读事件

if (ret > 0) // 3.1.1 监听事件发生

for (i = 0; i > FD_SETSIZE; i++) // 3.1.2 需要遍历所有fd

if (FD_ISSET(fd, &readfds))

handleEvent();

else if (ret == 0) // 3.2 超时

handle timeout

else if (ret < 0) // 3.3 发生错误

handle error

}
```

在 select 函数中，各个参数的含义如下。

- maxfd 为最大文件描述符+1，因此在程序中需要知道最大描述符的值。

- 接着的三个参数分别设置所监听的读、写、异常事件对应的描述符。该值传入需要监控事件的描述符，返回值保存了发生了的事件对应的描述符。

- 设置超时时间，NULL 表示一直等待。

返回值：成功则返回文件描述符状态已改变的总数；如果返回0代表在描述词状态改变前已超过 timeout 时间；当有错误发生时则返回 -1，错误原因存于 error ，此时参数 readfds, writefds, exceptfds 和 timeout 的值变成不可预测。

在有返回值时，需要通过 FD_ISSET 循环遍历各个描述符，从而文件描述符越多，效率也就越低。而且，每次调用 select 时都需要重新设置需要监控的文件描述符。

fd_set 在 sys/select.h 中定义，大致可以简化为如下的结构，也就是实际用数组表示，其中每一位代表一个文件描述符，最大可以表示 1024 个，也就是说 select 能监视的描述符最大为 1024 。

用户线程会启动一个监视Socket，调用select函数以后会阻塞，直到有描述符就绪(可读、可写和异常)，或超时，函数才返回。当函数返回以后，再遍历fdset来寻找就绪的描述符。select本质上是通过设置或者检查存放fd标志位的数据结构来进行下一步处理。

select有以下缺点：

1. 每次调用都需要把fd集合从用户态拷贝到内核态，这样会使得用户空间和内核空间在传递该结构时复制开销大；

2. 每次扫描时采取线性扫描内核中的所有fd，效率很低；

3. 单个线程处理的IO数量由fd限制，默认值为1024(x32)。

然而，它有良好的跨平台性，几乎所有平台都可以支持。

### poll

和 select() 不一样，poll() 没有使用低效的三个基于位的文件描述符 set ，而是采用了一个单独的结构体 pollfd 数组，由 fds 指针指向这个数组，**采用链表保存文件描述符**。

```c
struct pollfd {

int fd; /* 文件描述符 */

short events; /* 等待的事件 */

short revents; /* 实际发生了的事件 */

} ;

typedef unsigned long nfds_t;

int poll(struct pollfd *fds, nfds_t nfds, int timeout);

struct pollfd fds[MAX_CONNECTION] = {0};

fds[0].fd = 0;

fds[0].events = POLLIN;

ret = poll(fds, sizeof(fds)/sizeof(struct pollfd), -1)

if (ret > 0)

for (int i = 0; i < MAX_CONNECTION; i++)

if (fds[0].revents & POLLIN)

handleEvent(allConnection[i]);

else if (ret == 0)

handle timeout

else if (ret > 0)

handle error
```

poll和select的实现是类似的，区别在于它是**基于链表存储的，没有最大连接的限制**。

### epoll

epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。

epoll提供了三个函数：

- epoll_create 创建一个epoll句柄

- epoll_ctl 向内核注册新的描述符或者是改变某个文件描述符的状态。已注册的描述符在内核中会被维护在一棵红黑树上，通过回调函数内核会将 I/O 准备好的描述符加入到一个链表中管理

- epoll_wait则是等待事件的产生

epoll使用`事件`的就绪通知方式，通过epoll_ctl注册fd，一旦该fd就绪，内核就会采用类似callback的回调机制来激活该fd，epoll_wait便可以收到通知。

epoll对文件描述符的操作有两种模式：

- LT（level trigger，水平触发）：LT模式是默认模式，当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件；
- ET（edge trigger，边缘触发）：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。

#### epoll如何解决select的三个缺点

对于第一个缺点，epoll的解决方案在epoll_ctl函数中。每次注册新的事件到epoll句柄中时（在epoll_ctl中指定EPOLL_CTL_ADD），会把所有的fd拷贝进内核，而不是在epoll_wait的时候重复拷贝。epoll保证了每个fd在整个过程中只会拷贝一次。

对于第二个缺点，epoll的解决方案不像select或poll一样每次都把current轮流加入fd对应的设备等待队列中，而只在epoll_ctl时把current挂一遍（这一遍必不可少）并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，而这个回调函数会把就绪的fd加入一个就绪链表。epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd。

对于第三个缺点，epoll没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左右，一般来说这个数目和系统内存关系很大。

如果没有大量的空闲连接或者“死”连接，epoll的效率并不会比select/poll高很多，但是当有大量空闲连接的时候，就会发现epoll的效率会大大提高。

|              | select                                                       | poll             | epoll                                                        |
| ------------ | ------------------------------------------------------------ | ---------------- | ------------------------------------------------------------ |
| 最大连接数   | 1024(32位) 2048(64位)                                        | 无最大连接数限制 | 最大连接数由内存决定，1G的内存可以打开约10万个连接           |
| IO效率       | 每次调用都会对连接进行线性遍历，随着FD的增加使得遍历效率降低 | 同select为O(n)   | fd上是利用回调函数，只有活跃的socket才会主动调用callback , O(1) |
| 消息传递方式 | 通过拷贝的方式将内核的消息传递给用户空间                     | 同select         | 通过共享内存的方式，实现内核态和用户态的通信                 |

