---
title: TCP连接的建立和关闭
tags:
- tcp
categories:
- network
---

# 1. 有限状态机

有限状态机（FSM）是一种用来对象行为建模的工具，用以描述对象生命周期内所经历的状态序列，以及如何响应外界事件。

它有三个组成部分：

1. 状态：所有可能存在的状态，包括当前状态和条件满足后要迁移的状态；
2. 事件：也称为转移条件，当一个条件被满足，将会触发一个动作，或者执行一次状态的迁移。；
3. 动作：条件满足后执行的动作。动作执行完毕后，可以迁移到新的状态，也可以仍旧保持原状态。动作不是必需的，当条件满足后，也可以不执行任何动作，直接迁移到新状态。

以下图为例：

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220504120237.png)

上图中椭圆形中是状态，A1/A2/A3是指的事件，/后面指的是动作，比如：

- Smaill Mario状态下，满足A1事件后，触发动作+1000，并且状态转移成了Super Mario；
- Super Mario在满足A1事件后，触发动作+1000，状态转移成了Fire Mario；
- Fire Mario在满足A3事件下，触发发射火球的动作，在A2事件下，没有任何动作，但是状态转移成了Small Mario；
- Small Mario在满足A2事件下，没有任何动作，但是状态转移成了Dead Mario。

可以详细看下这里面举的相对简单的验票闸门（coin-operated turnstile）的例子：

[Finite-state machine - Wikipedia](https://en.wikipedia.org/wiki/Finite-state_machine)

# 2. TCP有限状态机

参考：[The TCP/IP Guide - TCP Operational Overview and the TCP Finite State Machine (FSM) (tcpipguide.com)](http://tcpipguide.com/free/t_TCPOperationalOverviewandtheTCPFiniteStateMachineF-2.htm)

![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220504122742.png)

借助于TCP的FSM我们来理解一下TCP的建立和关闭。

1. 三次握手建立连接

   我们使用最典型的连接图来理解：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220504152627.png)

   - 开始时客户端和服务器端都处于CLOESD状态，之后服务器端被动打开连接，建立起[TCB](http://tcpipguide.com/free/t_TCPConnectionPreparationTransmissionControlBlocksT.htm)数据块，由于服务端并不知道哪个客户端会连接它，所以它默认将TCB模块中客户端Socket数据初始化为0，并且进入LISTEN状态。这是一个客户端主动打开连接，建立TCB数据结构，接下来进入第一次握手阶段；
   - 第一次握手：客户端向服务端发送一个 SYN 报文（SYN = 1），并指明客户端的初始化序列号 ISN(x)，即图中的 seq = x，表示本报文段所发送的数据的第一个字节的序号。此时客户端转移到SYN-SEND 状态；
   - 第二次握手：服务器收到客户端的 SYN 报文之后，会发送 SYN 报文作为应答（SYN = 1），并且指定自己的初始化序列号 ISN(y)，即图中的 seq = y。同时会把客户端的 ISN + 1 作为确认号 ack 的值，表示已经收到了客户端发来的的 SYN 报文，希望收到的下一个数据的第一个字节的序号是 x + 1，所这里是发送了SYN+ACK，此时服务器处于 SYN-RECEIVED 的状态；
   - 第三次握手：客户端收到服务器端响应的 SYN+ACK 报文之后，会发送一个 ACK 报文，也是一样把服务器的 ISN + 1 作为 ack 的值，表示已经收到了服务端发来的的 SYN 报文，希望收到的下一个数据的第一个字节的序号是 y + 1，并指明此时客户端的序列号 seq = x + 1（初始为 seq = x，所以第二个报文段要 +1），此时客户端进入了 ESTABLISHED 状态。服务器收到 ACK 报文之后，也处于 ESTABLISHED状态，至此，双方建立起了 TCP 连接。

   说明：

   - 注意区分上面图中ack和ACK的区别；
   - ISN是动态生成的，每次都不一样；
   - 第三次握手丢失之后，也就是服务端一直没有收到回包，会尝试重传SYN+ACK，直到达到系统规定的重传次数限制，之后将该连接从半连接队伍（服务端为没有完全建立请求的连接建立的队列）中删除；
   - 三次握手结束之后，客户端和服务端都能同时确认双方发送和接受都没有问题，之后就可以正式开始传输数据。

   关于为什么要使用三次握手，而不是其他次数握手的答案： [TCP 为什么是三次握手，而不是两次或四次？ (qq.com)](https://mp.weixin.qq.com/s/NIjxgx4NPn7FC4PfkHBAAQ) 以及 [RFC793](https://www.ietf.org/rfc/rfc793.txt)

2. 四次挥手关闭连接

   照例先看一张图：

   ![](https://images-pigo.oss-cn-beijing.aliyuncs.com/20220504162911.png)

   首先必须明确一点的是客户端和服务端都可以主动发起关闭连接的动作，这里以客户端首先发起为例。

   - 开始时客户端和服务端都处于ESTABLISHED的状态；
   - 第一次挥手：客户端发送一个 FIN 报文（请求连接终止：FIN = 1），报文中会指定一个序列号 seq = u，并停止再发送数据，主动关闭 TCP 连接，这时TCP处于半关闭状态。客户端会转移到FIN-WAIT-1 状态，等待服务端的确认；
   - 第二次挥手：服务端收到 FIN 之后，会发送 ACK 报文，且把客户端的序号值 +1 作为 ACK 报文的序列号值，表明已经收到客户端的报文了，此时服务端转移到CLOSE-WAIT状态。客户端收到服务端的确认后，进入FIN-WAIT-2状态，等待服务端发出的连接释放报文段；
   - 服务端此时可能仍旧有数据向客户端发送；
   - 第三次挥手：如果服务端数据处理完毕，也想断开连接了，和客户端的第一次挥手一样，发送 FIN +ACK报文，且指定一个序列号，然后服务端到达LAST-ACK 的状态，等待客户端的确认；
   - 第四次挥手：客户端收到 FIN +ACK报文之后，一样发送一个 ACK 报文作为应答（ack = w+1），且把服务端传过来的ack值作为自己 ACK 报文的序号值（seq=u+1），此时客户端处于TIME-WAIT状态；
   - 服务端收到ACK报文，进入CLOSED状态。但是此时客户端并没有进入CLOSED状态，它需要经过2MSL（Maximum Segment Lifetime，报文在网络上存在的最长时间，RFC793中规定为2分钟）的时间，才会进入最后的CLOSED状态。这样做的原因是为了确保服务端接受到ACK报文，因为如果服务端没有接收到的话，服务端会重发FIN报文，客户端接受到FIN报文之后，就知道上次的ACK发送失败，就会再次发送ACK；
   - 由于处于TIME-WAIT时间比较长，所以很多系统会优化相关的参数，达到快速重用处于TIME-WAIT的socket的作用，这里需要考虑存在的风险。

为什么这里要使用四次挥手：

​	由于TCP连接的全双工特性，也就是任意一端在关闭了发送后仍然存在接受的功能，所以要想完全关闭，必须要双方同时确认都不发送了才可以，二次挥手能够释放一个方向的请求，四次挥手才能释放双方互相之间的请求。

# 3. 关于TCP/IP

作为计算机网络的核心，TCP/IP是任何学习计算机相关知识的人都需要了解的。

作为学习者其中最重要的一本书是TCP/IP详解，可以从下列地址下载：

> - 《TCPIP详解三部曲卷1：协议》 [百度云链接](https://pan.baidu.com/s/1uvlA8rgsdEw3PwWejZWKVg) 提取码：j2iq
> - 《TCPIP详解三部曲卷2：实现》 [百度云链接](https://pan.baidu.com/s/1dU5VNXfN0V9MIFsxVXw93g) 提取码：r1cl
> - 《TCPIP详解三部曲卷3》 [百度云链接](https://pan.baidu.com/s/13BDPDFfnaZXHdiGV0SuYKw) 提取码：zee6

以上地址来自于：[forthespada/CS-Books](https://github.com/forthespada/CS-Books)