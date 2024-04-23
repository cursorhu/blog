---
title: 浅谈Linux的Signal信号
date: 2020-11-08 18:53:03
tags: linux
categories: linux
---

本文将从以下几个方面来阐述信号:

(1) 信号的基本知识
(2) 信号生命周期与处理过程分析
(3) 基本的信号处理函数
(4) 保护临界区不被中断
(5) 信号的继承与执行
(6) 实时信号中锁的研究

# 第一部分: 信号的基本知识
## 1.信号本质:
信号的本质是软件层次上对中断的一种模拟。它是一种异步通信的处理机制，事实上，进程并不知道信号何时到来。
## 2.信号来源
(1)程序错误，如非法访问内存
(2)外部信号，如按下了CTRL+C
(3)通过kill或sigqueue向另外一个进程发送信号
## 3.信号种类
信号分为可靠信号与不可靠信号,可靠信号又称为实时信号，非可靠信号又称为非实时信号。
信号代码从1到32是不可靠信号,不可靠信号主要有以下问题:
(1)每次信号处理完之后，就会恢复成默认处理，这可能是调用者不希望看到的
(2)存在信号丢失的问题
现在的Linux对信号机制进行了改进，因此，不可靠信号主要是指信号丢失
信号代码从SIGRTMIN到SIGRTMAX之间的信号是可靠信号。可靠信号不存在丢失，由sigqueue发送，可靠信号支持排队。

可靠信号注册机制:
内核每收到一个可靠信号都会去注册这个信号，在信号的未决信号链中分配sigqueue结构，因此，不会存在信号丢失的问题。

不可靠信号的注册机制:
而对于不可靠的信号，如果内核已经注册了这个信号，那么便不会再去注册，对于进程来说，便不会知道本次信号的发生。
可靠信号与不可靠信号与发送函数没有关系，取决于信号代码，前面的32种信号就是不可靠信号，而后面的32种信号就是可靠信号。

## 4.信号响应的方式
(1)采用系统默认处理SIG_DFL,执行缺省操作
(2)捕捉信号处理，即用户自定义的信号处理函数来处理
(3)忽略信号SIG_IGN ,但有两种信号不能被忽略SIGKILL，SIGSTOP

# 第二部分: 信号的生命周期与处理过程分析
## 1. 信号的生命周期
信号产生->信号注册－>信号在进程中注销->信号处理函数执行完毕

(1)信号的产生是指触发信号的事件的发生

(2)信号注册
指的是在目标进程中注册，该目标进程中有未决信号的信息:

    struct sigpending pending：
    struct sigpending{
    struct sigqueue *head, **tail;
    sigset_t signal;
    };
    
    struct sigqueue{
    struct sigqueue *next;
    siginfo_t info;
    }

其中 sigqueue结构组成的链称之为未决信号链，sigset_t称之为未决信号集。
*head,**tail分别指向未决信号链的头部与尾部。
siginfo_t info是信号所携带的信息。
信号注册的过程就是将信号值加入到未决信号集siginfo_t中，将信号所携带的信息加入到未决信号链的某一个sigqueue中去。
 因此，对于可靠的信号，可能存在多个未决信号的sigqueue结构，对于每次信号到来都会注册。而不可靠信号只注册一次，只有一个sigqueue结构。
只要信号在进程的未决信号集中，表明进程已经知道这些信号了，还没来得及处理，或者是这些信号被阻塞。

(3)信号在目标进程中注销
 在进程的执行过程中，每次从系统调用或中断返回用户空间的时候，都会检查是否有信号没有被处理。如果这些信号没有被阻塞，那么就调用相应的信号处理函数来处理这些信号。则调用信号处理函数之前，进程会把信号在未决信号链中的sigqueue结构卸掉。是否从未决信号集中把信号删除掉，对于实时信号与非实时信号是不相同的。
非实时信号:由于非实时信号在未决信号链中只有一个sigqueue结构，因此将它删除的同时将信号从未决信号集中删除。
实时信号:由于实时信号在未决信号链中可能有多个sigqueue结构，如果只有一个，也将信号从未决信号集中删除掉。如果有多个那么不从未决信号集中删除信号，注销完毕。

(4)信号处理函数执行完毕
执行处理函数，本次信号在进程中响应完毕。
在第4步，只简单的描述了信号处理函数执行完毕，就完成了本次信号的响应，但这个信号处理函数空间是怎么处理的呢? 内核栈与用户栈是怎么工作的呢? 这就涉及到了信号处理函数的过程。

## 2. 信号处理函数的过程:
(1)注册信号处理函数
信号的处理是由内核来代理的，首先程序通过sigal或sigaction函数为每个信号注册处理函数，而内核中维护一张信号向量表，对应信号处理机制。这样，在信号在进程中注销完毕之后，会调用相应的处理函数进行处理。

(2)信号的检测与响应时机
在系统调用或中断返回用户态的前夕，内核会检查未决信号集，进行相应的信号处理。

(3)处理过程:
程序运行在用户态时->进程由于系统调用或中断进入内核->转向用户态执行信号处理函数->信号处理函数完毕后进入内核->返回用户态继续执行程序
首先程序执行在用户态，在进程陷入内核并从内核返回的前夕，会去检查有没有信号没有被处理，如果有且没有被阻塞就会调用相应的信号处理程序去处理。首先，内核在用户栈上创建一个层，该层中将返回地址设置成信号处理函数的地址，这样，从内核返回用户态时，就会执行这个信号处理函数。当信号处理函数执行完，会再次进入内核，主要是检测有没有信号没有处理，以及恢复原先程序中断执行点，恢复内核栈等工作，这样，当从内核返回后便返回到原先程序执行的地方了。
信号处理函数的过程大概是这样了。
具体的可参考http://www.spongeliu.com/linux/linux内核信号处理机制介绍/

# 第三部分: 基本的信号处理函数
首先看一个两个概念: 信号未决与信号阻塞
信号未决: 指的是信号的产生到信号处理之前所处的一种状态。确切的说，是信号的产生到信号注销之间的状态。
信号阻塞: 指的是阻塞信号被处理，是一种信号处理方式。

## 1. 信号操作

 信号操作最常用的方法是信号的屏蔽，信号屏蔽主要用到以下几个函数:

    int sigemptyset(sigset_t *set);
    int sigfillset(sigset_t *set);
    int sigaddset(sigset_t *set,int signo);
    int sigdelset(sigset_t *set,int signo);
    int sigismemeber(sigset_t* set,int signo);
    int sigprocmask(int how,const sigset_t*set,sigset_t *oset);

信号集，信号掩码，未决集
信号集: 所有的信号阻塞函数都使用一个称之为信号集的结构表明其所受到的影响。
信号掩码:当前正在被阻塞的信号集。
未决集: 进程在收到信号时到信号在未被处理之前信号所处的集合称为未决集。
可以看出，这三个概念没有必然的联系，信号集指的是一个泛泛的概念，而未决集与信号掩码指的是具体的信号状态。

对于信号集的初始化有两种方法: 一种是用sigemptyset使信号集中不包含任何信号，然后用sigaddset把信号加入到信号集中去。
另一种是用sigfillset让信号集中包含所有信号，然后用sigdelset删除信号来初始化。
sigemptyset()函数初始化信号集set并将set设置为空。
sigfillset()函数初始化信号集，但将信号集set设置为所有信号的集合。
sigaddset()将信号signo加入到信号集中去。
sigdelset()从信号集中删除signo信号。
sigprocmask()将指定的信号集合加入到进程的信号阻塞集合中去。如果提供了oset,那么当前的信号阻塞集合将会保存到oset集全中去。
参数how决定了操作的方式:
SIG_BLOCK 增加一个信号集合到当前进程的阻塞集合中去
SIG_UNBLOCK 从当前的阻塞集合中删除一个信号集合
SIG_SETMASK 将当前的信号集合设置为信号阻塞集合


下面看一个例子:

    int main(){
        sigset_t initset;
        int i;
        sigemptyset(&initset);//初始化信号集合为空集合
        sigaddset(&initset,SIGINT);//将SIGINT信号加入到此集合中去
        while(1){
            sigprocmask(SIG_BLOCK,&initset,NULL);//将信号集合加入到进程的阻塞集合中去
            fprintf(stdout,"SIGINT singal blocked/n");
            for(i=0;i<10;i++){
            
                sleep(1);//每1秒输出
                fprintf(stdout,"block %d/n",i);
            }
            //在这时按一下Ctrl+C不会终止
            sigprocmask(SIG_UNBLOCK,&initset,NULL);//从进程的阻塞集合中去删除信号集合
            fprintf(stdout,"SIGINT SINGAL unblokced/n");
            for(i=0;i<10;i++){
                sleep(1);
                fprintf(stdout,"unblock %d/n",i);
            }
        }
        exit(0);
    }

执行结果:

    SIGINT singal blocked
    block 0
    block 1
    block 2
    block 3
    block 4
    block 5
    block 6
    block 7
    block 8
    block 9

在执行到block 3时按下了CTRL+C并不会终止，直到执行到block9后将集合从阻塞集合中移除。

    [root@localhost C]# ./s1
    SIGINT singal blocked
    block 0
    block 1
    block 2
    block 3
    block 4
    block 5
    block 6
    block 7
    block 8
    block 9
    SIGINT SINGAL unblokced
    unblock 0
    unblock 1

由于此时已经解除了阻塞，在unblock1后按下CTRL+C则立即终止。

## 2. 信号处理函数
sigaction

    int sigaction(
        int signo,
        const struct sigaction *act,
        struct sigaction *oldact
    );

这个函数主要是用于改变或检测信号的行为。
第一个参数是变更signo指定的信号，它可以指向任何值，SIGKILL,SIGSTOP除外
第二个参数,第三个参数是对信号进行细粒度的控制。
如果*act不为空，*oldact不为空，那么oldact将会存储信号以前的行为。如果act为空，*oldact不为空，那么oldact将会存储信号现在的行为。

    struct sigaction {
        void (*sa_handler)(int);
        void (*sa_sigaction)(int,siginfo_t*,void*);
        sigset_t sa_mask;
        int sa_flags;
        void (*sa_restorer)(void);
    }

参数含义:
sa_handler是一个函数指针，主要是表示接收到信号时所要采取的行动。此字段的值可以是SIG_DFL,SIG_IGN.分别代表默认操作与内核将忽略进程的信号。这个函数只传递一个参数那就是信号代码。
当SA_SIGINFO被设定在sa_flags中，那么则会使用sa_sigaction来指示信号处理函数，而非sa_handler.
sa_mask设置了掩码集，在程序执行期间会阻挡掩码集中的信号。
sa_flags设置了一些标志， SA_RESETHAND当该函数处理完成之后，设定为为系统默认的处理模式。SA_NODEFER 在处理函数中，如果再次到达此信号时，将不会阻塞。默认情况下，同一信号两次到达时，如果此时处于信号处理程序中，那么此信号将会阻塞。
SA_SIGINFO表示用sa_sigaction指示的函数。
sa_restorer已经被废弃。

sa_sigaction所指向的函数原型:

    void my_handler(int signo,siginfo_t *si,void *ucontext);

第一个参数: 信号编号
第二个参数:指向一个siginfo_t结构。
第三个参数是一个ucontext_t结构。
其中siginfo_t结构体中包含了大量的信号携带信息，可以看出，这个函数比sa_handler要强大，因为前者只能传递一个信号代码，而后者可以传递siginfo_t信息。

    typedef struct siginfo_t{
        int si_signo;//信号编号
        int si_errno;//如果为非零值则错误代码与之关联
        int si_code;//说明进程如何接收信号以及从何处收到
        pid_t si_pid;//适用于SIGCHLD，代表被终止进程的PID
        pid_t si_uid;//适用于SIGCHLD,代表被终止进程所拥有进程的UID
        int si_status;//适用于SIGCHLD，代表被终止进程的状态
        clock_t si_utime;//适用于SIGCHLD，代表被终止进程所消耗的用户时间
        clock_t si_stime;//适用于SIGCHLD，代表被终止进程所消耗系统的时间
        sigval_t si_value;
        int si_int;
        void * si_ptr;
        void* si_addr;
        int si_band;
        int si_fd;
    };


sigqueue

    sigqueue(pid_t pid,int signo,const union sigval value)

sigqueue函数类似于kill,也是一个进程向另外一个进程发送信号的。
但它比kill函数强大。
第一个参数指定目标进程的pid.
第二个参数是一个信号代码。
第三个参数是一个共用体，每次只能使用一个，用来进程发送信号传递的数据。
或者传递整形数据，或者是传递指针。
发送的数据被sa_sigaction所指示的函数的siginfo_t结构体中的si_ptr或者是si_int所接收。

sigpending

    sigpending(sigset_t set);

这个函数的作用是返回未决的信号到信号集set中。即未决信号集，未决信号集不仅包括被阻塞的信号，也可能包括已经到达但没有被处理的信号。

## 示例1: sigaction函数的用法

    void signal_set(struct sigaction *act)
    {
    switch(act->sa_flags){
        case (int)SIG_DFL:
            printf("using default hander/n");
            break;
        case (int)SIG_IGN:
            printf("ignore the signal/n");
            break;
        default:
            printf("%0x/n",act->sa_handler);
        }
    }
    void signal_set1(int x){//信号处理函数
        printf("xxxxx/n");
        while(1){}
    }
    
    int main(int argc,char** argv)
    {
        int i;
        struct sigaction act,oldact;
        act.sa_handler = signal_set1;
        act.sa_flags = SA_RESETHAND;
        //SA_RESETHANDD 在处理完信号之后，将信号恢复成默认处理
        //SA_NODEFER在信号处理程序执行期间仍然可以接收信号
        sigaction (SIGINT,&act,&oldact) ;//改变信号的处理模式
        for (i=1; i<12; i++)
        {
            printf("signal %d handler is : ",i);
            sigaction (i,NULL,&oldact) ;
            signal_set(&oldact);//如果act为NULL，oldact会存储信号当前的行为
            //act不为空，oldact不为空，则oldact会存储信号以前的处理模式
        }
        while(1){
            //等待信号的到来
        }
        return 0;
    }

运行结果:

    [root@localhost C]# ./s2
    signal 1 handler is : using default hander
    signal 2 handler is : 8048437
    signal 3 handler is : using default hander
    signal 4 handler is : using default hander
    signal 5 handler is : using default hander
    signal 6 handler is : using default hander
    signal 7 handler is : using default hander
    signal 8 handler is : using default hander
    signal 9 handler is : using default hander
    signal 10 handler is : using default hander
    signal 11 handler is : using default hander
    xxxxx

解释:

    sigaction(i,NULL,&oldact);
    signal_set(&oldact);

由于act为NULL,那么oldact保存的是当前信号的行为，当前的第二个信号的行为是执行自定义的处理程序。
当按下CTRL＋C时会执行信号处理程序，输出xxxxxx，再按一下CTRL＋C会停止,是由于SA_RESETHAND恢复成默认的处理模式，即终止程序。
如果没有设置SA_NODEFER,那么在处理函数执行过程中按一下CTRL＋C将会被阻塞，那么程序会停在那里。

## 示例2: sigqueue向本进程发送数据的信号

    int main(){
        union sigval val;//定义一个携带数据的共用体
        struct sigaction oldact,act;
        act.sa_sigaction=myhandler;
        act.sa_flags=SA_SIGINFO;//表示使用sa_sigaction指示的函数，处理完恢复默认，不阻塞处理过程中到达下在被处理的信号
        //注册信号处理函数
        sigaction(SIGUSR1,&act,&oldact);
        char data[100];
        int num=0;
        while(num<10){
            sleep(2);
            printf("等待SIGUSR1信号的到来/n");
            sprintf(data,"%d",num++);
            val.sival_ptr=data;
            sigqueue(getpid(),SIGUSR1,val);//向本进程发送一个信号
        }
    }
    
    void myhandler(int signo,siginfo_t *si,void *ucontext){
        printf("已经收到SIGUSR1信号/n");
        printf("%s/n",(char*)(si->si_ptr));
    }

程序执行的结果是:

    等待SIGUSR1信号的到来
    已经收到SIGUSR1信号
    0
    等待SIGUSR1信号的到来
    已经收到SIGUSR1信号
    1
    等待SIGUSR1信号的到来
    已经收到SIGUSR1信号
    2
    等待SIGUSR1信号的到来
    已经收到SIGUSR1信号
    3
    等待SIGUSR1信号的到来
    已经收到SIGUSR1信号
    4
    等待SIGUSR1信号的到来
    已经收到SIGUSR1信号
    5
    等待SIGUSR1信号的到来
    已经收到SIGUSR1信号
    6
    等待SIGUSR1信号的到来
    已经收到SIGUSR1信号
    7
    等待SIGUSR1信号的到来
    已经收到SIGUSR1信号
    8
    等待SIGUSR1信号的到来
    已经收到SIGUSR1信号
    9

解释: 本程序用sigqueue不停的向自身发送信号,并且携带数据，数据被放到处理函数的第二个参数siginfo_t结构体中的si_ptr指针，当num<10时不再发。

一般而言，sigqueue与sigaction配合使用，而kill与signal配合使用。

## 示例3: 一个进程向另外一个进程发送信号，并携带信息

发送端:

    int main(){
        union sigval value;
        value.sival_int=10;
        
        if(sigqueue(4403,SIGUSR1,value)==-1){//4403是目标进程pid
            perror("信号发送失败/n");
        }
        sleep(2);
    }

接收端:

    int main(){
        struct sigaction oldact,act;
        act.sa_sigaction=myhandler;
        act.sa_flags=SA_SIGINFO|SA_NODEFER;
        //表示执行后恢复，用sa_sigaction指示的处理函数，在执行期间仍然可以接收信号
        sigaction(SIGUSR1,&act,&oldact);
        while(1){
            sleep(2);
            printf("等待信号的到来/n");
        }
    }
    
    void myhandler(int signo,siginfo_t *si,void *ucontext){
        printf("the value is %d/n",si->si_int);
    }

## 示例4: sigpending的用法
sigpending(sigset_t *set)将未决信号放到指定的set信号集中去，未决信号包括被阻塞的信号和信号到达时但还没来得及处理的信号

    int main(){
        struct sigaction oldact,act;
        sigset_t oldmask,newmask,pendingmask;
        act.sa_sigaction=myhandler;
        act.sa_flags=SA_SIGINFO;
        sigemptyset(&act.sa_mask);//首先将阻塞集合设置为空，即不阻塞任何信号
        //注册信号处理函数
        sigaction(SIGRTMIN+10,&act,&oldact);
        //开始阻塞
        sigemptyset(&newmask);
        sigaddset(&newmask,SIGRTMIN+10);
        printf("SIGRTMIN+10 blocked/n");
        sigprocmask(SIG_BLOCK,&newmask,&oldmask);
        sleep(20);//为了发出信号
        printf("now begin to get pending mask/n");
        if(sigpending(&pendingmask)<0){
            perror("pendingmask error");
        }
        if(sigismember(&pendingmask,SIGRTMIN+10)){
            printf("SIGRTMIN+10 is in the pending mask/n");
        }
        
        sigprocmask(SIG_UNBLOCK,&newmask,&oldmask);
        printf("SIGRTMIN+10 unblocked/n");
    }
    //信号处理函数
    void myhandler(int signo,siginfo_t *si,void *ucontext){
        printf("receive signal %d/n",si->si_signo);
    }

程序执行,在另一个shell发送信号:

     kill -44 4579
    
    SIGRTMIN+10 blocked
    now begin to get pending mask
    SIGRTMIN+10 is in the pending mask
    receive signal 44
    SIGRTMIN+10 unblocked

可以看到SIGRTMIN由于被阻塞所以处于未决信号集中。
关于基本的信号处理函数就介绍到这了。

# 第四部分: 保护临界区不被中断
## 1. 函数的可重入性

函数的可重入性是指可以多于一个任务并发使用函数，而不必担心数据错误。相反，不可重入性是指不能多于一个任务共享函数，除非能保持函数互斥(或者使用信号量，或者在代码的关键部分禁用中断)。可重入函数可以在任意时刻被中断，稍后继续执行，而不会丢失数据。

可重入函数：
* 不为连续的调用持有静态数据。
* 不返回指向静态数据的指针；所有数据都由函数的调用者提供。
* 使用本地数据，或者通过制作全局数据的本地拷贝来保护全局数据。
* 绝不调用任何不可重入函数。

不可重入函数可能导致混乱现象，如果当前进程的操作与信号处理程序同时对一个文件进行写操作或者是调用malloc()，那么就可能出现混乱，当从信号处理程序返回时，造成了状态不一致。从而引发错误。
因此，信号的处理必须是可重入函数。
简单的说，可重入函数是指在一个程序中调用了此函数，在信号处理程序中又调用了此函数，但仍然能够得到正确的结果。
printf，malloc函数都是不可重入函数。printf函数如果打印缓冲区一半时，又有一个printf函数，那么此时会造成混乱。而malloc函数使用了系统全局内存分配表。

## 2. 保护临界区不被中断

由于临界区的代码是关键代码，是非常重要的部分，因此，有必要对临界区进行保护，不希望信号来中断临界区操作。这里通过信号屏蔽字来阻塞信号的发生。

 下面介绍两个与保护临界区不被信号中断的相关函数。

    int pause(void);
    int sigsuspend(const sigset_t *sigmask);

pause函数挂起一个进程，直到一个信号发生。

sigsuspend函数的执行过程如下:
(1)设置新的mask去阻塞当前进程
(2)收到信号，调用信号的处理函数
(3)将mask设置为原先的掩码
(4)sigsuspend函数返回

可以看出，sigsuspend函数是等待一个信号发生，当等待的信号发生时，执行完信号处理函数后就会返回。它是一个原子操作。

保护临界区的中断:
(1)首先用sigprocmask去阻塞信号
(2)执行后关键代码后,用sigsuspend去捕获信号
(3)然后sigprocmask去除阻塞
这样信号就不会丢失了，而且不会中断临界区。

上面的程序是用pause去保护临界区，首先用sigprocmask去阻塞SIGINT信号，执行临界区代码，然后解除阻塞。最后调用pause()函数等待信号的发生。但此时会产生一个问题，如果信号在解除阻塞与pause之间发生的话，信号就可能丢失。这将是一个不可靠的信号机制。
因此，采用sigsuspend可以避免上述情况发生。

sigsuspend函数的用法：
sigsuspend函数是等待的信号发生时才会返回。
sigsuspend函数遇到结束时不会返回，这一点很重要。

示例:

下面的例子能够处理信号SIGUSR1,SIGUSR2,SIGSEGV,其它的信号被屏蔽，该程序输出对应的信号，然后继续等待其它信号的出现。

    void myhandler(int signo);
    int main(){
        struct sigaction action;
        sigset_t sigmask;
        sigemptyset(&sigmask);
        sigaddset(&sigmask,SIGUSR1);
        sigaddset(&sigmask,SIGUSR2);
        sigaddset(&sigmask,SIGSEGV);
        action.sa_handler=myhandler;
        action.sa_mask=sigmask;
        sigaction(SIGUSR1,&action,NULL);
        sigaction(SIGUSR2,&action,NULL);
        sigaction(SIGSEGV,&action,NULL);
        sigfillset(&sigmask);
        sigdelset(&sigmask,SIGUSR1);
        sigdelset(&sigmask,SIGUSR2);
        sigdelset(&sigmask,SIGSEGV);
        while(1){
            sigsuspend(&sigmask);//不断的等待信号到来
        }
        return 0;
    }
        
    void myhandler(int signo){
        switch(signo){
            case SIGUSR1:
                printf("received sigusr1 signal./n");
            break ;
            case SIGUSR2:
                printf("received sigusr2 signal./n");
            break;
            case SIGSEGV:
                printf("received sigsegv signal/n");
            break;
        }
    }

程序运行结果:

    received sigusr1 signal
    received sigusr2 signal
    received sigsegv signal
    received sigusr1 signal
    已终止

另一个终端用于发送信号:
先得到当前进程的pid, ps aux|grep 程序名

    kill -SIGUSR1 4901
    kill -SIGUSR2 4901
    kill -SIGSEGV 4901
    kill -SIGTERM 4901
    kill -SIGUSR1  4901

解释:
第一行发送SIGUSR1，则调用信号处理函数，打印出结果。
第二，第三行分别打印对应的结果。
第四行发送一个默认处理为终止进程的信号。
但此时，但不会终止程序，由于sigsuspend遇到终止进程信号并不会返回，此时并不会打印出"已终止"，这个信号被阻塞了。当再次发送SIGURS1信号时，进程的信号阻塞恢复成默认的值，因此，此时将会解除阻塞SIGTERM信号，所以进程被终止。

# 第五部分: 信号的继承与执行
当使用fork()函数时，子进程会继承父进程完全相同的信号语义，这也是有道理的，因为父子进程共享一个地址空间，所以父进程的信号处理程序也存在于子进程中。


示例: 子进程继承父进程的信号处理函数

    void myhandler(int signo,siginfo_t *si,void *vcontext);
    int main(){
        union sigval val;
        struct sigaction oldact,newact;
        newact.sa_sigaction=myhandler;
        newact.sa_flags=SA_SIGINFO|SA_RESETHAND;//表示采用sa_sigaction指示的函数以及执行完处理函数后恢复默认操作
        //注册信号处理函数
        sigaction(SIGUSR1,&newact,&oldact);
        
        if(fork()==0){
            val.sival_int=10;
            printf("子进程/n");
            sigqueue(getpid(),SIGUSR1,val);
        }
        else {
            val.sival_int=20;
            printf("父进程/n");
            sigqueue(getpid(),SIGUSR1,val);
        }
    }
    
    void myhandler(int signo,siginfo_t *si,void *vcontext){
        printf("信号处理/n");
        printf("%d/n",(si->si_int));
    }

输出的结果为:

    子进程
    信号处理
    10
    父进程
    信号处理
    20

可以看出来，子进程继承了父进程的信号处理函数。

# 第六部分: 实时信号中锁的研究

 

## 1. 信号处理函数与主函数之间的死锁

当主函数访问临界资源时，通常需要加锁，如果主函数在访问临界区时，给临界资源上锁，此时发生了一个信号，那么转入信号处理函数，如果此时信号处理函数也对临界资源进行访问，那么信号处理函数也会加锁，由于主程序持有锁，信号处理程序等待主程序释放锁。又因为信号处理函数已经抢占了主函数，因此，主函数在信号处理函数结束之前不能运行。因此，必然造成死锁。

示例1: 主函数与信号处理函数之间的死锁

    int value=0;
    sem_t sem_lock;//定义信号量
    void myhandler(int signo,siginfo_t *si,void *vcontext);//进程处理函数声明
    int main(){
        union sigval val;
        val.sival_int=1;
        struct sigaction oldact,newact;
        int res;
        res=sem_init(&sem_lock,0,1);
        if(res!=0){
            perror("信号量初始化失败");
        }
        
        newact.sa_sigaction=myhandler;
        newact.sa_flags=SA_SIGINFO;
        sigaction(SIGUSR1,&newact,&oldact);
        sem_wait(&sem_lock);
        printf("xxxx/n");
        value=1;
        sleep(10);
        sigqueue(getpid(),SIGUSR1,val);//sigqueue发送带参数的信号
        sem_post(&sem_lock);
        sleep(10);
        exit(0);
    }
    
    void myhandler(int signo,siginfo_t *si,void *vcontext){
        sem_wait(&sem_lock);
        value=0;
        sem_post(&sem_lock);
    }

此程序将一直阻塞在信号处理函数的sem_wait函数处。

## 2. 利用测试锁解决死锁
sem_trywait(&sem_lock);是非阻塞的sem_wait,如果加锁失败或者是超时，则返回－1。
示例2: 用sem_trywait来解决死锁

    int value=0;
    sem_t sem_lock;//定义信号量
    void myhandler(int signo,siginfo_t *si,void *vcontext);//进程处理函数声明
    int main(){
        union sigval val;
        val.sival_int=1;
        struct sigaction oldact,newact;
        int res;
        res=sem_init(&sem_lock,0,1);
        if(res!=0){
            perror("信号量初始化失败");
        }
        
        newact.sa_sigaction=myhandler;
        newact.sa_flags=SA_SIGINFO;
        sigaction(SIGUSR1,&newact,&oldact);
        sem_wait(&sem_lock);
        printf("xxxx/n");
        value=1;
        sleep(10);
        sigqueue(getpid(),SIGUSR1,val);//sigqueue发送带参数的信号
        sem_post(&sem_lock);
        sleep(10);
        sigqueue(getpid(),SIGUSR1,val);
        exit(0);
    }
    
    void myhandler(int signo,siginfo_t *si,void *vcontext){
        if(sem_trywait(&sem_lock)==0){
            value=0;
            sem_post(&sem_lock);
        }
    }

第一次发送sigqueue时，由于主函数持有锁，因此，sem_trywait返回－1，当第二次发送sigqueue时，主函数已经释放锁，此时就可以在信号处理函数中对临界资源加锁了。
但这种方法明显丢失了一个信号，不是很好的解决方法。

## 3. 利用双线程来解决主函数与信号处理函数死锁
我们知道，当进程收到一个信号时，会选择其中的某个线程进行处理，前提是这个线程没有屏蔽此信号。因此，可以在主线程中屏蔽信号，另选一个线程去处理这个信号。由于主线程与另外一个线程是平行执行的，因此，等待主线程执行完临界区时，释放锁，这个线程去执行信号处理函数，直到执行完毕释放临界资源。


这里用到一个线程的信号处理函数: pthread_sigmask

    int pthread_sigmask(int how,const sigset_t *set,sigset_t *oldset);

这个函数与sigprocmask很相似。
how的取值:
SIG_BLOCK 将信号集加入到线程的阻塞集中去
SIG_UNBLOCK 将信号集从阻塞集中删除
SIG_SETMASK 将当前集合设置为线程的阻塞集

示例: 利用双线程来解决主函数与信号处理函数之间的死锁

    void*thread_function(void *arg);//线程处理函数
    void myhandler(int signo,siginfo_t *si,void *vcontext);//信号处理函数
    int value;
    sem_t semlock;
    int main(){
        int res;
        pthread_t mythread;
        void *thread_result;
        res=pthread_create(&mythread,NULL,thread_function,NULL);//创建一个子线程
        if(res!=0){
            perror("线程创建失败");
        }
    
        //在主线程中将信号屏蔽
        sigset_t empty;
        sigemptyset(&empty);
        sigaddset(&empty,SIGUSR1);
        pthread_sigmask(SIG_BLOCK,&empty,NULL);
    
        //主线程中对临界资源的访问
        if(sem_init(&semlock,0,1)!=0){
            perror("信号量创建失败");
        }
        sem_wait(&semlock);
        printf("主线程已经执行/n");
        value=1;
        sleep(10);
        sem_post(&semlock);
        res=pthread_join(mythread,&thread_result);//等待子线程退出
        exit(EXIT_SUCCESS);
    }
    
    void *thread_function(void *arg){
        struct sigaction oldact,newact;
        newact.sa_sigaction=myhandler;
        newact.sa_flags=SA_SIGINFO;
        //注册信号处理函数
        sigaction(SIGUSR1,&newact,&oldact);
        union sigval val;
        val.sival_int=1;
        printf("子线程睡眠3秒/n");
        sleep(3);
        sigqueue(getpid(),SIGUSR1,val);
        pthread_exit(0);//线程结束
    }
    
    void myhandler(int signo,siginfo_t *si,void *vcontext){
        sem_wait(&semlock);
        value=0;
        printf("信号处理完毕/n");
        sem_post(&semlock);
    }

运行结果如下:

    主线程已经执行
    子线程睡眠3秒
    信号处理完毕

解释一下:
在主线线程中阻塞了SIGUSR1信号,首先让子线程睡眠3秒，目的让主线程先运行，然后当主线程访问临界资源时，让线程sleep(10),在这期间，子线程发送信号，此时子线程会去处理信号，而主线程依旧平行的运行，子线程被阻止信号处理函数的sem_wait处，等待主线程10后，信号处理函数得到锁，然后进行临界资源的访问。这就解决了主函数与信号处理函数之间的死锁问题。

扩展: 如果有多个信号到达时，还可以用多线程来处理多个信号，从而达到并行的目的，这个很好实现的，可以尝试一下。
