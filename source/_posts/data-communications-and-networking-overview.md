---
title: 数据通信过程概述
date: 2016-09-23 13:25:41
categories: style
tags: ['data communications and networking']
---

> 数据通过网络可以从一个地方传送到另一个地方，这是数据通信的基本概念。但是，数据通信的大致过程又是怎样的呢？

## 数据通信系统的五个组件

一个数据通信(data communication)系统的组成是这样的：

![components of the data communication system](/images/16/09/components-of-the-data-communication-system.png)

* 报文(Message)
* 发送方(Sender)
* 接受方(Receiver)
* 传输介质(Transmission Medium)
* 协议(Protocol)

不难理解他们：数据通信系统需要有发送方与接受方，进行通信的信息叫做报文，传输过程中承载报文的物理通路叫做传输介质，而协议则是通信双方之间约定的一组规则。

拓展阅读：

1. [数据流](#data-stream)
2. [协议与标准](#protocols-and-standards)。

## 数据通信系统是如何工作的

在讨论数据通信系统是如何工作之前，先了解一下网络模型，网络模型是分层结构模型。至于分层结构，就像一封信件的旅程一样：

![send mail](/images/16/09/send-mail.png)

同样，将一份电子邮件从世界上的某个地点发送到另一个地点也可以这么划分。这就是网络层次的概念：发送方与接受方都有相同的层次，下层为上层提供服务而不关心上层的数据，每一层都实现了特定的功能并对上层隐藏了所有细节。

在两台机器之间，一台机器上的第 x 层与另一台机器的第 x 层通过对等协议(peer-to-peer protocol)来进行通信，这种通信过程被成为对等过程(peer-to-peer process)。上层与下层之间的数据传递，是通过相邻两层的接口(interface)来实现的，通过对上层数据的封装(encapsulation)——添加头部(header)与尾部(trailer)，下层向上层提供信息和服务。

分层结构模型中重要的一种是_开放式系统互联通信参考模型_(Open System Interconnection Reference Model，缩写为 OSI)，简称为 OSI 模型(OSI model)。尽管未在因特网中被采用，可理解它也有助于理解其他网络模型。

### OSI 模型

OSI 模型国际标准化组织(ISO)提出的一个试图使各种计算机在世界范围内互连为网络的标准框架。它由七个有序的层组成，从下到上依次为：物理层(Physical Layer)、数据链路层(Data Link Layer)、网络层(Network Layer)、传输层(Transport Layer)、会话层(Session Layer)、表示层(Presentation Layer)、应用层(Application Layer)。

![OSI model](/images/16/09/osi-model.png)

__物理层负责比特(bit)从一个节点到另一个节点的不一定可靠的传递。__它定义了那些在物理介质上传输比特流所必须的具备的功能，并不对数据做任何改动。

__数据链路层负责数据帧(frame)从一跳(节点)到下一跳(节点)传递。__它将物理层的传输通道变成可靠的链路(链路是将数据从一台设备传输到另一台设备的通信通路)，从而可以将物理层的数据无差错地传递给上层。数据链路层是跳到跳传递(hop-to-hop delivery)，或者称为节点到节点传递(node-to-node-delivery)。

__网络层负责将分组(packet)从源地址传递到目的地址。__在分组传递的路线上，可能会存在多个链路连接到同一个节点上的情况，如何选择链路便是网络层的任务之一。它分发的数据单元并不一定可靠。

![source to destination delivery](/images/16/09/source-to-destination-delivery.png)

__传输层负责一个报文从一个进程到另一个进程的传递。__对 TCP 协议来说，数据单元为报文段(segment)；UDP 则是数据报(datagram)。网络层保证了各个分组的源端到目的端传递(source-to-destination delivery)，所有传输遗留的问题，则由传输层处理，确保整个报文无差错并按顺序到达目的地。这个目的地也不仅是从一台计算机到另一台，而是从一台计算机上的特定进程传递到另一台计算机上的特定进程。

__会话层负责对话控制和同步。__它的数据单元为数据(data)。主机间通讯，它管理应用程序之间的会话。

__表示层负责翻译、加密和压缩数据。__数据单元亦为数据。

__应用层负责向用户提供服务。__数据单元同数据。它针对特定应用规定各层协议、时序、表示等，进行封装 。

![summary of layer's ](/images/16/09/summary-of-layer's-functions.png)

### 网络通信的大致过程

OSI 模型虽好，TCP/IP 协议族(TCP/IP protocol suite)却是事实上的标准。这里并不打算从 TCP/IP 协议族的角度来细说其网络通信过程，只是一个关于它的概述，而且从 OSI 模型角度理解的。假设已对[网络的物理结构](#physical-topology)有所了解。

__采用 TCP/IP 协议族的互联网使用物理地址(physical address)、逻辑地址(logical address)与端口地址(port address)来唯一标识网络中一台主机上应用的一个进程。__

物理地址，也称为链路地址(link address)，定义在数据链路层中，是局域网或广域网定义的节点地址。数据链路层的数据单元为帧，物理地址是帧头部的一部分。下图总线结构的局域网(广播)中，物理地址为 10 的节点向物理地址为 87 的节点发送了一个帧，帧在两个方向(左和右)传播，凡是物理地址不是 87 的节点都丢弃这个帧。具有目的物理地址的节点发现帧中的目的地址与自己的相同，就检验该帧，去掉头部和尾部，提取出数据部分向上传递。

![physical address](/images/16/09/physical-address.png)

物理地址不适用于互联网环境，这是因为不同网络可能有不同的地址格式，而且物理地址在广域网中并不是唯一的。而互联网中的每台主机需要有唯一的标记，而不管下面的物理地址。为此设计了逻辑地址，或者叫 IP 地址(IP address)，唯一定义了连接到因特网的一台主机。

下图表示了用两个路由器连接三个局域网所组成的互联网的一部分：每台设备(路由器和计算机)都有一对地址(逻辑地址与物理地址)用作连接。此时，每台计算机仅连接到一条链路，因此它只有一对地址。但路由器连接了三个网络(图中仅画出了两个)，因此每一个路由器有三对地址，每个连接用一对。对每一个连接，每个路由器必须有一个独立的物理地址，却不需要一个明确的逻辑地址(这与路由选择有关)。

![logical address](/images/16/09/logical-address.png)

逻辑地址 A 和物理地址 10 的计算机(Sender)向逻辑地址 P 和物理地址 95 的计算机(Receiver)发送了一个分组。发送方在网络层封装这个分组并添加两个逻辑地址 A 和 P。在该分组传递之前，网络层必须寻找下一条的物理地址。网络层查阅它的路由表找到下一条的逻辑地址为 F(Router 1)，并发现它的物理地址为 20，网络层将这个物理地址传给数据链路层，接着在该层用目的物理地址 20 和源物理地址 10 封装改分组为帧。
该帧被 LAN 1 中的所有设备接收到，但只有 Router 1 在收到之后没有废弃它。Router 1 拆分该帧为分组，并读出逻辑地址 P。由于目的逻辑地址与 Router 1 的逻辑地址并不相同，于是改帧被转发。路由器查询它自己的路由表并发现下一条(Router 2)目的的物理地址，于是创建该分组的一个新帧发送到 Router 2。源和目的的物理地址被改变，但源和目的的逻辑地址依旧。
Router2 有相似的过程：物理地址将被改变并向目的计算机发送了一个新帧。当目的逻辑地址与设备逻辑地址相同时，该分组拆封的数据传递给上一层。
这过程，虽然跳到跳时传递物理地址将改变，但逻辑地址始终保持不变。

尽管物理地址与逻辑地址保证了分组从源主机传递到目的主机，但进程到进程的通信才是因特网通信的目标。标识不同进程的方法便是使用端口地址，端口地址在通信过程中也是始终保持不变的。

![port address](/images/16/09/port-address.png)

两台计算机通过因特网进行通信，发送计算机(Sender)此时运行三个进程，使用三个端口地址 a、b 和 c，接收计算机(Receiver)运行两个进程，使用两个端口地址。Sender 中的进程 a 与 Receiver 中的进程 j 进行通信。传输层对来自上层(应用层)的数据添加两个端口地址 a 和 j，即源和目的地址封装在分组中，然后网络层封装逻辑地址 A 和 P，数据链路层封装了物理地址(并未在图中表示出)。

## 拓展阅读

### <a id="data-stream"></a>数据流

两台设备之间的通信方式分为：单工、半双工、全双工。

![data stream](/images/16/09/data-stream.png)

单工模式(simplex mode)下，通信是单方向的，两台设备只有一台能够发送，另一台只能接受。比如键盘和传统的显示器，键盘用来输入，显示器用来输出。

半双工模式(half-duplex mode)下，每台设备均能够发送与接收，但不能同时进行。比如一条双向单车道的道路，一辆汽车朝一个方向行驶，朝另一个方向的汽车只能等待。

全双工模式(full-duplex mode)，双方设备均能够同时发送与接收。

### <a id="protocols-and-standards"></a>协议与标准

协议，打个比方：两个中国人交流，使用的语言——汉语就可以认为是一种协议。协议规定了通信的内容、通信的方式和通信的时间。

数据通信标准分为两类：

* 事实上的标准(de facto)
* 法定的标准(de jure)

他们的区别在于：事实标准未经组织团体承认，但已经在广泛使用中被接受。事实标准通常是由制造商在打算定义新的产品或技术的功能时最初建立的。比如 TCP/IP 协议族。而法定标准则由官方认可的团体制定，比如 OSI/RM。

协议是规则的同义词，标准是经过协商达成一致的规则。我的理解是标准有一种，但使用标准实现的协议却可以有多种。好比汉语中中的方言：汉语的规则有一种，却有多种方言。

### <a id="physical-topology"></a>网络的物理结构

网络是通过链路连接在一起的两台或两台以上的设备，链路是将数据由一台设备传输到另一台设备的通信通路。要想建立通信，两台设备必须已某种方式在同一时间连接到同一链路上：

* 点对点(point-to-point)连接提供两台设备之间专用的链路
* 多点连接(multipoint connection)是两台以上设备共享单一链路的情形

![connections](/images/16/09/connection.png)

物理拓扑结构(physical topology)指的是网络物理上分布的方式。有这么几种：网状(mesh)、星型(star)、总线(bus)、环状(ring)。

在网状拓扑结构中，各台设备之间都有一条专用的点到点链路：

![mesh](/images/16/09/mesh.png)

星型拓扑结构中，每台设备拥有一条仅与中央控制器连接的点到点专用链路。

![star](/images/16/09/star.png)

总线拓扑结构是多点连接，由一条较长的线缆作为主干来连接网络上的所有设备。

![bus](/images/16/09/bus.png)

环状拓扑结构中的每台设备只与其两侧的设备有一条专用的点到点连接。信号以一个方向在环中传输，从一台设备到达另一台设备，直到其到达目的设备。

![ring](/images/16/09/ring.png)

网络也可以是混合型拓扑结构。

网状结构比起其他网络拓扑结构有诸多优点。点到点的连接不仅保证了数据的传输速度，而且增强了系统的健壮性——即使一条链路不可用也不会使得整个系统不可用；又因特定的接受方才能收到数据，保证了数据的安全性；还便于故障识别和隔离。但过多的线缆与 I/O 端口也带来了诸多问题。
与网状结构相比，其他网络拓扑需要的线缆与 I/O 端口便少得多。
星型拓扑结构中中央控制器常称集线器(hub)，它不允许设备之间有直接的通信量，集线器扮演数据交换的角色。它最大的缺点也在于集线器，若集线器出了毛病，整个网络便会瘫痪。局域网(Local Area Network, LAN)便使用此拓扑结构。
总线拓扑结构的优势包括安装简易，节点由引出线和分接头连接到总线电缆，但它所能支持的分接头数目以及这些分接头之间的距离是有限的，而且难以进行重新连接和错误隔离。早期局域网曾使用过它。
因为自身的原因，环状拓扑结构易于安装和配置，但单向通信量可能是不利因素。
