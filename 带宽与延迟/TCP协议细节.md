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
在传输过程中，有可能出现发送方的速度大于接收方能够接受的速度，就会导致接收方负载过重而导致拥堵，流量控制酒就显得很重要。

TCP的流量控制通过接收窗口大小字段实现，该字段给出接收方的接收缓冲区当前可用字节，即接收方进行流量控制，由16位表示，所以最大为`65535`字节，但是通过抓包发现有些包的窗口字段大小远远超过此值，原因是`RFC 1323`定义的`窗口缩放（TCP Window Scaling）`机制。有关TCP窗口缩放配置参考维基百科：[https://en.wikipedia.org/wiki/TCP_window_scale_option](https://en.wikipedia.org/wiki/TCP_window_scale_option)通过开启窗口缩放，可以更充分利用带宽，提高性能。

在建立连接时即有接收窗口协商，在传输过程中，`接收窗口(rwnd)`大小可以随时改变。发送方实际发送字节数还与`拥塞控制`有关，`拥塞窗口(cwnd)`由发送方根据网络拥塞程度设定，最终的发送窗口为两者较小值：

```
发送窗口=Min[rwnd,cwnd]
```

## 拥塞控制

// todo