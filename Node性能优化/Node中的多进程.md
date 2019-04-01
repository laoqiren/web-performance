# Node中的多进程

在Node中与进程相关的三个对象：`process`、`child_process`、`cluster`。

然后此篇文章主要从以下几个方面对Node中的多进程架构进行阐述：

* 子进程的创建，对比Linux下的多进程编程
* IPC进程间通信
  * IPC建立过程
  * 内部实现
  * 消息传递
* 分发请求与负载均衡
  * `handle`传递
  * 负载均衡机制
  * 端口复用
* 进程守护、优雅退出、心跳检测等

## Process

`Process`对象用于提供当前进程的相关信息以及对进程进行控制。参考Node相关文档[https://nodejs.org/dist/latest-v8.x/docs/api/process.html](https://nodejs.org/dist/latest-v8.x/docs/api/process.html)

## child_process

`child_process`主要提供用于创建子进程的能力和有关子进程的信和和控制。

Node中的`fork()`与linux编程中的有所区别。Node中的`fork`并不是主进程的一个拷贝，而是拥有独立内存空间，独立的V8实例；然后`fork`出来的进程和主进程会有额外的`IPC通道`用于进程间通信。

Node中的`exec`和`spwan`和linux编程下的也有区别，在上一节中我们讲这两个函数簇会用其他进程替换掉当前进程，而在Node中是另起一个进程执行指定的程序。

Node中`exec`和`spawn`的区别：

* `spawn`出来的子进程提供流式API用于读取来自子进程的数据，适合返回数据比较大的情况
* `spwan`在未配置`shell: true`情况下直接执行文件，而不会起一个`shell`用于执行命令
* `exec`提供额外的`callback`参数，用于缓存子进程的数据结束时一次性返回给主进程，适合返回数据较小的场景，如只返回状态码

`exec`会先起一个`shell`，再执行命令，而`execFile`和其类似，但是不会先起一个`shell`。

`spawn`和linux编程下的`popen()`很类似，而`exec`和linux编程下的`system()`类似。

## IPC

进程间通信是多进程架构的核心

* 使用`fork()`创建子进程时，父进程会创建IPC通道并进行监听，再创建出子进程，并通过环境变量(`NODE_CHANNEL_FD`)告诉子进程这个IPC通道的文件描述符。子进程在启动过程中，根据文件描述符去连接这个IPC通道。
* IPC底层实现由`libuv`提供，在Windows下由命名管道(`named pipe`)实现，*nix系统采用`Unix Domain Socket`实现。
* 暴露到应用层的是基于事件发布订阅模式的`message`事件和`send()`方法。
* 可以发送一般信息也可以发送`handle`句柄信息。

通过发送句柄我们可以多个进程共同监听同一个端口。

## 分发请求与负载均衡
多进程的一个应用场景就是充分利用多核资源来提高服务的负载能力。

### 方式一：发送Server对象

```js
// master.js

const subprocess = require('child_process').fork('subprocess.js');

// Open up the server object and send the handle.
const server = require('net').createServer();
server.on('connection', (socket) => {
  socket.end('handled by parent');
});
server.listen(1337, () => {
  subprocess.send('server', server);
});

// subprocess.js
process.on('message', (m, server) => {
  if (m === 'server') {
    server.on('connection', (socket) => {
      socket.end('handled by child');
    });
  }
});
```
此种方案存在一个严重问题：请求来的时候，没有办法控制将请求交给哪一个进程处理，这样会导致两个问题：

* 多个进程之间会竞争 accpet 一个连接，产生惊群现象，效率比较低。
* 由于无法控制一个新的连接由哪个进程来处理，必然导致各 `worker` 进程之间的负载非常不均衡。

### 方式二：发送socket对象

```js
// master.js
const net = require('net');
const fork = require('child_process').fork;

var workers = [];
for (var i = 0; i < 4; i++) {
   workers.push(fork('./worker'));
}

var handle = net._createServerHandle('0.0.0.0', 3000);
handle.listen();
handle.onconnection = function (err,handle) {
    // 通过某种算法决策出一个子进程处理请求
    var worker = workers.pop();
    worker.send({},handle);
    workers.unshift(worker);
}

// subprocess.js
const net = require('net');
process.on('message', function (m, handle) {
  start(handle);
});

var buf = 'hello Node.js';
var res = ['HTTP/1.1 200 OK','content-length:'+buf.length].join('\r\n')+'\r\n\r\n'+buf;

function start(handle) {
    console.log('got a connection on worker, pid = %d', process.pid);
    var socket = new net.Socket({
        handle: handle
    });
    socket.readable = socket.writable = true;
    socket.end(res);
}
```
这样我们可以将决策过程交给master进程，使得进程间的负载就变得可控，也不会出现"惊群"现象。

### 端口复用
通过`cluster`创建服务集群简单的例子：
```js
// server.js
var cluster = require('cluster');
var cpuNums = require('os').cpus().length;
var http = require('http');

if(cluster.isMaster){
  for(var i = 0; i < cpuNums; i++){
    cluster.fork();
  }
}else{
  http.createServer(function(req, res){
    res.end(`response from worker ${process.pid}`);
  }).listen(3000);

  console.log(`Worker ${process.pid} started`);
}
```
每个子进程都会有相应的调用`listen(3000)`，这其中涉及到`cluster`内部对于端口复用方面的特殊处理，具体内部细节这里不详述，本质上最后监听端口的也只是`master`进程，`worker`进程做的只是接受`master`派发的请求进行处理。

在网上找到一张图解：
![/assets/cluster_net.png](/assets/cluster_net.png)

### 负载均衡
Node v0.11中提供一种`Round-Robin`的策略（调度）来进行分配任务。主进程接受连接，在N个工作进程中，每次选择第`i=(i+1)mod N`个进程来发送连接。Node默认选择`Round-Robin`方式

```js
cluster.schedulingPolicy = cluster.SCHED_RR;//启用RR
cluster.schedulingPolicy = cluster.SCHED_NONE;//不启用RR
```

或者在环境变量中设置NODE_CLUSTER_POLICY:

```
export NODE_CLUSTER_POLICY=rr;
export NODE_CLUSTER_POLICY=none;
```

## 其他要点
要提高多进程集群架构的稳定性，我们需要做的还有很多，如优雅退出、进程守护等。

### 优雅退出
由于`master`进程负责守护子进程，调度任务，所以必须要保证`master`进程足够健壮。当然在`master`挂掉时，也会有其他解决方案，这方面可以参考相关架构思想。

对于`worker`的退出，我们要停止接收新的请求，并告知`master`进程该`worker`即将退出，`master`可以重新`fork`新的子进程。`Node`中主要是`uncaughtException`和`domain`的应用。

参考文章：[http://www.infoq.com/cn/articles/quit-scheme-of-node-uncaughtexception-emergence](http://www.infoq.com/cn/articles/quit-scheme-of-node-uncaughtexception-emergence)

### 守护进程
对于整个应用，需要守护`master`进程。创建守护进程的方式有很多，参考

[http://www.ruanyifeng.com/blog/2016/02/linux-daemon.html](http://www.ruanyifeng.com/blog/2016/02/linux-daemon.html)

[https://www.freebsd.org/doc/zh_CN/books/handbook/basics-daemons.html](https://www.freebsd.org/doc/zh_CN/books/handbook/basics-daemons.html)

对于`worker`的退出，对应上面的`worker`的优雅退出，`worker`退出时，`master`可以重新`fork`新的子进程：

```js
cluster.on('exit', function () {
    clsuter.fork();
});

cluster.on('disconnect', function () {
    clsuter.fork();
});
```

## PM2
`PM2`为`Node`提供进程管理，提供守护进程、监控、日志、平滑重启、集群负载均衡能力等。

这里介绍一个点：

在`PM2`中有两种执行模式：`fork`和`cluster`模式，两者的区别是，`fork`模式使用`fork()`作为创建子进程执行程序的方式，这样可以方便地执行其他非`Node`程序；而`cluster`模式则只针对`Node`可用，因为其使用了`cluster`的一系列API，此中方案可以很方便地创建`cluster`集群。

参考[https://stackoverflow.com/questions/34682035/cluster-and-fork-mode-difference-in-pm2](https://stackoverflow.com/questions/34682035/cluster-and-fork-mode-difference-in-pm2)

## Nginx

Nginx的涉及理念和Node颇为相似，其同样具有多进程架构、事件驱动模型，具体这方面我们放在集群与负载均衡章节当中介绍。