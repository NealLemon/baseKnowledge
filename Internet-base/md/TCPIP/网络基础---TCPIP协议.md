# 网络基础---TCP/IP协议

   笔记只是通过学习以及网上的博文对自己需要的内容进行梳理和记录。大部分的基础知识是来自[一篇文章带你熟悉 TCP/IP 协议（网络协议篇二)](https://www.jianshu.com/p/9f3e879a4c9c)。

## 计算机结构分层

### OSI模型和TCP/IP概念模型

结构分层.png

## TCP/IP 基础

#####  IP 地址、端口号、协议号标识网络中的唯一的进程。

### 1.IP基础

- IP协议:无连接的网络协议
  - IP（IPv4、IPv6）相当于 OSI 参考模型中的第3层——网络层。网络层的主要作用是“实现终端节点之间的通信”。这种终端节点之间的通信也叫“点对点通信”。
- IP地址: 为了识别通信对端,类似于地址的识别码进行标识
  - IP 地址用于在“连接到网络中的所有主机中识别出进行通信的目标地址"。
  - 负责将数据包路至目标网络地址。



### 2.TCP协议

- 有连接的，可靠的，基于字节流的传输层通信协议。
- 充分地实现了数据传输时各种控制功能，可以进行丢包时的重发控制，还可以对次序乱掉的分包进行顺序控制。
- 能够实现高可靠性的通信（ 主要通过检验和、序列号、确认应答、重发控制、连接管理以及窗口控制等机制实现）。



### 3.UDP协议

- 利用 IP 提供面向无连接的通信服务。
- 传输途中出现丢包，UDP 也不负责重发。
- 获取数据包没有顺序纠正功能。



### 4.TCP与UDP协议



|  特点  |                TCP                 |               UDP                |
| :----: | :--------------------------------: | :------------------------------: |
|  连接  |               有连接               |              无连接              |
| 可靠性 | 可靠（“顺序控制”或“重发控制”机制） | 不可靠（不能保证消息一定会到达） |
| 有序性 |         有序（利用序列号）         |               无序               |
|  速度  |                 慢                 |                快                |
|  量级  |          重量级（20字节）          |        轻量级（8个字节）         |



### TCP/IP 进阶

### 1.TCP 三次握手（连接）

所谓三次握手是指建立一个 TCP 连接时需要客户端和服务器端总共发送三个包以确认连接的建立。

##### 控制位

- ACK:确认序号标志
- SYN:同步序列号，用于建立连接过程
- FIN：用于释放连接

##### 图解

三次握手.png



##### 文字解释

1. 第一次握手：客户端将标志位SYN置为1，随机产生一个值seq=J，并将该数据包发送给服务器端，客户端进入SYN_SENT状态，服务端进入LISTEN阶段，等待服务器端确认。
2. 第二次握手：服务器端收到数据包后由标志位SYN=1知道客户端请求建立连接，服务器端将标志位SYN和ACK都置为1，ack=J+1，随机产生一个值seq=K，并将该数据包发送给客户端以确认连接请求，服务器端进入SYN_RCVD状态。
3. 第三次握手：客户端收到确认后，检查ack是否为J+1，ACK是否为1，如果正确则将标志位ACK置为1，ack=K+1，并将该数据包发送给服务器端，服务器端检查ack是否为K+1，ACK是否为1，如果正确则连接建立成功，客户端和服务器端进入ESTABLISHED状态，完成三次握手，随后客户端与服务器端之间可以开始传输数据了。



### 2.TCP 四次挥手（终止连接）

四次挥手即终止TCP连接，就是指断开一个TCP连接时，需要客户端和服务端总共发送4个包以确认连接的断开。

##### 图解

四次挥手.jpg



##### 文字解释

1. 客户端进程发出连接释放报文，并且停止发送数据。释放数据报文首部，FIN=1，其序列号为seq=u（等于前面已经传送过来的数据的最后一个字节的序号加1），此时，客户端进入FIN-WAIT-1（终止等待1）状态。
   TCP规定，FIN报文段即使不携带数据，也要消耗一个序号。
2. 服务器收到连接释放报文，发出确认报文，ACK=1，ack=u+1，并且带上自己的序列号seq=v，此时，服务端就进入了CLOSE-WAIT（关闭等待）状态。TCP服务器通知高层的应用进程，客户端向服务器的方向就释放了，这时候处于半关闭状态，即客户端已经没有数据要发送了，但是服务器若发送数据，客户端依然要接受。这个状态还要持续一段时间，也就是整个CLOSE-WAIT状态持续的时间。
3. 客户端收到服务器的确认请求后，此时，客户端就进入FIN-WAIT-2（终止等待2）状态，等待服务器发送连接释放报文（在这之前还需要接受服务器发送的最后的数据）。
4. 服务器将最后的数据发送完毕后，就向客户端发送连接释放报文，FIN=1，ack=u+1，由于在半关闭状态，服务器很可能又发送了一些数据，假定此时的序列号为seq=w，此时，服务器就进入了LAST-ACK（最后确认）状态，等待客户端的确认。
5. 客户端收到服务器的连接释放报文后，必须发出确认，ACK=1，ack=w+1，而自己的序列号是seq=u+1，此时，客户端就进入了TIME-WAIT（时间等待）状态。注意此时TCP连接还没有释放，必须经过2∗∗MSL（最长报文段寿命）的时间后，当客户端撤销相应的TCB后，才进入CLOSED状态。
6. 服务器只要收到了客户端发出的确认，立即进入CLOSED状态。同样，撤销TCB后，就结束了这次的TCP连接。可以看到，服务器结束TCP连接的时间要比客户端早一些。



#### 相关问题

1. 为什么TIME_WAIT状态需要经过2MSL(最大报文段生存时间)才能返回到CLOSE状态？

- 确保足够的时间让对方收到ACK包，如果被动方关闭的没收到ACK，就会触发被动方再次发送ACK包，一来一去正好是2MSL。
- 避免连接混淆。

  2.关闭的时候需要四次握手？

-   因为TCP是全双工（允许数据在两个方向上同时传输即客户端可以发送数据给服务端，服务端可以发送数据给客户端）。服务器和客户端都需要FIN报文和ACK报文。因此两方各需两次挥手即可，只不过其中一个是被动方。



### 3.滑动窗口

##### 图解

滑动窗口.png

##### 文字解释

- 上图中的窗口内的数据即便没有收到确认应答也可以被发送出去。不过，在整个窗口的确认应答没有到达之前，如果其中部分数据出现丢包，那么发送端仍然要负责重传。为此，发送端主机需要设置缓存保留这些待被重传的数据，直到收到他们的确认应答。
- 在滑动窗口以外的部分包括未发送的数据以及已经确认对端已收到的数据。当数据发出后若如期收到确认应答就可以不用再进行重发，此时数据就可以从缓存区清除。
- 收到确认应答的情况下，将窗口滑动到确认应答中的序列号的位置。这样可以顺序地将多个段同时发送提高通信性能。这种机制也别称为滑动窗口控制。






