[toc]

参考[详解磁盘IO、网络IO、零拷贝IO、BIO、NIO、AIO、IO多路复用(select、poll、epoll) - 予真 - 博客园](https://www.cnblogs.com/JavaYuYin/p/18006057)

# 0. 序言

以前就听说过**网络io的存在**, 但不知道为什么会有这样的知识点存在. 它究竟解决了什么问题. 这里我想用简单的语言描述网络io模式, 可能会有错.

先说结论吧, 网络io的存在是为了让**网络服务器更多的占用cpu的资源, 为了优化性能**

# 1. 磁盘IO原理简记

在磁盘io中, 用户程序想要访问一个文件, 按照一般设计套路, 会先访问速度快容量小的缓存, 如果命中则返回. 否则, 会切换到**内核态**, 让操作系统执行系统调用, 将文件从磁盘复制到**内核空间(系统内存空间)**. 随后, 再回到**用户态**, 用户程序就可以读取内核空间(只读)的文件内容, 并复制到用户空间(可读写).

之所以有内核态和用户态的区分, 是为了**保证安全性, 防止用户访问和破坏系统区域**

计算机的发展历程之一可以使用**为了榨干cpu**来形容

至于**io设备数据传输的较具体流程, 还取决于不同的架构设计**

因为确实存在一些不可解决的实质性问题, 比如

* 主存无法并行连接多个设备
* 不同设备的处理速度不同(因此出现了cache)

这里仅介绍一个简单的DMA架构方式吧, 类似南桥(通道)

(1)应用程序发起文件读取->

(2)CPU通知DMA, 要读取数据(如果磁盘不忙), 之后CPU做别的事情->

(3)磁盘开始准备数据(**将数据放入磁盘的缓存**), 缓存满了或数据准备完成, 发送请求给DMA->

(4)DMA利用**主存周期窃取**, 将数据保存到内核空间->

(5)DMA发送中断请求给cpu, cpu响应中断->

(6)cpu将数据从内核空间存储到用户空间.

更详细的过程参考我自己的**计组笔记吧**

# 2. 网络IO

网络io和磁盘io的**最大区别是, 磁盘io有数据就是有数据, 没数据就是没数据; 网络io则是socket里没数据, 不表明一直没数据, 只是客户端此时没有发送数据, 所以用户程序需要持续监测是否有后续数据到达**

## 2.1 零IO

一般读取文件并发送到网络的流程是

磁盘->内核空间句柄->用户空间->socket缓存池->网卡

**零IO直接在内核空间发送数据**

磁盘->内核空间句柄->socket缓存池->网卡

linux中的sendfile就实现了这个功能.

## 2.2 BIO, 阻塞IO

当一个线程进入内核态并等待socket的数据时, 它会阻塞在那里, 直到数据到来或者它的执行时间完毕(时间片轮转)并让出cpu.

## 2.3 NIO, 同步非阻塞IO

当一个线程进入内核态并查询socket的数据时, 无论数据是否存在, 都立刻返回. 线程只有在内核空间复制数据到用户空间的过程中会阻塞, 占用cpu.

这种情况下, 一个线程可以处理多个请求, 比如让一个线程遍历多个请求, 如果数据没准备好. 就遍历下一个请求.

<span style="color: red">NIO的问题是会频繁的在用户态和内核态中切换, 会浪费不少的性能</span>

切换涉及断点程序的保存, 中断服务程序的载入.

## 2.4 IO多路复用

NIO的问题是, 一个线程管理多个socket的请求时, 每访问一个socket, 都在内核态和用户态间切换.

IO多路复用的思想是, **用一次状态切换查询多个socket的数据状态**:

它是**阻塞式的**, 如果没有一个socket有数据, 这种io仍然会进入阻塞状态.

根据返回的就绪socket的文件句柄集合, 单一线程在后续就能处理所有就绪的数据.



# 3. 多路复用的三种方式

## 3.1 select方式

```c++
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

select的机制如下:

每次调用select时, 决定哪些socket句柄是要读的, 哪些是要写入的. 并把它们交给内核(复制到内核)判断, 内核根据这些句柄, 来确定应该查询谁. 查询后文件句柄数组会被清空.

之所以每次**都要向内核传入句柄, 是为了复位**, 因为socket的状态位在内核中, 而用户决定哪些socket的数据被处理了(有可能用户程序读取了数据, 但不处理, socket缓存不应该丢弃?)

select对于每次的文件句柄数有限制, 为1024.

## 3.2 poll方式

```c++
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```

其中pollfd是结构体

```c++
struct pollfd {
    int fd; /* file descriptor */  
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};
```

参数说明

* fds: 用于存放需要检测其状态的socket文件描述符，并且调用poll函数之后fds数组不会被清空
* nfds: 是监控的socket文件句柄数量
* timeout: 是等待的毫秒数，这段时间内无论IO是否准备好，poll都会返回。timeout为负数表示无线等待，timeout为0表示调用后立即返回

* return返回值: poll函数执行结果：为0表示超时前没有任何事件发生，-1表示失败，成功则返回结构体pollfd的revents不为0的文件描述符个数。

events域可能取值, 它表示监视该文件描述符的事件,由用户来设置这个域

* POLLIN：有数据可读。
* POLLRDNORM：有普通数据可读。
* POLLRDBAND：有优先数据可读。
* POLLPRI：有紧迫数据可读。
* POLLOUT：写数据不会导致阻塞。
* POLLWRNORM：写普通数据不会导致阻塞。
* POLLWRBAND：写优先数据不会导致阻塞。
* POLLMSG、SIGPOLL：消息可用。

revents域是文件描述符的操作结果事件, 由内核在调用返回时设置这个域. events域中请求的任何事件都可能在revents域中返回

* POLLER：指定的文件描述符发生错误。
* POLLHUP：指定的文件描述符挂起事件。
* POLLNVAL：指定的文件描述符非法。

举一个例子，events=POLLIN | POLLPRI等价于select()的读事件，events=POLLOUT | POLLWRBAND等价于select()的写事件。此外，POLLIN等价于POLLRDNORM |POLLRDBAND，而POLLOUT则等价于POLLWRNORM。

当我们要同时监视一个文件描述符是否可读和可写，可以设置 events=POLLIN |POLLOUT（由用户设置）。在poll返回时，可以检查revents中的标志（由内核在调用返回时设置）。如果revents=POLLIN，表示Socket文件描述符可以被读取而不阻塞。如果revents=POLLOUT，表示Socket文件描述符可以写入而不导致阻塞。注意，POLLIN和POLLOUT并不是互斥的：它们可能被同时设置，表示这个文件描述符的读取和写入操作都会正常返回而不阻塞。

### 3.2.1 poll执行流程

现在，我们来总结一下poll机制的过程：

(1) 调用poll函数，进入系统调用，将pollfd数组拷贝至内核，此时用户进程会进入阻塞状态。
(2) 此时已经从用户空间复制了pollfd数组来存放所有的Socket文件描述符，此时把数组pollfd组织成poll_list链表。
(3) 对poll_list链表进行遍历，判断每个Socket文件描述符的状态，如果某个Socket就绪，就在Socket的就绪队列上加入一项，然后继续遍历。若遍历完所有的文件描述符后，都没发现任何Socket就绪，则继续阻塞当前进程，直到有Socket就绪或者等待超时，就唤醒用户进程，返回就绪队列。

(4) 如果用户进程被唤醒后，Socket就绪队列有就绪的Socket，用户进程就获取所有就绪的Socket，调用系统调用接口（read或write），转入内核态执行read或write操作，此时进程又进入阻塞状态，在内核态执行完系统调用以后，再次通知用户进程，用户进程获取到数据就可以进行处理了。
(5) 如果用户进程因等待超时被唤醒，Socket就绪队列为空，进程会再次执行系统调用，重复步骤1、2、3。

### 3.2.2 poll水平触发

poll还有一个特点是“水平触发”，如果报告了某个就绪的Socket文件描述符后，没有被处理，那么下次poll时会再次报告该Socket fd。

## 3.3 epoll方式

```c++
int epoll_create(int size);  
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  
int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);
```

参数解释

(1) int epoll_create(int size); 

​	创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大，这个参数不同于select()中的第一个参数，给出最大监听的fd+1的值，这里的size并不是限制了epoll所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议，也就是说，size是内核保证能够正确处理的最大句柄数，多于这个最大数时内核可不保证效果。当创建好epoll句柄后，它就会占用一个fd值，所以在使用完epoll后，必须调用**close()**关闭，否则可能导致该进程的fd被耗尽。

(2) int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

该函数是对上面建立的epoll文件句柄执行op操作。例如，将刚建立的socket加入到epoll中让其监控，或者把 epoll正在监控的某个socket句柄移出epoll不再监控它等等。

* epfd：是epoll的文件句柄。
* op：表示op操作，用三个宏来表示：添加EPOLL_CTL_ADD，删除EPOLL_CTL_DEL，修改EPOLL_CTL_MOD，分别表示添加、删除和修改对fd的监听事件。
* fd：是需要监听的fd（文件描述符）
* epoll_event：告诉内核需要监听什么事件，struct epoll_event结构如下：

```c++
struct epoll_event {
    __uint32_t events; /* Epoll events */
    epoll_data_t data; /* User data variable */
};
```

* events可能取值:
  * EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
  * EPOLLOUT：表示对应的文件描述符可以写；
  * EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
  * EPOLLERR：表示对应的文件描述符发生错误；
  * EPOLLHUP：表示对应的文件描述符被挂断；
  * EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
  * EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里

(3) int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);

* epoll_wait在调用时，在给定的timeout时间内，当在监控的所有句柄中有事件发生时（即有Socket就绪时），就返回用户态的进程。

* epfd：是epoll的文件句柄。
* events：从内核得到事件的集合
* maxevents：events的大小
* timeout：超时时间，timeout为负数表示无线等待，timeout为0表示调用后立即返回。

### 3.3.1 epoll执行机制

(1) 执行epoll_create在内核建立专属于epoll的**高速cache区**，并在该缓冲区建立红黑树和就绪链表，用户态传入的文件句柄将被放到红黑树中（第一次拷贝）。
(2) epoll_ctl执行add动作时除了将Socket文件句柄放到红黑树上之外，还向内核注册了该文件句柄的回调函数，内核在检测到某Socket文件句柄就绪，则调用该回调函数，回调函数将文件句柄放到就绪链表。
(3) epoll_wait只监控就绪链表就可以，如果就绪链表有文件句柄，则表示该文件句柄可读（或可写），并返回到用户态（少量的拷贝）；
(4) 由于内核不修改文件句柄的状态位，因此只需要在第一次传入就可以重复监控，直到使用epoll_ctl删除，否则不需要重新传入，因此无多次拷贝。

## 3.4 epoll和poll的区别

(1) epoll同poll一样，也没有文件句柄的数量限制。
(2) select/poll每次系统调用时，都要传递所有监控的socket给内核缓冲区，如果有数以万计的Socket文件句柄，意味着每次都要copy几十几百KB的内存到内核态，非常低效。epoll不需要每次都将Socket文件句柄从用户态拷贝到内核态，在执行epoll_create时已经在内核建立了epoll句柄，每次调用epoll_ctl只是在往内核的数据结构里加入新的socket句柄，所以不需要每次都重新复制一次。
(3) select 和 poll 都是主动轮询，select在内核态轮询所有的fd_set来判断有没有就绪的文件句柄，poll 轮询链表判断有没有就绪的文件句柄，而epoll是被动触发方式。epoll_ctl执行add动作时除了将Socket文件句柄放到红黑树上之外，还向内核注册了该文件句柄的回调函数，当Socket就绪时，则调用该回调函数将文件句柄放到就绪链表，epoll_wait只监控就绪链表就可以判断有没有事件发生了。

# 4. AIO 异步阻塞

这部分参考博客吧



# 5. 小结

可以知道, 为了提高网络io的效率, 让cpu更多的服务请求线程(服务进程). 可以通过以下方面来进行:

* 减少内核态和用户态间的切换
* 减少阻塞的发生, 避免让出cpu

为了让跑服务器的机器上运行b站会很卡, 不准摸鱼!
