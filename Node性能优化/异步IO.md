# 异步I/O
在JS或Node.js的一系列概念中，我们常常见到单线程、多进程、非阻塞I/O、异步I/O、事件循环等等。那么这些概念之间有什么联系呢？

我认为要想真正理解这些概念，得从底层操作系统内核层面到浏览器内核（或者Node底层的libuv)再到应用层面的异步编程去理解。

## CPU计算与I/O操作

在操作系统的发展过程中，经历了人工到单道批处理、多道批处理系统再到如今的分时、实时等系统。下面是单道批处理和多到批处理系统的示意图：

![http://7xsi10.com1.z0.glb.clouddn.com/cpuio.png](http://7xsi10.com1.z0.glb.clouddn.com/cpuio.png)

可以看到多道批处理可以更充分利用计算机资源，增加系统吞吐量。某个进程需要I/O而阻塞时，将CPU资源让给其他进程。

## 进程与线程

进程我们在[多进程架构](/Node性能优化/多进程架构.md)中已经做了较多的介绍，它是计算机资源分配的独立单位。我们知道进程之间的调度、通信、切换等都需要大量地开销；虽然我们可以利用多进程来利用多核CPU（多核的问题又涉及到计算机系统结构，这里请参考其他资料)或者分时共享，但是进程太重，因此在进程的基础上提出了线程的概念。一个进程可有多个线程，线程是CPU调度的单位，多个线程共享所属进程的内存等资源。

## Linux I/O模型
这里以Linux系统为例介绍不同的I/O模型。操作系统为了安全考虑起见，将整个内存空间划分为内核空间和用户空间，实际的I/O操作由内核程序完成，暴露给用户空间程序的只是一些与I/O操作相关的API，I/O数据先是拷贝到内核空间再拷贝到用户空间。

### 阻塞I/O
阻塞I/O理解起来很简单，程序调用I/O操作以后，会进入阻塞状态（不占用CPU)，一直等到I/O操作返回：

![http://7xsi10.com1.z0.glb.clouddn.com/blockingio.png](http://7xsi10.com1.z0.glb.clouddn.com/blockingio.png)

### 非阻塞I/O
非阻塞I/O不同于阻塞I/O的是，调用I/O操作后立即返回，之后通过不同的方式（轮询或者I/O多路复用）判断I/O操作是否完成。

#### 轮询
此种方式虽然调用I/O操作立即返回，但是需要用户空间程序不断去轮询查询I/O操作是否完成，结果就是将大量的CPU时间耗费在了轮询上面，如`read()`调用：

![http://7xsi10.com1.z0.glb.clouddn.com/readio.png](http://7xsi10.com1.z0.glb.clouddn.com/readio.png)


#### I/O多路复用

Linux中的`select()`、`poll()`、`epoll()`都是I/O多路复用的例子。

![http://7xsi10.com1.z0.glb.clouddn.com/selectio.png](http://7xsi10.com1.z0.glb.clouddn.com/selectio.png)

I/O多路复用方式调用I/O这一步虽然是非阻塞的，但是却阻塞在了`select()`、`poll()`和`eoll()`函数上，要么等待文件描述符状态变化后返回，要么等待事件唤醒而处于休眠状态。是同步I/O。

关于`epoll()`可以阅读我的一篇博客文章[libuv网络I/O机制](thunder://QUFmdHA6Ly9keTpkeUB4bGouMnR1LmNjOjEwMzc3L1vov4Xpm7fkuIvovb13d3cuMnR1LmNjXeWLh+aVoueahOW/gy5CRDEyODDotoXmuIXlm73oi7Hlj4zor63lj4zlrZcubWt2Wlo=)其中关于网络I/O就是使用的epoll实现的。

### 异步I/O

AIO是Linux kernal 2.6中的新特性，其真正做到了异步I/O，用户程序调用I/O操作后立即返回，然后可以去做其他的事情，执行其他用户逻辑代码，等到I/O完成后，会通过信号或回调函数将数据传给用户程序：

![http://7xsi10.com1.z0.glb.clouddn.com/asyncio.png](http://7xsi10.com1.z0.glb.clouddn.com/asyncio.png)

由于跨平台型问题和其本身的缺陷，这种方式用得比较少。

## 浏览器的进程与线程

// todo 
## 事件循环线程与观察者

// todo
## Node事件循环

// todo

## microtask和macrotask

// todo