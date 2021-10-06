---
title: 路由 [1] —— 概述
date: 2017-02-25 19:00:08
categories: style
tags: ['data communications and networking', 'netwok layer', 'routing']
---

> 上学期，上课、上机、做实验、电装实习……忙得团团转，所以这内容只能慢慢补咯。

这里是书上一些概念的总结。

## 传递(delivery)

传递是指在网络层控制下用底层网络对一个分组进行处理的方法。分为__直接传递(direct delivery)__与__间接传递(indirect delivery)__。

直接传递就是分组的最终目的端主机与发送方都连接在同一个物理网络上。（两种情况：1.源端与目的端在同一个物理网络上；2.传递是在最后一个路由器和主机之间进行时。）

如果目的主机与发送方不在同一个网络上，分组就是间接传递。

一个传递永远包含一个直接传递和零或多个间接传递，最后的传递总是直接传递。

![delivery](/images/17/02/delivery.png)

## 转发(forwarding)

转发是指将一个分组传递到下一个站点的方法。

转发技术：

1. 下一条与路由方法
  * 下一条方法(next-hop method)：路由表只保留一条下一跳的地址，而不保留完整的路由信息）。
  * 路由方法(route method)：在路由表中保留完整路由信息。  
![next-hop method and route method](/images/17/02/next-hop-and-route.png)

2. 特定网络方法与特定主机方法
  * 特定网络方法(network-specific method)：不是对连接在同一个物理网络上的所有主机都有一个项目，而是仅用一个项目来定义这个目的网络本身的地址。  
  ![host-specific method and network-specific method](/images/17/02/host-specfic-and-network-specific.png)
  * 默认方法(default method)。如图，路由器 R1 用来将分组转发到连接网络 N2 的主机；网络的其余部分则使用路由器 R2。主机 A 可以仅使用一个称为__默认__的项目(通常定义网络地址为 0.0.0.0)代替因特网中的所有主机。
  ![default method](/images/17/02/default-method.png)

转发过程：

1. 无类地址转发
  * 假定主机和路由器使用无类寻址，在无类寻址中，一个路由表至少有 4 列：掩码、网络地址、__下一跳地址__、接口。分组到达路由器后，第一列中的掩码作用于分组中的地址，得到网络号，采用__最长掩码匹配(longest mask matching)原则__来匹配第二列中的网络号，进而将下一跳地址和接口号传递到 ARP 做进一步处理。
    ![forwarding process](/images/17/02/forwarding-process.png)
2. 地址聚合，减少路由表项
3. 最长掩码匹配
4. 层次结构路由选择及地理路由选择

## 单拨路由协议

一个路由器通常连接多个网络，当它接收到分组时，应当将分组转发到哪个网络呢？这个决定是基于最优化原则而作出的。一种方法是给定通过网络指定代价，称这个代价为__度量(metric)__。

因特网有两大类路由选择协议：内部网关协议IGP (Interior Gateway Protocol)——在一个自治系统内部使用的路由选择协议，外部网关协议EGP (External Gateway Protocol)。
<small>自治系统(autonomous system)是一个单一管理机构管辖下的一组网络和路由器，在自治系统内部的路由选择称为域内路由选择(interadomain routing)，自制系统之间的路由选择称为域间路由选择(interdomain routing)。</small>
![autonomous system](/images/17/02/autonomous-system.png)

![routing protocol](/images/17/02/routing-protocol.png)

__距离向量路由选择__
在记录向量路由选择中，任何两个节点之间路由最低代价是最小距离的路径。开始时，每个节点__仅知道与它直接连接的邻站的距离__，每个节点与它的临站周期性地或有变化时共享它的路由表，直到每一个节点达成一个稳定状态(采用 Bellman-Ford 算法实现)。
问题：两个节点的不稳定性、三个节点的不稳定性
[路由选择信息协议(Routing Information Protocol, RIP)](https://en.wikipedia.org/wiki/Routing_Information_Protocol)

__链路状态路由选择__
在链路状态路由选择中，区域中的每一个节点拥有该区域的全部拓扑结构(所有节点和链路的列表，他们如何连接包含类型、代价也就是度量和链路的连通或断开的情形)。根据这个拓扑结构，一个节点可用 Dijkstra 算法建立一个路由表。
[开方最短路径优先(Open Shortest Path First, OSPF)](https://en.wikipedia.org/wiki/Open_Shortest_Path_First)
要点：

1. 洪泛法向本自治系统中所有路由器发送信息
2. 发送与本路由器相邻的所有路由器的链路状态(本路由器都与哪些路由器相邻，以及该链路的度量)，只是该路由器知道的信息
3. 只有链路状态发生变化，才用洪泛法向所有路由器发送此信息

__路径向量路由选择__
[边界网关协议(Border Gateway Protocol, BGP)](https://en.wikipedia.org/wiki/Border_Gateway_Protocol)

## 多播路由选择协议

不会=_=
