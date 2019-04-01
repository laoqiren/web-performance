# 异步I/O
在JS或Node.js的一系列概念中，我们常常见到单线程、多进程、非阻塞I/O、异步I/O、事件循环等等。那么这些概念之间有什么联系呢？

我认为要想真正理解这些概念，得从底层操作系统内核层面到浏览器内核（或者Node底层的libuv)再到应用层面的异步编程去理解。

## CPU计算与I/O操作

在操作系统的发展过程中，经历了人工到单道批处理、多道批处理系统再到如今的分时、实时等系统。下面是单道批处理和多到批处理系统的示意图：

![/assets/cpuio.png](/assets/cpuio.png)

可以看到多道批处理可以更充分利用计算机资源，增加系统吞吐量。某个进程需要I/O而阻塞时，将CPU资源让给其他进程。

## 进程与线程

进程我们在[多进程架构](/Node性能优化/多进程架构.md)中已经做了较多的介绍，它是计算机资源分配的独立单位。我们知道进程之间的调度、通信、切换等都需要大量地开销；虽然我们可以利用多进程来利用多核CPU（多核的问题又涉及到计算机系统结构，这里请参考其他资料)或者分时共享，但是进程太重，因此在进程的基础上提出了线程的概念。一个进程可有多个线程，线程是CPU调度的单位，多个线程共享所属进程的内存等资源。

## Linux I/O模型
这里以Linux系统为例介绍不同的I/O模型。操作系统为了安全考虑起见，将整个内存空间划分为内核空间和用户空间，实际的I/O操作由内核程序完成，暴露给用户空间程序的只是一些与I/O操作相关的API，I/O数据先是拷贝到内核空间再拷贝到用户空间。

### 阻塞I/O
阻塞I/O理解起来很简单，程序调用I/O操作以后，会进入阻塞状态（不占用CPU)，一直等到I/O操作返回：

![/assets/blockingio.png](/assets/blockingio.png)

### 非阻塞I/O
非阻塞I/O不同于阻塞I/O的是，调用I/O操作后立即返回，之后通过不同的方式（轮询或者I/O多路复用）判断I/O操作是否完成。

#### 轮询
此种方式虽然调用I/O操作立即返回，但是需要用户空间程序不断去轮询查询I/O操作是否完成，结果就是将大量的CPU时间耗费在了轮询上面，如`read()`调用：

![/assets/readio.png](/assets/readio.png)


#### I/O多路复用

Linux中的`select()`、`poll()`、`epoll()`都是I/O多路复用的例子。

![/assets/selectio.png](/assets/selectio.png)

I/O多路复用方式调用I/O这一步虽然是非阻塞的，但是却阻塞在了`select()`、`poll()`和`eoll()`函数上，要么等待文件描述符状态变化后返回，要么等待事件唤醒而处于休眠状态。是同步I/O。

关于`epoll()`可以阅读我的一篇博客文章[libuv网络I/O机制](thunder://QUFmdHA6Ly9keTpkeUB4bGouMnR1LmNjOjEwMzc3L1vov4Xpm7fkuIvovb13d3cuMnR1LmNjXeWLh+aVoueahOW/gy5CRDEyODDotoXmuIXlm73oi7Hlj4zor63lj4zlrZcubWt2Wlo=)其中关于网络I/O就是使用的epoll实现的。

### 异步I/O

AIO是Linux kernal 2.6中的新特性，其真正做到了异步I/O，用户程序调用I/O操作后立即返回，然后可以去做其他的事情，执行其他用户逻辑代码，等到I/O完成后，会通过信号或回调函数将数据传给用户程序：

![/assets/asyncio.png](/assets/asyncio.png)

由于跨平台型问题和其本身的缺陷，这种方式用得比较少。

## 浏览器的进程与线程

### 浏览器采用多进程架构

![/assets/chrometasks.png](/assets/chrometasks.png)

可以看到浏览器有一个主进程；每个扩展程序都会是一个单独的进程；另外每个标签页都会起一个进程（多个空标签页可能会合并，如上图所示）。

具体来讲，浏览器会具有下面这几种进程：

* 主进程：负责统筹协调，包括处理用户交互、界面显示、网络资源下载等
* 扩展程序进程
* GPU进程（如上面截图所示）
* 渲染进程（每个标签页一个渲染进程）

使用多进程可以提高服务稳定性，避免各种任务的互相影响，同时也能充分利用多核提高性能。

### 渲染进程为多线程
在之前的网页渲染原理章节，我们提到了浏览器内核的概念，我们讲到浏览器内核包括渲染引擎和JS引擎。实际上渲染和JS都是单独的线程；而我们也知道浏览器背后有一个事件循环在运作，它也是单独的线程。另外定时器和ajax请求也会是单独的线程。

### 关系
各种进程和渲染进程多个线程是怎么合作通信的呢？

* 主进程收到用户请求，首先需要获取页面内容，随后将该任务通过RendererHost接口传递给渲染进程；
* 渲染进程的渲染接口收到消息，简单解释后，交给渲染线程，然后开始渲染；
* 渲染线程接收请求，加载网页并渲染网页，这其中可能需要主进程获取资源和需要GPU进程来帮助渲染，期间可能会有JS线程操作DOM（这样可能会造成回流并重绘）；
* 最后渲染进程将结果传递给主进程，主进程接收到结果并将结果绘制出来。

渲染引擎中的各个线程：

* 渲染线程与JS线程互斥，防止JS修改页面同时渲染网页造成不一致问题（之前讲到的JS执行阻塞DOM构建就是这个原因）；
* 事件循环线程负责管理各种事件（后面详解）
* 可能会有子JS线程（WebWorker）
## 事件循环线程与观察者
从上面的一系列分析我们明白，虽然我们常说JS是单线程的，这只是说明执行JS的JS引擎是单线程的，浏览器内核可不是单线程；同理Nodejs也不是，执行js代码虽然是单线程的，但是整个平台是多线程的，我之前写过一篇关于libuv异步I/O的博客文章，分析了部分Node源码，涉及到了js代码背后C++层面的事件循环，执行js的V8和背后的事件循环程序都是Node进程下的不同线程，彼此合作才实现了我们在应用层用到的异步API。

### 观察者
借用《深入浅出Node.js》中的例子，一个饭馆中的厨房，厨房制作菜肴，而具体要哪些菜肴需要根据客户需求得知，而收银台小妹负责记录客户需要的菜肴，厨房向收银台小妹询问得知菜肴需求，每做完一道菜，再去询问收银台小妹，直到没有其他需求。这里收银台小妹就是观察者，厨师就是事件循环，客户就是我们应用层程序调用异步API，事件循环不断检查观察者队列，直到没有观察者时退出循环。以我们调用`listen()`方法监听socket端口为例：

* 用户调用异步API说明我们监听什么事件，我们需要当有客户端请求来临时触发连接事件；
* 从js层面来到C++层面，将js层面感兴趣的事件加入其对应的观察者事件队列里，如这里的IO观察者，并将IO观察者加入到事件循环的观察者队列里；
* 事件循环程序循环地遍历观察者队列，从观察者那里得知需要处理哪些事件，调用系统或者浏览器接口处理底层的异步逻辑（如处理用户点击、linux调用`epoll()`监听socket状态变化)
* 当监听的事件发生时（如用户点击、socket状态变化），拿到数据或其他信息，调用相应的回调函数；
* 回到js层面，执行回调函数代码。

### 事件循环
分析Node源码，我们发现了背后的事件循环程序：
```js
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);

//这里就是那个被称作event loop的while loop
  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);
    uv__run_timers(loop);
    ran_pending = uv__run_pending(loop);
    uv__run_idle(loop);
    uv__run_prepare(loop);

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);

    uv__io_poll(loop, timeout);
    uv__run_check(loop);
    uv__run_closing_handles(loop);

    if (mode == UV_RUN_ONCE) {
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}
```
从源代码就可以看出，事件循环的观察者队列里有不同观察者，而处理这些观察者是有顺序的，参考Node.js官方文档：[https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)

大致的顺序如下：

```
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
```

* timers处理定时任务`setTimeout() `和`setInterval()`；
* I/O callbacks处理一些系统调用的错误；
* poll观察者即IO观察者；
* check观察者：`setImmediate()`。

## microtask和macrotask

关于`task queque`（任务队列）的文档：[https://html.spec.whatwg.org/multipage/webappapis.html#task-queue](https://html.spec.whatwg.org/multipage/webappapis.html#task-queue)

* 一个事件循环(event loop)会有一个或多个任务队列(task queue) task queue 就是 macrotask queue
* 每一个 event loop 都有一个 microtask queue

程序执行过程：

* 在 macrotask 队列中执行最早的那个 task ，然后移出
* 执行 microtask 队列中所有可用的任务，然后移出
* 下一个循环，执行下一个 macrotask 中的任务

具体：

* macrotasks: `setTimeout()` `setInterval()` 、`setImmediate()`、`I/O`、`UI渲染`
* microtasks: `Promise`、`process.nextTick()`、`Object.observe()`、`MutationObserver`

这方面更加具体的讲解，可以参考这篇文章：[https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)

## 异步I/O下的高性能服务器

//todo