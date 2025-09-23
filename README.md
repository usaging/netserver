## 介绍

主从Reactor多线程模型 (Master-Slave Reactors)：

Main Reactor： 运行在主线程，负责监听和接受新连接。其事件循环通过epoll监听listenfd上的EPOLLIN事件。

Acceptor： 作为Main Reactor的处理器，专门处理新连接请求。接受连接后，使用Round-Robin策略将新连接分发给某个Sub Reactor。

Sub Reactors： 多个Sub Reactor（从反应器）分别运行在独立的IO线程中，每个都拥有自己的epoll实例和事件循环。负责监听所有已分配连接的读写事件，实现真正的I/O多路复用。

Worker Thread Pool： 一个额外的线程池，负责处理计算密集型或可能阻塞的业务逻辑（如数据库查询），确保I/O线程不被阻塞，保持高响应速度。

事件驱动与核心组件 (Event-Driven & Core Components)：

EventLoop： 核心类，每个Reactor线程一个。封装了epoll_wait的事件循环，负责执行事件分发、定时任务和异步回调。

Channel： 每个文件描述符（fd）对应一个，封装了fd、关心的事件（如EPOLLIN | EPOLLOUT）以及对应的读/写/错误回调函数。是事件处理的最小单元。

Epoll： EventLoop的成员，是对epoll系统调用的面向对象封装（使用epoll_create, epoll_ctl, epoll_wait）。

ThreadPool： 基于C++11的std::thread、std::mutex和std::condition_variable实现的简单线程池，用于业务计算。

# 类的功能

InetAddress：socket的地址协议类，封装 IP 地址和端口号。

Socket：封装 socket 文件描述符，提供bind、listen、accept等操作。

Epoll：封装epoll。

EventLoop：事件循环核心类，每个线程一个EventLoop。包括主事件循环和从事件循环。

Channel：封装文件描述符和事件回调，是事件处理的基本单元。

Acceptor：对Channel的封装，用于接受新的连接，是 Main Reactor 的处理器。

Connection：对Channel的封装，表示已连接上来的客户端。

Buffer：应用层缓冲区，处理数据读写和粘包问题。

ThreadPool：线程池。

TcpServer：服务器主类，协调所有组件工作。拥有主事件循环和从事件循环。

Timestamp：时间戳相关功能，用于定时器。

EchoServer：回显服务器。

# 学习体会

## Reactor架构

单线程的Reactor模型不能发挥多核CPU的性能。

Acceptor运行在主Reactor(主进程)中，Connection运行在从Reactor(线程池)中。

主线程负责创建客户端连接，然后把conn分配给线程池。

一个从Reactor负责多个Connection，每个Connection的工作内容包括IO和计算(处理业务)。

IO不会阻塞事件循环，但是计算可能会阻塞事件循环。如果计算阻塞了事件循环，那么在同一Reactor中的全部Connection将会被阻塞。

## 服务端关心的事件

1. 处理新客户端连接请求。
2. 关闭客户端的连接。
3. 客户端的连接错误。
4. 处理客户端的请求报文。
5. 数据发送完成后。
6. epoll_wait()超时。

## epoll_event

在 Linux 的 epoll 机制中，当你使用 epoll_ctl(EPOLL_CTL_ADD, ...) 向 epoll 实例添加一个文件描述符（fd）时，你需要同时提供一个 epoll_event 结构体。这个结构体包含两个主要部分：

events: 你关心的事件（如 EPOLLIN | EPOLLOUT）。

data: 一个联合体（union），用于存放你希望与这个 fd 关联的用户数据。当事件发生时，epoll_wait 返回的 epoll_event 结构体中的 data 字段就是你当初设置的值。

epoll_data 联合体通常有几种用法：
'''
typedef union epoll_data {
    void    *ptr;  // 最常用、最灵活的方式，可以指向任何东西
    int      fd;   // 可以直接存放文件描述符本身
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;
'''

最常见和强大的用法是使用 ptr 指针，指向一个自定义的结构体，这个结构体可以包含处理事件所需的所有信息（比如 fd、回调函数等）。

## Channel

Channel 类理解为 epoll 中 epoll_data 的一个面向对象的、功能强大的封装和扩展。

Channel 类是 Reactor 模式的核心组件之一。它的核心职责是：为一个文件描述符（fd）封装其相关的事件和事件处理逻辑。

一个典型的 Channel 类通常包含以下成员：

fd_: 它所负责的文件描述符（socket、timerfd等）。

events_: 它当前关心的事件（类似于 epoll_event.events），如可读、可写。

revents_: 由 epoll_wait 返回的、当前实际发生的事件。

readCallback_, writeCallback_, errorCallback_,closeCallback_: 回调函数。这些是 Channel 类的灵魂，它定义了当对应事件发生时应该执行什么操作。

## buffer

在非阻塞的网络服务程序中，事件循环不会阻塞在recv和send中，如果数据接收不完整，或者发送缓冲区已填满，都不能等待。所以buffer是必须的。

在Reactor模型中，每个Connection对象拥有inputbuffer和sendbuffer。

接收缓冲区:

TcpConnection必须要有inputbuffer。TCP是一个无边界的字节流协议，接收方必须要处理“收到的数据尚不构成一条完整的消息”和“一次收到两条消息的数据”等情况。一个常见的情景是，发送方send()了两条信息，接收方可能分多次收到。

网络库在处理socket可读事件的时候，必须一次性把socket里的数据读完（从操作系统buffer搬到应用层buffer），否则会反复触发POLLIN事件，造成busy-loop。那么网络库必然要应对数据不完整的情况，收到的数据先放到inputbuffer里，等构成一条完整信息再通知程序的业务逻辑。所以，在TCP网络编程中，网络库必须要给每个TCPconnection配置inputbuffer。

发送缓冲区:

TcpConnection必须要有outputbuffer。考虑一个常见场景：程序想通过TCP连接发送数据，但是在write()调用中，操作系统只接收了一部分，你肯定不想在原地等待，因为不知道会等多久。程序应该尽快交出控制权，返回EventLoop。对于应用程序而言，它只管生成数据，不应该关系到底数据是一次性发送还是分成几次发送，这些应该由网络库来操心。程序只用调用send()就行，网络库会负责到底。网络库应该接管剩余的数据，把它保存在该TcpConnection的outputbuffer里，然后注册POLLOUT事件，一旦socket变得可写就立刻发送数据。如果还有剩余，网络库应该继续关注POLLOUT事件。如果数据发送完了，网络库应该停止关注POLLOUT，以免造成busyloop。

## 异步唤醒事件循环

如果由工作线程负责发送数据，就会和IO线程在buffer内发生资源的竞争，加锁的话开销过大，所以应该把发送数据的任务交给IO线程。

通知线程的方法：条件变量、信号量、socket、管道、eventfd。

事件循环阻塞在epoll_wait函数中，条件变量、信号量有自己的等待函数，不适合用于通知事件循环。

socket、管道、eventfd都是fd，可加入epoll，如果要通知事件循环，往socket、管道、eventfd中写入数据即可。

## 清理空闲的Connection

空闲的Connection对象是指长时间没有进行通讯的Tcp连接。

空闲的Connection对象会占用资源，需要定期清理。清理Connection还可以避免攻击。

定时器用于执行定时任务，例如清理空闲的tcp连接。

传统的做法，alarm()函数可设置定时器，触发SIGALRM信号。

新版Linux内核把定时器和信号抽象为fd，让epoll统一监视。

在事件循环中添加闹钟，闹钟响了就遍历Connection Map，判断每个Connection是否超时，如果超时就删除。

## 智能指针的使用

独占所有权	std::unique_ptr

共享所有权	std::shared_ptr

需要使用权，且资源可能失效，我需要安全地“感知”	std::weak_ptr

需要使用权，但我能 100% 保证在使用期间资源绝对有效（例如，在另一个对象的函数内，使用其成员变量的裸指针）	裸指针 (T*) 或 引用 (T&)

与C语言接口或旧式API交互	裸指针 (通过 shared_ptr.get())

## 主流网络库的实现

1. libevent
特点：历史最悠久、最经典、使用最广泛的C网络库之一（如Memcached、Chromium早期使用它）。API稳定，社区庞大，文档丰富。
2. libev
特点：是libevent的一个衍生品，设计更轻量、更高效。API更简洁，性能在某种程度上优于libevent。但它只关注事件循环，功能更纯粹。
3. libuv
特点：Node.js的底层引擎，为Node.js而生。它的最大特点是跨平台，在Windows上使用IOCP，在Linux上使用epoll，在Mac上使用kqueue，封装了底层差异。
4. libhv
特点：国产的优秀网络库，作者是中国人。接口设计非常友好，提供了一站式的网络解决方案（HTTP/SERVER/CLIENT/UDP/TCP...），API风格类似libuv和Node.js。

libevent 因为设计缺陷，例如使用全局变量，定时器无法处理时间跳变，诸如此类的设计缺陷导致了 libev 的出现。
然而 libev 还未来得及被C的缺点拖累，因为他不支持 IOCP. 于是 libuv 就出来给 libev 擦屁股了。

preactor模式(难写)
asio和ACE。
Muduo
纯c++写的库，仅在linux下使用。
boost::asio
C++语言跨平台。用bind做回调也并不比虚函数好，看上去灵活了，代价却更高了。不光是运行时的内存和时间代价，编译时间也更长。基于ASIO开发应用，要求程序员熟悉函数对象，函数指针，熟悉boost库中的boost::bind，内存管理控制方面。
asio是一个高性能的网络开发库，Windows下使用IOCP，Linux下使用epoll。
考虑到asio的使用，对现有系统的改造难度大


## Reactor VS Proactor

核心区别：I/O操作的发起与完成

理解两者差异的关键在于谁来执行实际的I/O操作（数据在内核空间和用户空间的拷贝）。

Reactor (非阻塞同步网络模式)

模式：“你来问，有我就给你”。

过程：

应用程序（Handler）向Reactor注册读就绪事件。

Reactor监听，当数据到达内核缓冲区（可读）时，通知应用程序。

应用程序收到通知后，自己调用read()函数，将数据从内核缓冲区同步地、非阻塞地读到用户空间。

角色：应用程序是主动去读取数据的一方。

Proactor (异步网络模式)

模式：“你放着，我好了给你送过来”。

过程：

应用程序发起一个异步I/O操作（如aio_read），并提供一个缓冲区地址和回调函数。

操作系统负责完成整个I/O操作：将数据从网络读到内核缓冲区，然后再从内核缓冲区拷贝到应用程序提供的用户空间缓冲区。

操作完成后，操作系统通知应用程序（通过回调函数或完成事件）。

角色：应用程序是被动接收数据的一方，整个数据拷贝过程由操作系统代劳。

Reactor因为Linux和epoll的成功，成为了当今高性能网络编程中无可争议的最主流架构。 Proactor是一种理论上更优雅的模式，但由于操作系统支持（尤其是在Linux上）和生态的原因，它并未能取代Reactor。

## 高并发性能指标

QPS、TPS、RT、并发数、吞吐量

QPS：Queries Per Second 每秒查询
RPS：Request Per Second 每秒请求数
TPS：Transactions Per Second 每秒事务
RT：Response-time 响应时间
Concurrency 并发数
吞吐量
系统的吞吐量（承压能力）和处理对CPU的消耗、外部接口、IO等因素紧密关联。单个处理请求对CPU消耗越高，外部系统接口、IO速度越慢，系统吞吐能力越低，反之越高。
系统吞吐量有几个重要指标参数：QPS（TPS）、并发数、响应时间。
QPS（TPS）：（Queries Per Second）每秒钟请求/事务数量。
并发数： 系统同时处理的请求/事务数。
响应时间： 一般取平均响应时间。

QPS（TPS）= 并发数/平均响应时间
并发数 = QPS*平均响应时间

最佳线程数量=（（线程等待时间+线程cpu时间）/ 线程cpu时间）* cpu数量

## wrk结果解读思路

1. QPS 稳定 + 延迟低 → 系统性能良好。

2. QPS 上不去，CPU 没打满 → 可能是单线程瓶颈/锁竞争/网络配置问题。

3. 高延迟尾部 (P99 拉高) → 可能有 GC、I/O 慢查询、线程饥饿。

4. 带宽跑满但 CPU 充裕 → 网络瓶颈。

5. CPU 打满但 QPS 低 → 程序实现效率不足，需要优化算法/框架。


## 系统参数调优
服务器“并发”的概念：
①服务器能够同时承担的客户端数量（最基本要求）
②能处理指定数量以上的相应请求
③能够对数据库进行操作
④有磁盘的操作
⑤CPU的占用率最好不要超过60%
⑥内存占用率最好不要超过80%

默认情况下，系统有限制，一个进程最多只能1024个文件（或文件描述符）。
通过ulimit命令可以查看。
修改最大文件数：
第一步：
通过ulimit命令或者修改/etc/security/limits.conf配置文件，修改用户可打开的最大文件描述符个数。由于ulimit命令修改的结果不能永久保存，因此我们此处修改/etc/security/limits.conf配置文件
下面修改/etc/security/limits.conf，将所有用户可打开的文件描述符个数修改为1048576（1024*1024）
第二步：
我们虽然修改了/etc/security/limits.conf，但是该配置文件约束的是“用户级别”的系统限制。还有两个“内核级别”的配置参数与文件描述符有关。一个为nr_open（表示单个进程打开文件句柄数上限），另一个为file-max（表示系统范围内所有进程可打开的文件句柄的数量限制）
其中nr_open默认为1048576，我们就不需要修改了
file-max默认为180566，因此我们需要将其修改为1048576
修改/etc/sysctl.conf文件，在里面修改file-max的值为1048576，然后保存退出
执行sysctl的命令，将/etc/sysctl.conf文件的内容更新到/proc/sys/fs/file-max中生效，令执行完之后查看/proc/sys/fs/file-max文件，更新成功。
第三步：
还需要修改服务端程序中的MAX_EPOLL_EVENTS，因为其代表的是epoll_wait()可以处理事件的数量。

一个套接字fd就对应一个“五元组”，一个五元组包含下面几项内容：
1.源IP地址
2.目的IP地址
3.源端口
4.目的端口
5.协议类型（TCP、UDP等）

源IP地址、目的IP地址、协议类型（TCP）、目的端口都保持不变，唯一可以变化的就是源端口（也就是客户端的端口）。
使用相同IP地址进行测试的话会出现五元组”组合用尽的情况。在实际开发中是不会出现的，因为每个客户端的IP地址都是不同的。

nf_conntrack_max限制了每一个程序可以建立的TCP的连接的最大数目（是连接数目，而不是套接字描述符的个数）。
我们只需要修改客户端的内核参数nf_conntrack_max就可以，可以将其修改为1048576
第一步：nf_conntrack_max参数默认是不开启的，需要先加载下面的内核模块，加载完成之后其值默认为65536
sudo modprobe ip_conntrack
第二步：然后在客户端的/etc/sysctl.conf配置文件中将nf_conntrack_max设置为1048576
第三步：调用sysctl命令将/etc/sysctl.conf配置文件的内容更新到/proc/sys/net/nf_conntrack_max中。

实际过程中也不会出现这个问题，因为不可能有一个客户端程序会发起上万个连接。

## 用户态协议栈设计

在服务器承载百万客户端的时候，这时候会有大量的数据在进行交互。对于默认的应用程序来说，其数据的传输都要经过协议栈缓冲区。
客户端发送数据到服务端时，数据先达到网卡，然后将数据拷贝到协议栈中，最后再把数据从协议栈中拷贝到服务端程序中。
服务端回送数据时是相反的顺序，先把数据从服务端程序拷贝到协议栈，然后再把数据拷贝到网卡发送给客户端。

当服务端与百万客户端数据交互的时候会带来下面的弊端：
数据要进行大量的拷贝，从而造成程序的性能降低。
数据交互速度受限与缓慢。

数据要经过一层协议栈，会对性能带来影响，因此可以自己设计“用户态协议栈”，大致原理如下：
数据交互时不再经过协议栈，将网卡与服务端的数据mmap映射到内存中，直接从内存中进行交互，从而减少了中间的拷贝
这就是常说的“零拷贝”。
