《网络是怎样连接的》


## 设备：集线器/路由器/交换机
发送者发出的消息首先经过子网中的集线器，转发到距离发送者最近的路由器上。
集线器:工作在第一层，原理是向所有端口进行广播，后来被交换机取代。
交换机工作在第二层，原理是根据Mac地址有选择地转发。
而路由器则工作在第三层，能根据IP地址转发到不同的网络。

## Socket库
也是一种库，其中包含的程序组件可以让其他的应用程序调用操作系统的网络功能，举个例子，解析器就是这个库中的其中一种程序组件。
解析器负责收发DNS请求，真正收发数据是交给OS来做的。所以：所有的网络io都要交给操作系统来执行，其中又会涉及到用户态和内核态间的切换。
Linux里这些操作系统自带的库都会涉及到系统调用，程序会从用户态切到内核态。

- 套接字创建完成后，协议栈会返回一个描述符，应用程序会将收到的描述符存放在内存中。
Linux里"一切皆文件"，哪怕网络连接也会被视作文件。建立socket后会返回一个fd，表示文件描述符。fd是一个整数，唯一地标识着操作系统现在正在处理的文件。fd理论上是有限的，所以用完后要close( )

- 但描述符是和委托创建套接字的应用程序进行交互时使用的，并不是用来告诉网络连接的另一方的，因此另一方并不知道这个描述符
文件描述符是计算机内部自己用的，用来标识具体是哪个套接字要办业务。至于发数据就委托给了协议栈，传回的数据再交给协议栈转发过来。因此客户端和服务器端的fd都是各用各的，互不关心。

- 既然文件描述符只在内部使用，那怎么跟对方的套接字建立联系?
那就用端口就好了，只要端口对，就能联系到对应的套接字。只要指定了事先规定好的端口号，就可以连接到相应的服务器程序的套接字。
这是因为客户端在创建套接字时，协议栈会为这个套接字随便分配一个端口号。接下来，当协议栈执行连接操作时，会将这个随便分配的端口号通知给服务器。
客户端的端口号随机分配，服务器的端口号事先约定。

- 由于套接字中已经保存了已连接的通信对象的相关信息，所以只要通过描述符指定套接字，就可以识别出通信对象
在编程实践中，socket库里有一个结构体专门存fd(文件标识符)、端口、ip、协议类型。这些就是所谓的"已连接的通信对象的相关信息"。

- 调用read时需要指定用于存放接收到的响应消息的内存地址，这一内存地址称为接收缓冲区
写入数据时若不能一次性完成，就得用缓冲区(高频，划重点)。数据先写到缓冲区，然后再memcpy到真正要写的地方。在c语言里就是char buf[maxBuffSize]；

## TCP/IP
### 数据的两台计算机之间连接了一条数据通道，数据沿着这条通道流动，最终到达目的地。
先建立可靠的TCP通道，确保能传输数据，再在通道里发起不可靠的http请求。

### TCP为什么是可靠的？
如果确认没有遗漏，接收方会将到目前为止接收到的数据长度加起来，计算出一共已经收到了多少个字节，然后将这个数值写入TCP头部的ACK号中发送给发送方。
能互相确认是TCP可靠的原因

### 重传机制
前置知识：ack(用来确认应答的，收到消息后回一个ack给对方，表示自己收到了)、syn(序列号，用来标记发送信息)、RTO(一个数值，超过这个数值没收到认为超时，通常用自适应的算法计算)
TCP采用这样的方式：确认对方是否收到了数据，在得到对方确认之前，发送过的包都会保存在发送缓冲区中。如果对方没有返回某些包对应的ACK号，那么就重新发送这些包。
工作流程
- 发送：发送端将数据封装成 TCP 数据段，为其分配一个序列号syn，发出去的同时启动一个定时器，时长为当前RTO。
- 等待：发送端等待接收端返回的 ACK 报文。如果在定时器超时前收到了ACK，说明数据已经成功到达。发送端停止定时器，继续发送数据。
- 重传：如果定时器超时仍未收到 ACK，发送端认为该数据段丢失或损坏，重新发数据并重置定时器。重传后，RTO 值通常会加倍，以避免频繁重传对网络造成更大的压力。
- 尝试：如果重传后仍然没有收到 ACK，发送端会继续重传，每次重传都会将 RTO 值加倍，直到达到最大重传次数或者连接被关闭。
  
Q：TCP超时重传的是什么，是一个tcp段，还是滑动窗口内的所有tcp段？

A：是一个tcp段，原因见上。

Q：TCP的序号为什么要随机初始化?

A：不随机的话会被攻击；会和之前的数据包搞混；同一台主机上有不同TCP连接，可能会搞混。

Q：快速重传怎么实现？

A：如果发送端接收到连续相同的ack号后，说明该ack号对应的数据丢包了，就重发一遍。

### 比较TCP和UDP
- 可靠性：TCP可靠，通过三次握手四次挥手、重传机制等保证数据能到达；UDP不可靠。
- 速度：TCP慢，因为维护更费力；UDP快。
- 应用：TCP(http、ftp、SMTP)；UDP(DNS查询、音视频直播、打游戏)。

### TCP的流量控制
TCP 使用滑动窗口机制实现流量控制。

1.请求方告诉服务器我的窗口为200字节，然后开始请求数据。

2.服务器先发送了80字节数据，当前还能发的就只剩200-80=120字节了。

3.服务器又发送了120字节数据，剩120-120=0字节，服务器就先停下。

4.请求方发送了确认报文，说明自己收到80字节数据了，服务器又有了80字节的余地，又能开始发送...如此往复

但其实窗口是不固定的，会受操作系统的缓冲区大小和应用程序是否繁忙决定。操作系统的缓冲区大小也就是char buf[maxBuffsize]中的maxBuffsize，是写死的；而应用程序有时候很繁忙，来不及读取缓冲区，很多数据就滞留在里面，这时请求方就会告诉服务器我的窗口变少了少发点。

### TCP的拥塞控制
- 慢启动
慢启动的意思就是一点一点的提高发送数据包的数量，如果一上来就发大量的数据，这不是给网络添堵。
发送方每收到一个ack，拥塞窗口的大小就会+1。
最开始拥塞窗口是1，能发一个。收到一个ack后拥塞窗口变成2。
现在能发2个，收到2个ack后拥塞窗口变成4。
现在能发4个，收到4个ack后拥塞窗口变成8。
所以慢启动是指数级别的。

- 拥塞避免
到达阈值后，拥塞窗口的变化规律改成：每收到一个ack，拥塞窗口的大小就会1/窗口大小。
比如8个ack来了后，每个确认近似于增加1/8，一共加了1，下一次就能发9个。
所以拥塞窗口从8变成了9，线性增长。

- 拥塞发生
这块要和上面的超时重传、快速重传一起看。
超时重传：阈值为拥塞窗口的一半，然后窗口设为1，曲线陡降。
快速重传：TCP认为不严重，窗口设为原来一半，然后阈值设为此时窗口的值，进入快速回复

- 快速回复
此时拥塞的根本问题是相同的ack导致快速重传，所以为了能尽快将丢失的数据包发出，拥塞窗口线性增加。
刚才为了缓解拥塞，先降低窗口；此刻为解决问题，再增加窗口。
一直等到收了新的ack后，拥塞窗口回到阈值，开始新一轮的拥塞避免。
快速重传和超时重传比起来，没有一夜回到解放前，整体还是在比较高的值，后面也有线性增长。

### 三次握手
"刚才服务器返回响应时将ACK比特设置为1，相应地，客户端也需要将ACK比特设置为1并发回服务器，告诉服务器刚才的响应包已经收到。当这个服务器收到这个返回包之后，连接操作才算全部完成"
三次握手。为什么不是两次或者四次？可能客户端发起连接请求后寄了，但服务器还不知道，一直在等待传输，浪费资源。多确认一次能有效避免该情况。那为什么不是四次？显然多确认一次就要多一份开销，我们认为三次就足够说明两台机器能通信了。

举个更具体的例子：
第一次握手：主机A发送位码为syn＝1，随机产生seq number=1234567的数据包到服务器，主机B由SYN=1知道，A要求建立联机。

第二次握手：主机B收到请求后要确认联机信息，向A发送ack number=(主机A的seq+1)，syn=1，ack=1，随机产生seq=7654321的包。

第三次握手：主机A收到后检查ack number是否正确，即第一次发送的seq number+1，以及位码ack是否为1，若正确，主机A会再发送ack number=(主机B的seq+1)，ack=1，主机B收到后确认seq值与ack=1则连接建立成功。

完成三次握手，主机A与主机B开始传送数据。

### 四次挥手
1. 服务端发给客户端，FIN=1

2. 客户端收到后，返回ACK号确认

3. 客户端标记结束后，会返回FIN=1表示自己这边结束了

4. 服务端返回ACK号，表示确认收到，知道我们结束了。

### time_wait发生的时间和持续时间
TIME_WAIT 状态发生在 TCP 连接关闭过程中，具体是主动关闭连接的一方在发送最后一个 ACK 报文后进入的状态。

TIME_WAIT 持续时间是2MSL(TCP 分段在网络中能够生存的最长时间)常见的取值有 30 秒、1 分钟等。设置 2MSL 为了确保：

- 最后一个 ACK 报文到达对端。如果这个ACK丢失，对方发现没收到，会重新发送FIN报文，处于 TIME_WAIT 状态的主动关闭方可以再次发送ACK。

- 避免已失效的连接请求报文段出现在后续的连接中。在 2MSL 时间后，网络中所有与该连接相关的报文都会消失，不会影响新的连接。

## 冷得不行的冷知识
#### 如今除了互联网并没有其他的网络了，因此Class的值永远是代表互联网的IN
很中二地想到暗网/军方用的网络
#### IP地址是保存在A记录中的，而邮件服务器则是保存在MX记录中的

#### 协议栈并不是一收到数据就马上发送出去，而是会将数据存放在内部的发送缓冲区中，并等待应用程序的下一段数据

#### MTU表示一个网络包的最大长度，在以太网中一般是1500字节
呃好熟悉，好像考计网时做过相似的计算题。MTU设置太小会导致效率低，太大又好像会收不到？

#### ACK号的等待时间（这个等待时间叫超时时间）
原来平时说的超时时间就是ack号的等待时间。

#### 路由表
这张表里面记录了每一个地址对应的发送方向，也就是按照头部里记录的目的地址在表里进行查询，并根据查到的信息判断接下来应该发往哪个方向

#### 秩序严明
MAC头部、IP头部、TCP头部、数据块

#### IP对应网卡
IP地址实际上并不是分配给计算机的，而是分配给网卡的，因此当计算机上存在多块网卡时，每一块网卡都会有自己的IP地址。
