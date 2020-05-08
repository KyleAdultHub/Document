---
title: golang 常识
date: "2019-1-5 10:45:00"
categories:
- Golang
- Golang特性
tags:
- Golang
toc: true
typora-root-url: ..\..\..
---

###  golang  基础常识   

#### recover能处理所有的异常吗

recover能处理程序主动触发的panic以及空指针访问、异常地址访问等错误，因此可以认为是能处理所有异常了。

#### golang中常量是怎么实现的

从汇编中看是对字符串常量加了一个标号，同时设置为SRODATA，也就是只读，对数字常量直接在代码中作为立即数使用了

#### golang的make和new的区别是什么
 new有点像c++里面的new,用来初始化各种type，然后返回其指针。 只不过由于没有构造函数的存在，所以全部用零值来填充，比较特殊的是slice,map,channel， 它们的零值都是nil。另外由于golang直接可以用&struct{} 形式来初始化，所以平时用到new的机会也比较少。
make是用来初始化map,slice,以及channel的，并且可以指定初始长度， 它返回的不是指针，而是对象本身。另外，make出来的map,slice,channel都是可以直接使用的。

#### golang 的channel是怎么实现的
 golang的channel是个结构体，里面大概包含了三大部分：
 a. 指向内容的环形缓存区，及其相关游标
 b. 读取和写入的排队goroutine链表
 c. 锁
 任何操作前都需要获得锁， 当写满或者读空的时候，就将当前goroutine加入到recvq或者sendq中， 并出让cpu(gopark)。

#### golang 的GC算法

golang  采用的是三色标记法

#### 标记-清除算法：

对象只有黑白两色

1. stop the world,即停止所有goroutine
2. 从根对象（全局指针和栈上的对象）出发，把所有能直接或间接访问到的对象标记为黑色，其它所有对象标志为白色
3. 清除所有白色对象
4. start the world

#### 三色标记法：

对象有黑白灰三色

1. stop the world
2. 将根对象全部标记为灰色
3. start the world
4. 在goroutine中进行对灰色对象进行遍历， 将灰色对象引用的每个对象标记为灰色，然后将该灰色对象标记为黑色。
5. 重复执行4， 直接将所有灰色对象都变成黑色对象。
6. stop the world，清除所有白色对象

这里4，5是与用户程序是并发执行的，所以stw的时间被大大缩短了。 不过这样做可能会导致新创建的对象被误清除，因此使用了写屏障技术来解决该问题，大体逻辑是当创建新对象时将新对象置为灰色。

#### Golang中除了加Mutex锁以外还有哪些方式安全读写共享变量？

Golang中Goroutine 可以通过 Channel 进行安全读写共享变量。

####  go语言的并发机制以及它所使用的CSP并发模型．

Golang的CSP并发模型，是通过Goroutine和Channel来实现的。

Goroutine 是Go语言中并发的执行单位。有点抽象，其实就是和传统概念上的”线程“类似，可以理解为”线程“。 Channel是Go语言中各个并发结构体(Goroutine)之前的通信机制。通常Channel，是各个Goroutine之间通信的”管道“，有点类似于Linux中的管道。

通信机制channel也很方便，传数据用channel <- data，取数据用<-channel。

在通信过程中，传数据channel <- data和取数据<-channel必然会成对出现，因为这边传，那边取，两个goroutine之间才会实现通信。

而且不管传还是取，必阻塞，直到另外的goroutine传或者取为止。

#### Golang 中常用的并发模型？

- 通过channel通知实现并发控制
- 通过sync包中的WaitGroup实现并发控制
- 在Go 1.7 以后引进的强大的Context上下文，实现并发控制

#### 协程，线程，进程的区别。

- 进程

进程是具有一定独立功能的程序关于某个数据集合上的一次运行活动,进程是系统进行资源分配和调度的一个独立单位。每个进程都有自己的独立内存空间，不同进程通过进程间通信来通信。由于进程比较重量，占据独立的内存，所以上下文进程间的切换开销（栈、寄存器、虚拟内存、文件句柄等）比较大，但相对比较稳定安全。

- 线程

线程是进程的一个实体,是CPU调度和分派的基本单位,它是比进程更小的能独立运行的基本单位.线程自己基本上不拥有系统资源,只拥有一点在运行中必不可少的资源(如程序计数器,一组寄存器和栈),但是它可与同属一个进程的其他的线程共享进程所拥有的全部资源。线程间通信主要通过共享内存，上下文切换很快，资源开销较少，但相比进程不够稳定容易丢失数据。

- 协程

协程是一种用户态的轻量级线程，协程的调度完全由用户控制。协程拥有自己的寄存器上下文和栈。协程调度切换时，将寄存器上下文和栈保存到其他地方，在切回来的时候，恢复先前保存的寄存器上下文和栈，直接操作栈则基本没有内核切换的开销，可以不加锁的访问全局变量，所以上下文的切换非常快。

#### 10. 什么是channel，为什么它可以做到线程安全？

Channel是Go中的一个核心类型，可以把它看成一个管道，通过它并发核心单元就可以发送或者接收数据进行通讯(communication),Channel也可以理解是一个先进先出的队列，通过管道进行通信。

Golang的Channel,发送一个数据到Channel 和 从Channel接收一个数据 都是 原子性的。而且Go的设计思想就是:不要通过共享内存来通信，而是通过通信来共享内存，前者就是传统的加锁，后者就是Channel。也就是说，设计Channel的主要目的就是在多任务间传递数据的，这当然是安全的。

#### 11. Epoll原理.

开发高性能网络程序时，windows开发者们言必称Iocp，linux开发者们则言必称Epoll。大家都明白Epoll是一种IO多路复用技术，可以非常高效的处理数以百万计的Socket句柄，比起以前的Select和Poll效率提高了很多。

先简单了解下如何使用C库封装的3个epoll系统调用。

```
int epoll_create(int size);  
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  
int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);  
```

使用起来很清晰，首先要调用`epoll_create`建立一个epoll对象。参数size是内核保证能够正确处理的最大句柄数，多于这个最大数时内核可不保证效果。 epoll_ctl可以操作上面建立的epoll，例如，将刚建立的`socket`加入到epoll中让其监控，或者把 epoll正在监控的某个socket句柄移出epoll，不再监控它等等。

`epoll_wait`在调用时，在给定的timeout时间内，当在监控的所有句柄中有事件发生时，就返回用户态的进程。

从调用方式就可以看到epoll相比select/poll的优越之处是,因为后者每次调用时都要传递你所要监控的所有socket给select/poll系统调用，这意味着需要将用户态的socket列表copy到内核态，如果以万计的句柄会导致每次都要copy几十几百KB的内存到内核态，非常低效。而我们调用`epoll_wait`时就相当于以往调用select/poll，但是这时却不用传递socket句柄给内核，因为内核已经在epoll_ctl中拿到了要监控的句柄列表。

所以，实际上在你调用`epoll_create`后，内核就已经在内核态开始准备帮你存储要监控的句柄了，每次调用`epoll_ctl`只是在往内核的数据结构里塞入新的socket句柄。

在内核里，一切皆文件。所以，epoll向内核注册了一个文件系统，用于存储上述的被监控socket。当你调用epoll_create时，就会在这个虚拟的epoll文件系统里创建一个file结点。当然这个file不是普通文件，它只服务于epoll。

epoll在被内核初始化时（操作系统启动），同时会开辟出epoll自己的内核高速cache区，用于安置每一个我们想监控的socket，这些socket会以红黑树的形式保存在内核cache里，以支持快速的查找、插入、删除。这个内核高速cache区，就是建立连续的物理内存页，然后在之上建立slab层，通常来讲，就是物理上分配好你想要的size的内存对象，每次使用时都是使用空闲的已分配好的对象。

```
static int __init eventpoll_init(void)  {  
    ... ...  
  
    /* Allocates slab cache used to allocate "struct epitem" items */  
    epi_cache = kmem_cache_create("eventpoll_epi", sizeof(struct epitem),  
            0, SLAB_HWCACHE_ALIGN|EPI_SLAB_DEBUG|SLAB_PANIC,  
            NULL, NULL);  
  
    /* Allocates slab cache used to allocate "struct eppoll_entry" */  
    pwq_cache = kmem_cache_create("eventpoll_pwq",  
            sizeof(struct eppoll_entry), 0,  
            EPI_SLAB_DEBUG|SLAB_PANIC, NULL, NULL);  
 ... ...  
 }
```

epoll的高效就在于，当我们调用`epoll_ctl`往里塞入百万个句柄时，`epoll_wait`仍然可以飞快的返回，并有效的将发生事件的句柄给我们用户。这是由于我们在调用`epoll_create`时，内核除了帮我们在epoll文件系统里建了个file结点，在内核cache里建了个红黑树用于存储以后epoll_ctl传来的socket外，还会再建立一个list链表，用于存储准备就绪的事件，当epoll_wait调用时，仅仅观察这个list链表里有没有数据即可。有数据就返回，没有数据就sleep，等到timeout时间到后即使链表没数据也返回。所以，epoll_wait非常高效。

而且，通常情况下即使我们要监控百万计的句柄，大多一次也只返回很少量的准备就绪句柄而已，所以，epoll_wait仅需要从内核态copy少量的句柄到用户态而已，因此就会非常的高效！

然而,这个准备就绪list链表是怎么维护的呢？当我们执行epoll_ctl时，除了把socket放到epoll文件系统里file对象对应的红黑树上之外，还会给内核中断处理程序注册一个回调函数，告诉内核，如果这个句柄的中断到了，就把它放到准备就绪list链表里。所以，当一个socket上有数据到了，内核在把网卡上的数据copy到内核中后就来把socket插入到准备就绪链表里了。

如此，一个红黑树，一张准备就绪句柄链表，少量的内核cache，就帮我们解决了大并发下的socket处理问题。执行`epoll_create`时，创建了红黑树和就绪链表，执行epoll_ctl时，如果增加socket句柄，则检查在红黑树中是否存在，存在立即返回，不存在则添加到树干上，然后向内核注册回调函数，用于当中断事件来临时向准备就绪链表中插入数据。执行epoll_wait时立刻返回准备就绪链表里的数据即可。

最后看看epoll独有的两种模式LT和ET。无论是LT和ET模式，都适用于以上所说的流程。区别是，LT模式下，只要一个句柄上的事件一次没有处理完，会在以后调用epoll_wait时每次返回这个句柄，而ET模式仅在第一次返回。

当一个socket句柄上有事件时，内核会把该句柄插入上面所说的准备就绪list链表，这时我们调用`epoll_wait`，会把准备就绪的socket拷贝到用户态内存，然后清空准备就绪list链表，最后，`epoll_wait`需要做的事情，就是检查这些socket，如果不是ET模式（就是LT模式的句柄了），并且这些socket上确实有未处理的事件时，又把该句柄放回到刚刚清空的准备就绪链表了。所以，非ET的句柄，只要它上面还有事件，epoll_wait每次都会返回。而ET模式的句柄，除非有新中断到，即使socket上的事件没有处理完，也是不会每次从epoll_wait返回的。

因此epoll比select的提高实际上是一个用空间换时间思想的具体应用.对比阻塞IO的处理模型, 可以看到采用了多路复用IO之后, 程序可以自由的进行自己除了IO操作之外的工作, 只有到IO状态发生变化的时候由多路复用IO进行通知, 然后再采取相应的操作, 而不用一直阻塞等待IO状态发生变化,提高效率.