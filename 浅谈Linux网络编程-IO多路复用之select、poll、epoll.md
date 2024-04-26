---
title: '浅谈Linux网络编程: IO多路复用之select、poll、epoll'
date: 2020-12-07 13:58:53
tags: linux
categories: linux
---

# 概述
I/O多路复用：通过一种机制，可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。

但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。

IO多路复用适用如下场合：

 - 当客户处理多个描述符时（一般是交互式输入和网络套接口），必须使用I/O复用。
 - 当一个客户同时处理多个套接口时，而这种情况是可能的，但很少出现。
 - 如果一个TCP服务器既要处理监听套接口，又要处理已连接套接口，一般也要用到I/O复用。
 - 如果一个服务器即要处理TCP，又要处理UDP，一般要使用I/O复用。
 - 如果一个服务器要处理多个服务或多个协议，一般要使用I/O复用。

与多进程和多线程技术相比，I/O多路复用技术的最大优势是系统开销小，系统不必创建进程/线程，也不必维护这些进程/线程，从而大大减小了系统的开销。

# select实现

逻辑时序：
![1](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212051401453.png)
具体实现：
![2](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212051401118.png)

fd_set(监听的端口个数)：32位机默认是1024个，64位机默认是2048。

（1）使用copy_from_user从用户空间拷贝fd_set到内核空间
（2）注册回调函数__pollwait
（3）遍历所有fd，调用其对应的poll方法（对于socket，这个poll方法是sock_poll，sock_poll根据情况会调用到tcp_poll,udp_poll或者datagram_poll）
（4）以tcp_poll为例，其核心实现就是__pollwait，也就是上面注册的回调函数。
（5）__pollwait的主要工作就是把current（当前进程）挂到设备的等待队列中，不同的设备有不同的等待队列，对于tcp_poll来说，其等待队列是sk->sk_sleep（注意把进程挂到等待队列中并不代表进程已经睡眠了）。在设备收到一条消息（网络设备）或填写完文件数据（磁盘设备）后，会唤醒设备等待队列上睡眠的进程，这时current便被唤醒了。
（6）poll方法返回时会返回一个描述读写操作是否就绪的mask掩码，根据这个mask掩码给fd_set赋值。
（7）如果遍历完所有的fd，还没有返回一个可读写的mask掩码，则会调用schedule_timeout是调用select的进程（也就是current）进入睡眠。当设备驱动发生自身资源可读写后，会唤醒其等待队列上睡眠的进程。如果超过一定的超时时间（schedule_timeout指定），还是没人唤醒，则调用select的进程会重新被唤醒获得CPU，进而重新遍历fd，判断有没有就绪的fd。
（8）把fd_set从内核空间拷贝到用户空间。

select的几大缺点：
（1）每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大
（2）同时每次调用select都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大
（3）select支持的文件描述符数量太小了，默认是1024

# poll实现
poll的实现和select非常相似，只是描述fd集合的方式不同，poll使用pollfd结构而不是select的fd_set结构，而使其没有连接数的限制。其他的都差不多。

# epoll
epoll支持水平触发和边缘触发，最大的特点在于边缘触发，它只告诉进程哪些fd刚刚变为就绪态，并且只会通知一次。还有一个特点是，epoll使用“事件”的就绪通知方式，通过epoll_ctl注册fd，一旦该fd就绪，内核就会采用类似callback的回调机制来激活该fd，epoll_wait便可以收到通知。
![3](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212051401522.png)

## epoll的几大改进
epoll既然是对select和poll的改进，就应该能避免上述的三个缺点。那epoll都是怎么解决的呢？在此之前，我们先看一下epoll和select和poll的调用接口上的不同，select和poll都只提供了一个函数——select或者poll函数。而epoll提供了三个函数，epoll_create,epoll_ctl和epoll_wait，epoll_create是创建一个epoll句柄；epoll_ctl是注册要监听的事件类型；epoll_wait则是等待事件的产生。

对于第一个缺点，epoll的解决方案在epoll_ctl函数中。
每次注册新的事件到epoll句柄中时（在epoll_ctl中指定EPOLL_CTL_ADD），会把所有的fd拷贝进内核，而不是在epoll_wait的时候重复拷贝。epoll保证了每个fd在整个过程中只会拷贝一次。

对于第二个缺点，epoll的解决方案不像select或poll一样每次都把current轮流加入fd对应的设备等待队列中。
而只在epoll_ctl时把current挂一遍（这一遍必不可少）并为每个fd指定一个回调函数，当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，而这个回调函数会把就绪的fd加入一个就绪链表）。epoll_wait的工作实际上就是在这个就绪链表中查看有没有就绪的fd（利用schedule_timeout()实现睡一会，判断一会的效果，和select实现中的第7步是类似的）。即epoll最大的优点就在于它只管你“活跃”的连接，而跟连接总数无关。

对于第三个缺点，epoll没有这个限制。
它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。

## epoll小结：

（1）select，poll实现需要自己不断轮询所有fd集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而epoll其实也需要调用epoll_wait不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是它是设备就绪时，调用回调函数，把就绪fd放入就绪链表中，并唤醒在epoll_wait中进入睡眠的进程。虽然都要睡眠和交替，但是select和poll在“醒着”的时候要遍历整个fd集合，而epoll在“醒着”的时候只要判断一下就绪链表是否为空就行了，这节省了大量的CPU时间。这就是回调机制带来的性能提升。
（2）select，poll每次调用都要把fd集合从用户态往内核态拷贝一次，并且要把current往设备等待队列中挂一次，而epoll只要一次拷贝，而且把current往等待队列上挂也只挂一次（在epoll_wait的开始，注意这里的等待队列并不是设备等待队列，只是一个epoll内部定义的等待队列）。这也能节省不少的开销。

# select、poll、epoll区别
1. 支持一个进程所能打开的最大连接数
![4](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212051402990.png)

2. FD剧增后带来的IO效率问题
![5](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212051402536.png)

3. 消息传递方式
![](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212051403370.png)

# 总结
在选择select，poll，epoll时要根据具体的使用场合以及这三种方式的自身特点：

 - 连接数多，活跃链接占比不高的场景下，epoll的性能最好
 - 在连接数少并且连接都十分活跃的情况下，select和poll的性能可能比epoll好，毕竟epoll的通知机制需要很多函数回调。
 - select低效是因为每次它都需要轮询。但低效也是相对的，可通过良好的设计改善。
