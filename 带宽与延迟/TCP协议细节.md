# TCP协议细节

`TCP`属于OSI七层模型中的传输层协议，位于网络边缘，提供端到端的可靠数据传输，其有着承上启下的作用，协议数据单元为报文段（Message Segment)。

TCP需要提供以下功能：

* 分组和复用
* 应用报文的差错检测（包括出错、丢失、重复、失序、超时等）
* 提供端到端的流量控制
* 提供拥塞控制
* 运输连接建立与释放
* 连接控制
* 报文段序号

传输层为应用进程之间提供端到端的逻辑通信，之所以称为逻辑通信，是因为其看似为两端之间的水平传输，实际为垂直方向的封包和拆包过程。此概念上一节已提到。

## 复用和分用

应用层的多个进程通过传输层的端口号将应用报文向下交给传输层，然后传输层向下`复用`网络层服务。网络层的IP地址和传输层的端口号共同标识主机上特定的进程，当网络层数据报到达目的主机后，拆包得到运输层segment，通过其中的端口号信息，将应用层报文交给相应的进程，此为`分用`。

![http://7xsi10.com1.z0.glb.clouddn.com/tcp1.png](http://7xsi10.com1.z0.glb.clouddn.com/tcp1.png)

## TCP协议的PDU
一图以明之：
![http://7xsi10.com1.z0.glb.clouddn.com/tcp_pdu.png](http://7xsi10.com1.z0.glb.clouddn.com/tcp_pdu.png)

## 连接建立与释放
这里将通过抓包分析连接建立和释放过程。

### 三次握手建立连接
一图以概之：
![http://7xsi10.com1.z0.glb.clouddn.com/tcp_hand.jpg](http://7xsi10.com1.z0.glb.clouddn.com/tcp_hand.jpg)

访问`luoxia.me`抓包，首先看一下总的流程：
![http://7xsi10.com1.z0.glb.clouddn.com/hand_SUm.png](http://7xsi10.com1.z0.glb.clouddn.com/hand_SUm.png)
#### 第一次握手
第一次握手由客户端（此处为端口号为60782的Chrome进程)主动向服务端发起，设置初始`Sequence number`(序列号)为0；`Acknowledge number`设为0表示期望接受到的返回包序列号为0；`ACK标志`为0，`Syn`为1。即SYN=1， ACK标志=0表示一个连接请求报文段(SYN用于建立连接时的同步信号):

![http://7xsi10.com1.z0.glb.clouddn.com/hand1.png](http://7xsi10.com1.z0.glb.clouddn.com/hand1.png)

#### 第二次握手
服务端在收到连接建立请求后返回包中：设置初始`Sequence number`(序列号)为0；`Acknowledge number`设为1表示期望下次接受到的请求包序列号为1，同时也通知客户端已经收到0号包；`ACK`标志为1，`Syn`为1。即SYN=1， ACK标志=1表示同意建立连接（ACK标志为1时表示此包为确认包）:
![http://7xsi10.com1.z0.glb.clouddn.com/hand2.png](http://7xsi10.com1.z0.glb.clouddn.com/hand2.png)

#### 第三次握手
客户端在收到服务端答复后再一次请求：设置`Sequence number`(序列号)为`0+1=1`；`Acknowledge number`设为1表示期望下次接受到的返回包序列号为1，也即告知服务端已经收到服务端发送的0号包；`ACK`标志为1，`Syn`为0。
![http://7xsi10.com1.z0.glb.clouddn.com/hand3.png](http://7xsi10.com1.z0.glb.clouddn.com/hand3.png)

至此，连接建立，方可进行应用数据传输。可以看见，为了可靠传输而需建立连接，这就是握手时延，通常为减少握手时延，使用持久连接技术，这方面会在`http1.x`相关小节详解。

### 四次挥手释放连接

四次挥手的抓包过程这里就省略了，一图以概之：
![http://7xsi10.com1.z0.glb.clouddn.com/tcpbye.jpg](http://7xsi10.com1.z0.glb.clouddn.com/tcpbye.jpg)

图中`2MSL`的`TIME-WAIT`用以保证对方接收到最后的ACK信息。

## 流量控制
在传输过程中，有可能出现发送方的速度大于接收方能够接受的速度，就会导致接收方负载过重而导致拥堵，流量控制就显得很重要。

TCP的流量控制通过接收窗口大小字段实现，该字段给出接收方的接收缓冲区当前可用字节，即接收方进行流量控制，由16位表示，所以最大为`65535`字节，但是通过抓包发现有些包的窗口字段大小远远超过此值，原因是`RFC 1323`定义的`窗口缩放（TCP Window Scaling）`机制。有关TCP窗口缩放配置参考维基百科：[https://en.wikipedia.org/wiki/TCP_window_scale_option](https://en.wikipedia.org/wiki/TCP_window_scale_option)通过开启窗口缩放，可以更充分利用带宽，提高性能。

在建立连接时即有接收窗口协商，在传输过程中，`接收窗口(rwnd)`大小可以随时改变。发送方实际发送字节数还与`拥塞控制`有关，`拥塞窗口(cwnd)`由发送方根据网络拥塞程度设定，最终的发送窗口为两者较小值：

```
发送窗口=Min[rwnd,cwnd]
```

## 拥塞控制

与流量上述的流量控制不同，拥塞控制解决发送方与网络的关系。造成网络拥塞有许多原因，涉及到网络协议的传输层、网络层、数据链路层。

TCP的拥塞控制手段主要有：慢启动、拥塞避免、快重传、快恢复。

### 慢启动
即最开始拥塞窗口（`cwnd`）为1个MSS，经过一个RTT后，若没有出现拥塞，则`cwnd`设置为2,第二个RTT后设置为4，即指数增长，直到达到门限值`ssthresh`。
![http://7xsi10.com1.z0.glb.clouddn.com/con1.png](http://7xsi10.com1.z0.glb.clouddn.com/con1.png)

### 拥塞避免
即`AIMD`中的`AI`（线性增长)，当`cwnd`大于或等于`ssthresh`时，不再以指数增长的形式增加cwnd，而是每个RTT只增加一个MSS大小即`加法增`，避免过早出现网络拥塞。

### 拥塞产生
当发送方没有按时收到ACK（超时）或是收到了重复的ACK，即发生了拥塞。乘法减(MD)就是将`ssthresh`快速下降到此时`cwnd`的一半。然后有两种处理方案:`TCP Tahoe`版本和`TCP Reno`版本。

#### `TCP Tahoe`:
当发送方检测到超时的发生时，就采取此种方案。将`cwnd`重新设置为1个MSS，再从头开始应用慢启动算法。

#### `TCP Reno`:
`快重传`: 当发送方接收到3个重复的ACK时，立即重传，而不用等到超时发生。将`cwnd`设置为新的`ssthresh`值（之前`cwnd`值的一半)，然后使用`快恢复`算法。

`快恢复`:

* `cnwd`设置为`ssthresh + 3 * MSS`，因为收到3个重复ACK，说明有3个包收到了:
```c
if ( dupacks >= 3 ) {  
        ssthresh = max( 2 , cwnd / 2 ) ;  
        cwnd = ssthresh + 3 * SMSS ;  
}
```
* 重传重复ACK指定的包
* 此后每收到一个重复的ACK确认时，`cwnd++`
* 当收到对新发送数据的ACK确认时，`cwnd = ssthresh`，之后进入拥塞避免算法（线性增）

如图：

![http://7xsi10.com1.z0.glb.clouddn.com/con2.png](http://7xsi10.com1.z0.glb.clouddn.com/con2.png)

但是`TCP Reno`存在的问题是，如果同一窗口丢失了多个包时，快重传机制只会重传一个包，剩下的只能等到超时时间到达。现在TCP使用的是`TCP New Reno`算法解决此问题。

## 重传机制

谈及重传时，需要先介绍几种实现可靠传输协议（停等协议、回退N协议、选择重传协议）中的选择重传协议，TCP使用选择重传协议。
### 选择重传协议

![http://7xsi10.com1.z0.glb.clouddn.com/con3.png](http://7xsi10.com1.z0.glb.clouddn.com/con3.png)
即接收端在收到顺序不对的包时，窗口不滑动，将这些包先放入到缓冲区保存。发送端只重传超时的包。

### 超时重传
上图所示的重传是超时重传，但是这可能需要等比较长的时间。比如`client`发送了`0,1,2,3`包，`server`端收到`0,1,3`即2号包丢失，返回`ACK`为`1,2,4`，2号包将一直等到超时才会重传。见上图。这里面还涉及到超时时间(RTO)的计算，此处不详解。

### 快重传
比如`client`发送了`0,1,2,3,4`包，`server`端收到`0,1,3,4`即2号包丢失,返回`ACK`为`1,2,2,2`，`2`号包丢失，后面的`3,4`都收到了，但`ACK`返回`2`，`client`收到连续三次重复的ACK，就对丢失的`2`号包进行重传。

## 其他
### 队首阻塞
TCP保证各个分组之间的正确顺序，上面的选择重传例子中，如果中间某一个包丢失了，后续到达的包只能暂时存在接受者的缓冲区，直到错误包重传后才能滑动窗口，这就是所谓的队首延迟，后续我们会在HTTP/2中继续讨论此话题。
### 延迟确认
由于单独的确认信息比较小，许多TCP栈实现了`延迟确认`算法，在特定的窗口时间（100-200ms)内将确认消息放到缓冲区，等待有能够捎带它的数据分组，如果没有等到，就单独发送确认消息。这会导致一个性能问题：当没有足够多的回传数据分组时，会引入相当大的时延，在Linux下，可以通过调用`setsockopt()`函数，设置TCP_QUICKACK:
```c
setsockopt(fd, IPPROTO_TCP, TCP_QUICKACK, (int[]){1}, sizeof(int));
```

### 慢启动重启

除了调节新连接的传输速度，TCP 还实现了 `SSR`(Slow-Start Restart，慢启动重 启)机制。这种机制会在连接空闲一定时间后重置连接的拥塞窗口。道理很简单， 在连接空闲的同时，网络状况也可能发生了变化，为了避免拥塞，理应将拥塞窗 口重置回“安全的”默认值。
毫无疑问，SSR 对于那些会出现突发空闲的长周期 TCP 连接(比如 HTTP 的 keep-alive 连接)有很大的影响。因此，我们建议在服务器上禁用 SSR。在 Linux 平台，可以通过如下命令来检查和禁用 `SSR`:
```
$> sysctl net.ipv4.tcp_slow_start_after_idle
$> sysctl -w net.ipv4.tcp_slow_start_after_idle=0
```

### `Nagle`算法与`TCP_NODELAY`

`Nagle`算法用于避免太多的小数据包传输，减轻网络负载，比如有可能发送端每次只发送一个字节的包。

`Nagle`算法试图在发送一个分组前，将大量TCP数据绑定在一起，一并发送，只有当其他分组被确认后，才允许发送非全尺寸的分组，否则先缓存起来，凑齐全尺寸后再发送。

但这也会造成性能问题，小的数据包可能永远无法发送，且存在延迟确认机制。

可以通过设置`TCP_NODELAY`来禁用`Nagle`算法。

## 总结

通过TCP协议细节，我们可以看到，为了保证端到端的可靠数据传输，TCP协议在性能方面存在许多方面的瓶颈。连接建立会带来握手时延，如果连接不能复用，每次都建立新的连接则会造成更大延时；窗口大小限制使得吞吐量存在瓶颈；为了实现拥塞控制，慢启动，拥塞避免等算法也会会造成性能瓶颈；除此之外，协议设计时为了提高性能采取的策略在有些时候反而会造成延时，也需要我们针对特定情况进行设置。

针对这些可能存在的性能瓶颈采取的优化措施，我们会在单独的小节中进行总结。