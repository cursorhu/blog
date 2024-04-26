---
title: 浅谈Linux进程的内存分布和进程IPC
date: 2020-11-15 20:38:39
tags: linux
categories: linux
---

# Linux虚拟内存空间分布

（1）虚拟内存空间与物理内存：
带MMU控制器的CPU支持将物理内存以分页的方式，细粒度的动态分配给进程，使每个进程只看得到这个虚拟的内存空间，每个进程认为自己可以访问整个内存空间。进程根本不知道其访问的某个内存页的实际物理地址，也许在SDRAM上，或者硬盘的交换分区上。

进程的虚拟地址通过页表（page table）映射到物理内存，页表由操作系统维护并被处理器引用。每个进程都拥有一套属于它自己的页表。

（2）下面讨论用户进程能看到什么样的虚拟内存空间：

以32位系统为例，CPU可寻址4GB的内存空间。此时虚拟地址空间范围为0～4G，Linux内核将这4G字节的空间分为两部分：

 - 将最高的1G字节（从虚拟地址0xC0000000到0xFFFFFFFF）供内核使用，称为“内核空间”。
 - 将较低的3G字节（从虚拟地址0x00000000到0xBFFFFFFF）供各个进程使用，称为“用户空间

因为每个进程可以通过**系统调用**进入内核，因此，Linux内核由系统内的所有进程共享。从具体进程的角度来看，每个进程可以拥有4G字节的虚拟空间。

![image-20221205115648795](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212051156842.png)

注意：

 - 内核可见的内存空间只有全局的1GB; 用户进程可见的内存空间包括该进程独有的3GB空间，和全局内核的1GB;
 - 用户进程虽然可见内核空间的1GB，但不可直接访问，要通过系统调用（或中断等方式），涉及上下文切换；
 - 当进程访问内核空间时，称为“进入内核态”，返回时称为“进入用户态”；
 - 内核空间分布在虚拟内存空间的高地址，用户空间在低地址

（3）用户进程的内部空间详解

编译好的程序都分为几个段(section)，在程序运行过程中的临时变量还产生堆栈，程序手动分配的内存使用堆, 还有命令行参数和环境变量等配置信息，这些东西都属于进程空间的数据。

![image-20221205115908003](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212051159094.png)

详解如下：
代码段(Text):存放程序指令，一些只读数据(.rodata)也可归为此类
数据段(Data):存放初始化过的全局数据
BSS段:存放未初始化(默认为0)的全局数据
栈 (Stack): 用于控制函数调用和返回过程中的临时变量，存储函数内的临时变量; 存储函数的返回指针，
堆 (Heap):存储动态内存分配, 需要程序员手工分配, 手工释放。注意与数据结构中的堆(优先队列)是不同，分配方式类似于链表。

# Linux进程间通信(IPC)
进程本身是为了隔离程序的资源，但不同程序间可能有数据通信或调用关系，因此需要进程通信机制。

进程通信最主要的几种方式有：管道(pipe) , 共享内存(shared memory), 消息队列(message queue), socket等。为了进程间的时序同步和资源处理，信号量(semaphore)通常配合使用。

本节重点讲管道和共享内存，关于Linux IPC 的全面内容，参考：
[An introduction to Linux IPC](http://www.cs.fsu.edu/~zwang/files/cop4610/Fall2016/linux-ipc.pdf)
[inter-process_communication_in_linux](https://opensource.com/sites/default/files/gated-content/inter-process_communication_in_linux.pdf)

## 进程通信的基本思路
根据上节的内存空间分布，所有进程共享同一个内核空间，最简单的进程通信就是通过 进程A->内核->进程B：
![1637063328269_12](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212051157104.png)

以上虽然可以实现，但有两次拷贝以及上下文切换，其总体思路是管道和共享内存方式的基础。

## 管道
管道的实质就是一个内核缓冲区；
管道对于管道两端的进程而言就是一个文件，与普通文件的区别是管道只存在于内存中；
进程通过读写管道文件，传递数据；

管道依据是否有名字分为匿名管道和命名管道，其功能有以下区别：
匿名管道(通常管道就是指匿名管道)：

 - 半双工的，即管道设置好后，数据只能从进程A到进程B；如果还需要从B到A,需要创建另外的管道
 - 只能用于父子进程或兄弟进程之间的通信

命名管道(FIFO)：

 - 可用于无关联进程的通信，其基本原理和匿名管道一样，本节不详细描述

管道内部提供了同步机制
临界资源： 大家都能访问到的共享资源
临界区： 对临界资源进行操作的代码
同步： 临界资源访问的可控时序性（一个操作完另一个才可以操作）
互斥： 对临界资源同一时间的唯一访问性（保护临界资源安全）

### (匿名)管道使用三部曲

1.创建本进程的管道
使用pipe函数创建管道文件
![image-20221205115729244](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212051157310.png)

2.fork子进程，共享管道
![image-20221205115734973](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212051157031.png)

3.设置管道为单向
![image-20221205115744442](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212051157508.png)

## 共享内存

Linux中每个进程都有属于自己的进程控制块（PCB）和地址空间（Addr Space），并且都有一个与之对应的页表，负责将进程的虚拟地址与物理地址进行映射，通过内存管理单元（MMU）进行管理。

两个不同的虚拟地址通过页表映射到物理空间的同一区域，它们所指向的这块区域即共享内存。

共享内存的通信原理：
![image-20221205115751869](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212051157936.png)

共享内存的关键是一份内存资源被两个进程占用，因此需要信号量等同步机制，实现进程同步与资源互斥。

这里简单说明我对信号量的理解：

 - 信号量的作用是“流程同步”，这个流程可以是两个进程访问共享内存，也可以是同一进程内的多个线程访问共享数据；
 - 注意，信号量并不一定用于共享资源的情景，可能只是简单的主线程等待工作线程这种情况。这是其和互斥锁的关键区别；
 - 信号量如果用于共享资源，其本质是“引用计数”，即共享资源是否可用的计数，计数为0表示无资源可用。各进程如果获得资源计数-1，释放资源计数+1。

# 参考文章
[Linux进程地址空间和进程的内存分布](https://blog.csdn.net/cl_linux/article/details/80328608)
[An introduction to Linux IPC](http://www.cs.fsu.edu/~zwang/files/cop4610/Fall2016/linux-ipc.pdf)
[inter-process_communication_in_linux](https://opensource.com/sites/default/files/gated-content/inter-process_communication_in_linux.pdf)
[Linux 进程间通信（IPC）总结](https://www.cnblogs.com/huansky/p/13170125.html#:~:text=Linux%20%E8%BF%9B%E7%A8%8B%E9%97%B4%E5%9F%BA%E6%9C%AC%E7%9A%84%E9%80%9A%E4%BF%A1%E6%96%B9%E5%BC%8F%E4%B8%BB%E8%A6%81%E6%9C%89%EF%BC%9A%E7%AE%A1%E9%81%93%20%28pipe%29,%28%E5%8C%85%E6%8B%AC%E5%8C%BF%E5%90%8D%E7%AE%A1%E9%81%93%E5%92%8C%E5%91%BD%E5%90%8D%E7%AE%A1%E9%81%93%29%E3%80%81%E4%BF%A1%E5%8F%B7%20%28signal%29%E3%80%81%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97%20%28queue%29%E3%80%81%E5%85%B1%E4%BA%AB%E5%86%85%E5%AD%98%E3%80%81%E4%BF%A1%E5%8F%B7%E9%87%8F%E5%92%8C%E5%A5%97%E6%8E%A5%E5%AD%97%E3%80%82)

