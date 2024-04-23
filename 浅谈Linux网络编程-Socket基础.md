---
title: '浅谈Linux网络编程: Socket基础'
date: 2020-10-30 12:06:24
tags: linux
categories: linux
---

# 套接字（socket）基础
套接字是网络编程中，应用层和传输层之间的数据结构，其作用如下:
应用层通过传输层进行数据通信时，TCP和UDP会遇到同时为多个应用程序进程提供并发服务的问题。多个TCP连接或多个应用程序进程可能需要通过同一个TCP协议端口传输数据。为了区别不同的应用程序进程和连接，许多计算机操作系统为应用程序与TCP／IP协议交互提供了称为套接字(Socket)的接口，以区分不同应用程序进程间的网络通信和连接。
# 套接字地址结构
通用套接字地址的结构体sockaddr定义如下：
![1](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051411648.png)
在以太网中，不直接使用sockaddr结构体，而使用sockaddr_in,其定义如下：
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051411109.png)
![3](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051411045.png)

通用结构体sockaddr和以太网的sockaddr_in结构体，有点像C++中的父类和子类的关系，sockaddr_in是对sockaddr的细化，其存储结构大小相同，分布如下：
![4](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051411360.png)
由于大小相同，在设置socket地址时，一般先设置sockaddr_in结构体，然后强转为sockaddr类型

# 套接字地址结构在用户层和内核层的交互
sockaddr的使用，以socket流程中的bind()函数为例：
![5](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051411733.png)
bind函数需要传入sockaddr结构体的指针，和sockaddr结构体的长度

## 向内核传入数据
向内核传入数据的socket函数有：bind,send
传入过程如下：

 - sockaddr结构体的长度，以传值方式传入内核
 - 内核通过sockaddr结构体的指针以及结构体长度，以内存复制的方式，从用户层拷贝sockaddr结构体到内核层。

![6](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051411951.png)

## 从内核获取数据
从内核得到数据的socket函数有：accept,recv
 - sockaddr结构体的长度，以传值方式传入内核
 - 内核通过sockaddr结构体的指针以及结构体长度，以内存复制的方式，从内核层拷贝sockaddr结构体到用户层。
 - 内核返回内核的结构体的长度
![7](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051412551.png)
# Socket编程流程
## 总体架构
TCP编程主要为C/S模式，即服务器和客户端编程，我们讲socket编程，要区分是服务器端的流程还是客户端的流程。
 - 服务器端：创建服务-等待客户端连接-收到连接请求-处理
 - 客户端：发起对服务器的连接请求-根据服务器的响应做处理

![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051412551.png)

服务端各函数含义:

 - socket：套接字初始化
 - bind：绑定套接字和端口
 - listen：配置服务器的请求队列，监测连接请求
 - accept：接受客户端连接
 - read/write：数据的接收、发送
 - close：断开连接，释放套接字

客户端函数：

 - 客户端套接字不需要绑定端口和监听，直接connect发起连接请求，其他函数和服务端一致。

## socket函数
socket函数用于创建socket套接字的文件描述符，

![9](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051412223.png)

有三个入参：

 - domain：域，区分本地，IPV4 Internet，IPV6 Internet等。有的以PF开头，有的以AF开头，这两者值一样。

![10](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051412180.png)

 - type：通信类型，如流式（TCP）,数据报式（UDP）等

![11](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051412975.png)
 - protocal：协议类型，指定通信类型中的子类型，一般为0

socket套接字初始化的一个例子：
![12](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051412776.png)

## socket函数在应用层和内核层的交互
用户调用的socket函数，会调用内核的sys_socket函数

![2](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051415958.png)


sys_socket做两件事：

 - sock_create生成内核的socket结构，和应用层的结构不同，如下：

   ![13](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051413920.png)

 - sock_map_fd将内核socket结构绑定文件描述符fd,用户层可通过fd访问内核socket结构

## bind函数
服务端用socket函数建立套接字文件描述符后，需要绑定地址和端口到该文件描述符，才能接受客户端请求。

![14](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051413732.png)

 - sockfd：socket函数创建的文件描述符
 - sockaddr结构的指针：指向的sockaddr结构，包含ip和port等信息
 - addrlen：即sizeof(struct sockaddr)

bind函数绑定UNIX族的套接字：

![15](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051413483.png)

bind函数绑定AF_INET族的套接字:

![16](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051413289.png)

## bind函数在应用层和内核层的交互
以AF_INET族的套接字绑定为例，不同协议族实际上是调用内核不同的绑定函数
![image-20221205141814380](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051418474.png)

## listen函数
listen函数用于初始化服务器的可连接队列，即服务器处理客户的请求，不是并行处理，而是异步的串行处理。服务器建立可连接队列，将当前不能同步处理的新请求放到队列中，等队列前面的请求处理完了，才异步处理这个请求。
![18](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051414568.png)

 - backlog是服务器可连接队列的最大长度
 - 当前队列没满，即当前队列的请求没超过backlog值，才可以调用accept
 - listen函数只针对SOCK_STREAM和SOCK_SEQPACKET才能调用，因为TCP请求才需要建立连接。对于SOCK_DGRAM不支持listen，因为UDP是无连接的。

TCP连接中，SOCK_STREAM类型的套接字，调用listen的示例：
![image-20221205141912644](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051419762.png)
![image-20221205141921877](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051419995.png)

## listen函数在应用层和内核层的交互

![image-20221205141934130](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051419209.png)

## accept函数
服务端用listen建立连接队列后，客户端以connect发来一个请求，会加入到服务端连接队列的队尾，当这个请求到达队头，会调用accept真正处理该请求。
accept会创建一个新的套接字文件描述符，用来描述客户端的连接，这个时候会有两个套接字描述符并存：

 - socket函数创建的老的sockfd，表示正在监听的ip和端口
 - accept函数创建的新的clientfd，表示当前的客户端连接，后续的客户端的收发和客户端关闭，即send，recv，close函数，都使用clientfd

![image-20221205142030402](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051420456.png)

流式连接的accept示例：
![image-20221205142038423](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051420563.png)
![image-20221205142048069](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051420232.png)

## accept函数在应用层和内核层的交互

![image-20221205142100841](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051421941.png)

## connect函数
connect函数是客户端调用的函数，在客户端调用socket函数创建套接字文件sockfd后，可调用connect函数向服务端发起连接请求，请求的ip和端口信息包含在sockaddr结构体内。
![image-20221205142256082](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051422137.png)

客户端的socket connect示例：
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051422137.png)

## connect函数在应用层和内核层的交互
根据数据流式或数据报式的请求，具体调用inet_stream_connect或inet_dgram_connect
![image-20221205142315046](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051423142.png)

## read和write函数
服务端和客户端真正建立连接后（socket通信逻辑连接，不是TCP/UDP的面向连接/无连接的传输层连接），即客户端发起connect，服务端accept完成，双方就可以相互read/write，读写对方的数据，通过sockfd文件描述符，就像读写本地文件一样。

 - read：从套接字文件读取数据，写入本地缓冲区，返回非空的有效数据大小
 ![image-20221205142328017](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051423066.png)
 - write：向套接字文件写入数据，将本地缓冲区数据写入socket函数创建的socket文件
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051423066.png)


## close和shutdown函数

 - close：关闭socket连接，释放内核的套接字资源，不能通过套接字文件来读写操作
 - shutdown：支持读写的单向关闭，即关闭套接字文件的读、写、或者读写能力（等同于close）

# Socket客户端和服务端交互的例程
## 整体架构
客户端从标准输入读取用户输入的字符串，发送给服务端，服务端读取数据，回写这些数据到客户端，客户端收到数据后输出到标准输出。
![image-20221205142345794](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051423842.png)

客户端和服务端可以在同一台机器部署，访问回环地址127.0.0.1即可，注意服务端的监听端口不能和其他进程的端口冲突。

## 代码实现
服务端代码：

    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    #include <sys/types.h>
    #include <sys/socket.h>
    #include <arpa/inet.h>
    #include <unistd.h>
    //#define PORT 8088						/*侦听端口地址*/
    #define BACKLOG 2						/*侦听队列长度*/
    
    int main(int argc, char *argv[])
    {
    	int ss,sc;		/*ss为服务器的socket描述符，sc为客户端的socket描述符*/
    	struct sockaddr_in server_addr;	/*服务器地址结构*/
    	struct sockaddr_in client_addr;	/*客户端地址结构*/
    	int err;							/*返回值*/
    	pid_t pid;							/*分叉的进行ID*/
    
    	/*建立一个流式套接字*/
    	ss = socket(AF_INET, SOCK_STREAM, 0);
    	if(ss < 0){							/*出错*/
    		printf("socket error\n");
    		return -1;	
    	}
    	
    	/*设置服务器地址*/
    	bzero(&server_addr, sizeof(server_addr));			/*清零*/
    	server_addr.sin_family = AF_INET;					/*协议族*/
    	server_addr.sin_addr.s_addr = htonl(INADDR_ANY);	/*本地地址*/
    	//server_addr.sin_port = htons(PORT);
    	server_addr.sin_port = htons(atoi(argv[1]));		/*服务器端口*/
    	
    	/*绑定地址结构到套接字描述符*/
    	err = bind(ss, (struct sockaddr*)&server_addr, sizeof(server_addr));
    	if(err < 0){/*出错*/
    		printf("bind error\n");
    		return -1;	
    	}
    	
    	/*设置侦听*/
    	err = listen(ss, BACKLOG);
    	if(err < 0){										/*出错*/
    		printf("listen error\n");
    		return -1;	
    	}
    	
    		/*主循环过程*/
    	for(;;)	{
    		socklen_t addrlen = sizeof(struct sockaddr);
    		/*接受客户端连接*/
    		sc = accept(ss, (struct sockaddr*)&client_addr, &addrlen); 
    		if(sc < 0){							/*出错*/
    			continue;						/*结束本次循环*/
    		}	
    		
    		/*建立一个新的进程处理到来的连接*/
    		pid = fork();						/*分叉进程*/
    		if( pid == 0 ){						/*子进程中*/
    			process_conn_server(sc);		/*处理连接*/
    			close(ss);						/*在子进程中关闭服务器的侦听*/
    		}else{
    			close(sc);						/*在父进程中关闭客户端的连接*/
    		}
    	}
    }

服务端注意几点:

 - accept后处理连接的过程，是在子进程中处理的，使用fork创建子进程用于连接处理。根据fork返回的pid是0还是其他，判断当前调度到子进程还是父进程，从全局上来讲，这个`if-else`的两种流程分别在父进程和子进程中指向。
 - 服务端有两个套接字：侦听套接字和连接套接字。处理连接传入的是连接套接字。
 - 在父进程（侦听进程）中，要关闭连接套接字；在子进程（连接处理进程）中，要关闭侦听套接字。这是为了避免子父进程相互影响。
 - 对于多进程，一个进程的套接字关闭不会释放该套接字内存，只有所有进程都关闭了这个套接字，内核才会 释放该套接字，所有可以放心在侦听进程和连接处理进程中，关闭对方的套接字。

客户端代码：

    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    #include <sys/types.h>
    #include <sys/socket.h>
    #include <unistd.h>
    #include <arpa/inet.h>
    //#define PORT 8088								/*侦听端口地址*/
    
    int main(int argc, char *argv[])
    {
    	int s;										/*s为socket描述符*/
    	struct sockaddr_in server_addr;			/*服务器地址结构*/
    	
    	s = socket(AF_INET, SOCK_STREAM, 0); 		/*建立一个流式套接字 */
    	if(s < 0){									/*出错*/
    		printf("socket error\n");
    		return -1;
    	}	
    	
    	/*设置服务器地址*/
    	bzero(&server_addr, sizeof(server_addr));	/*清零*/
    	server_addr.sin_family = AF_INET;					/*协议族*/
    	server_addr.sin_addr.s_addr = htonl(INADDR_ANY);	/*本地地址*/
    	server_addr.sin_port = htons(atoi(argv[2]));		/*服务器端口*/
    	
    	/*将用户输入的字符串类型的IP地址转为整型*/
    	inet_pton(AF_INET, argv[1], &server_addr.sin_addr);	
    	/*连接服务器*/
    	connect(s, (struct sockaddr*)&server_addr, sizeof(struct sockaddr));
    	process_conn_client(s);						/*客户端处理过程*/
    	close(s);									/*关闭连接*/
    	return 0;
    }



建立连接后的读写交互代码，包含服务端的调用和客户端的调用：

    #include <stdio.h>
    #include <string.h>
    /*客户端的处理过程*/
    void process_conn_client(int s)					/* 传入的是客户端调用socket时创建的s */
    {
    	ssize_t size = 0;
    	char buffer[1024];							/*数据的缓冲区*/
    	
    	for(;;){									/*循环处理过程*/
    		/*从标准输入中读取数据放到缓冲区buffer中,标准输入：0，标准输出：1，标准错误：2*/
    		size = read(0, buffer, 1024);
    		if(size > 0){							/*读到数据*/
    			write(s, buffer, size);				/*发送给服务器*/
    			/*客户端阻塞，等待服务器有数据可读*/
    			size = read(s, buffer, 1024);		/*从服务器读取数据*/
    			write(1, buffer, size);				/*写到标准输出*/
    		}
    	}	
    }
    /*服务器对客户端的处理*/
    void process_conn_server(int s) 				/* 传入的是服务端调用accept时创建的sc */
    {
    	ssize_t size = 0;
    	char buffer[1024];							/*数据的缓冲区*/
    	
    	for(;;){									/*循环处理过程*/		
    		size = read(s, buffer, 1024);			/*从套接字中读取数据放到缓冲区buffer中*/
    		if(size == 0){							/*没有数据*/
    			return;	
    		}
    		
    		/*构建响应数据*/
    		//sprintf(buffer, "server receive %d bytes from client\n", size);
    		//write(s, buffer, strlen(buffer));
    		write(s, buffer, size);					/*发回给客户端*/
    	}	
    }

Makefile编译脚本:

    all:client server					#all规则，它依赖于client和server规则
    
    client:tcp_process.o tcp_client.o	#client规则，生成客户端可执行程序
    	gcc -o client tcp_process.o tcp_client.o
    server:tcp_process.o tcp_server.o	#server规则，生成服务器端可执行程序
    	gcc -o server tcp_process.o tcp_server.o	
    tcp_process.o:						#tcp_process.o规则，生成tcp_process.o
    	gcc -c tcp_process.c -o tcp_process.o
    clean:								#清理规则，删除client、server和中间文件
    	rm -f client server *.o

## 部署和运行
后台运行server,指定监听端口:
![image-20221205142401747](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051424795.png)
运行client，指定服务端的ip, port：
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051424795.png)
客户端每输入一个字符串，服务端返回完全相同的字符串，通信正常
如果运行服务端时，有bind error，可能是端口被占用，`netstat`找到占用端口的PID，kill之后再运行server
![image-20221205142416741](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051424796.png)
