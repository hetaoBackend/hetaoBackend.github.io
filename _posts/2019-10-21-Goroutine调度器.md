---
layout:     post
title:      Goroutine调度器
subtitle:   Goroutine调度器架构
date:       2019-10-21
author:     Walnut
header-img: img/post-bg-keyboard.jpg
catalog: true
tags:
    - Go
---

# Go 调度器：M，P和G

## 基础知识
Go的运行时管理帧调度、垃圾回收以及goroutine的运行环境。
运行时负责运行goroutine并把它们映射到操作系统的线程上。goroutine比线程还轻量，启动的时候花费很少。每个goroutine都是由一个G结构来表示，这个结构体的字段用来跟踪此goroutine的栈和状态，所以`G = goroutine`。

运行时管理着G并把它们映射到Logical Processor，P可以看作一个抽象的资源或者一个上下文，他需要获取以便操作系统线程（称之为M）可以运行G。

通过`runtime.GOMAXPROCS(numLogicalProcessors)`可以控制多少P可以获取。如果你需要调整这个参数，只设置一次，因为他需要STW。

本质上，操作系统运行线程，线程运行你的代码。Go的技巧是编译器会在Go运行时的一些地方插入系统调用，（比如通过channel发送值，调用runtime包等），所以Go可以通知调度器质性特定的操作。

  
![](https://colobu.com/2017/05/04/go-scheduler/go-sched.png)

Go运行时存在两种类型的queue，一种是一个全局的queue，一种是每个P都维护自己的G的queue。

为了运行goroutine，M需要持有上下文P。M会从P的queue弹出一个goroutine并执行。

当你创建一个新的goroutine的时候，它会被放入P的queue。当然还有一个work-stealing调度算法，当M执行了一些G后，如果它的queue为空，它会随机的选择另外一个P，从它的queue中取到一半的G到自己的queue中执行。

当你的goroutine执行阻塞的系统调用的时候（syscall），阻塞的系统调用会中断，如果当前有一些G在执行，运行时会把这个线程从P中摘除，然后再创建一个新的操作系统线程（如果没有空闲的线程可用的话）来服务于这个P。

当系统调用继续的时候，这个goroutine被放入到本地运行queue，线程会park它自己（休眠），加入到空闲线程中。


-   G: 表示goroutine，存储了goroutine的执行stack信息、goroutine状态以及goroutine的任务函数等；另外G对象是可以重用的。
-   P: 表示逻辑processor，P的数量决定了系统内最大可并行的G的数量（前提：系统的物理cpu核数>=P的数量）；P的最大作用还是其拥有的各种G对象队列、链表、一些cache和状态。
-   M: M代表着真正的执行计算资源。在绑定有效的p后，进入schedule循环；而schedule循环的机制大致是从各种队列、p的本地队列中获取G，切换到G的执行栈上并执行G的函数，调用goexit做清理工作并回到m，如此反复。M并不保留G状态，这是G可以跨M调度的基础。

## sysmon

和操作系统按时间片调度线程不同，Go并没有时间片的概念。如果某个G没有进行system call调用、没有进行I/O操作、没有阻塞在一个channel操作上，那么**m是如何让G停下来并调度下一个runnable G的呢**？答案是：G是被抢占调度的。

前面说过，除非极端的无限循环或死循环，否则只要G调用函数，Go runtime就有抢占G的机会。Go程序启动时，runtime会去启动一个名为sysmon的m(一般称为监控线程)，该m无需绑定p即可运行，该m在整个Go程序的运行过程中至关重要：

sysmon每20us~10ms启动一次，按照《Go语言学习笔记》中的总结，sysmon主要完成如下工作：

-   释放闲置超过5分钟的span物理内存；
-   如果超过2分钟没有垃圾回收，强制执行；
-   将长时间未处理的netpoll结果添加到任务队列；
-   向长时间运行的G任务发出抢占调度；
-   收回因syscall长时间阻塞的P；

我们看到sysmon将“向长时间运行的G任务发出抢占调度”，这个事情由retake实施：

如果一个G任务运行10ms，sysmon就会认为其运行时间太久而发出抢占式调度的请求。一旦G的抢占标志位被设为true，那么待这个G下一次调用函数或方法时，runtime便可以将G抢占，并移出运行状态，放入P的local runq中，等待下一次被调度。

## channel阻塞或network I/O情况下的调度

如果G被阻塞在某个channel操作或network I/O操作上时，G会被放置到某个wait队列中，而M会尝试运行下一个runnable的G；如果此时没有runnable的G供m运行，那么m将解绑P，并进入sleep状态。当I/O available或channel操作完成，在wait队列中的G会被唤醒，标记为runnable，放入到某P的队列中，绑定一个M继续执行。

## system call 阻塞情况下的调度

如果G被阻塞在某个system call操作上，那么不光G会阻塞，执行该G的M也会解绑P(实质是被sysmon抢走了)，与G一起进入sleep状态。如果此时有idle的M，则P与其绑定继续执行其他G；如果没有idle M，但仍然有其他G要去执行，那么就会创建一个新M。

当阻塞在syscall上的G完成syscall调用后，G会去尝试获取一个可用的P，如果没有可用的P，那么G会被标记为runnable，之前的那个sleep的M将再次进入sleep。