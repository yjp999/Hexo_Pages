---
title: 基于预线程化的并发服务器
date: 2016-09-27 20:08:29
tags: [并发编程,网络编程]
categories: [编程]
---

## Background
今年暑假读CSAPP第12章并发编程时看到的一个demo

这个demo不但涉及到了服务器的开发，也包括了并发编程和网络编程相关知识

## Preface
一个服务器必须能够同时服务两个或两个以上的客户端，为了满足这一要求，有以下几种方案

* 父进程接受监听描述符，子进程处理连接描述符
    *  父进程accept以后fork一个子进程，父进程接着accept
    *  子进程负责处理这个fd(file descriptor)
    *  代码实现在[这里](http://www.martinbroadhurst.com/source/forked-server.c.html)
* 通过I/O多路复用技术实现并发
    *   网络客户端发起连接请求
    *   等待数据到达服务器
    *   代码实现在[这里](http://csapp.cs.cmu.edu/2e/ics2/code/conc/select.chttp://csap.cs.cmu.edu/2e/ics2/code/conc/select.c)
* 基于多线程实现并发服务器

以上3个方案从理论上来说都可以满足一个服务器同时服务多个客户端的需求，但是从工业的角度来讲，都不大现实。拿第一个方案来说，不现实的三个原因如下：

* 每来一个连接就fork一个子进程，开销会很大，在linux系统下，调用fork不会发生地址空间COW(copy on write)，但是会复制父进程的页表。
* 进程调度器压力过大。当并发量很大时，系统里有很多的进程，那么绝大部分时间将会花在决定哪一个是下一个运行进程以及上下文切换。
* 内存的消耗。父子进程之间需要IPC，而高并发下的IPC所带来的overhead也不可忽略。

将进程换成线程，虽然可以解决fork产生的问题，但是依旧无法处理调度压力和内存开销的麻烦。然而采用固定数量的线程（线程池）是一个非常不错的选择，这就是下面要提到的“预线程并发”模型。

## Body
我们使用下图所示的生产者-消费者模型来来降低为每一个新客户端创建一个线程而导致的开销。服务器是由`一个主线程`和`一组工作线程`构成的。

* 主线程不断地接受来自客户端的连接请求，并将得到的连接描述符放在一个`有限缓冲区`中
* 每一个工作者线程反复地从共享缓冲区中取出描述符来为客户端服务，然后等待下一个fd

{% asset_img prethreaded_organization.png  The Architecture %}

为了实现这一目标，我们首先需要开发一个SBUF包，它是用来构造生产者-消费者程序的。SBUF操作类型为sbuf_t的有限缓冲区，SBUF包的头文件如下(sbuf.h)

``` C
#ifndef __SBUF_H__
#define __SBUF_H__

#include "csapp.h"

/* $begin sbuft */
typedef struct {
    int *buf;          /* Buffer array */         
    int n;             /* Maximum number of slots */
    int front;         /* buf[(front+1)%n] is first item */
    int rear;          /* buf[rear%n] is last item */
    sem_t mutex;       /* Protects accesses to buf */
    sem_t slots;       /* Counts available slots */
    sem_t items;       /* Counts available items */
} sbuf_t;
/* $end sbuft */

void sbuf_init(sbuf_t *sp, int n);
void sbuf_deinit(sbuf_t *sp);
void sbuf_insert(sbuf_t *sp, int item);
int sbuf_remove(sbuf_t *sp);

#endif /* __SBUF_H__ */
```
项目存放在一个动态分配的n项整数数组中，front和rear索引值记录该数组中的第一项和最后一项。三个信号量同步对缓冲区的访问，mutex信号量提供互斥的缓冲区访问，slots和items信号量分别记录空槽位和可用项目的数量。

demo的全部代码在[这里](http://csapp.cs.cmu.edu/2e/ics2/code/conc/echoservert_pre.c)。

## Conclusion
虽说这个demo与之前三种方案相比较已经有了很大的提升，但是在工业中依旧不可使用，因为一个完整的服务器所需要考虑的内容远远不止这些，有兴趣的可以去读读Nginx的代码。其实编写事件驱动程序并不只有I/O多路复用这一种方法，上面这个小demo实际上也是一个事件驱动服务器，带有主线程和工作者线程的简单状态机。

*   主线程有两种状态（“等待连接请求”和“等待可用的缓冲区槽位”）
*   两个I/O事件（“连接请求到达”和“缓冲区槽位变为可用”）
*   两个转换（“接受连接请求”和“插入缓冲区项目”）
*   每个工作者线程有一个状态（“等待可用的缓冲项目”），一个I/O事件（“缓冲区项目可用”）和一个转换（“取出缓冲区项目”）

## Future
关于CS专业，看完书如果不动手其实很难真正理解或者容易遗忘，所以我打算在后续阶段开发一个toyd（玩具式服务器），相对比较完整，支持静态/动态内容，服务器架构采用线程池以及事件驱动和非阻塞I/O。以此来学以致用~

