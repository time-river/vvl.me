---
title: 路由 [3] —— 策略路由
date: 2017-02-26 20:35:47
categories: style
tags: ['data communications and networking', 'netwok layer', 'routing']
---

一开始就打算写写与策略路由有关的东西，前两篇的内容都在为此篇做准备。

原则型路由(policy-based routing, PBR)也称为策略路由(policy route)，是一种决定路由的方式，由网络管理者决定路由原则，再根据这些原则来决定路由。比如如下的两个场景：对所有来自网络 A 的数据包，选择 X 路径；其他的选择 Y 路径；对所有 TOS 为 A 的包选择路径 F，其他选择路径 K。这些路由策略的实现便用到了策略路由。

## 静态路由与策略路由的区别

为特定的网段指定特定的传输路径，可以采用静态路由的方式。这种办法针对少量的规则还可以轻松应对，但规则一旦增加，麻烦也就接踵而至，网段地址不断变化就必须及时更新路由表，否则...如果可以根据用户访问进来的路径设定策略路由就会方便很多，而策略路由就是为此而生。

## 策略路由的种种

策略路由的实现需要使用多路由表(Multiple Routing Tables)，还需要使用规则(rule)。

规则是策略性的关键性的新的概念。我们可以用自然语言这样描述规则，例如我门可以指定这样的规则：

* 规则一：“所有来自 192.16.152.24 的 IP 包，使用路由表 10， 本规则的优先级别是 1500” —— `1500:  from 192.16.152.24 lookup 10`
* 规则二：“所有的包，使用路由表 253，本规则的优先级别是 32767” —— `32767:  from all lookup 32767`

我们可以看到，规则包含3个要素：

* 什么样的包，将应用本规则（所谓的 SELECTOR，可能是 filter 更能反映其作用）；
* 符合本规则的包将对其采取什么动作（ACTION），例如用那个表；
* 本规则的优先级别。优先级别越高的规则越先匹配（数值越小优先级别越高）。

查看当前规则：

```bash
me@me-U24:~$ ip rule
0:  from all lookup local
32766:  from all lookup main
32767:  from all lookup default
```

## 一个实例

![routing diagram](/images/17/02/routint-diagram.svg)
网络拓扑图如上所示，达成以下目标：网段 192.168.2.0/24 经过路由 R2、R3 到达 R1 (路径为蓝色虚线)，而网段 100.64.2.0/24 则仅经过路由 R2 便到达 R1 (路径为绿色实线)。

### 网络实现

环境为 VirtualBox 中配置的五台虚拟机(设置启用网络->网卡->高级->混杂模式->允许虚拟电脑)。为了让这些设备互通，需要配置路由信息，静态路由太烦，动态路由～找到了 [Quagga](https://en.wikipedia.org/wiki/Quagga_(software)) 这软件用于配置动态路由。

> Quagga 是一个网络路由软件套件，为 Unix 平台，特别是 Linux，Solaris，FreeBSD 和 NetBSD 提供开放最短路径优先（OSPF），路由信息协议（RIP），边界网关协议（BGP）和 IS-IS 的实现。
> Quagga体系结构包括一个核心守护进程—— zebra，它充当底层 Unix 内核的抽象层，Quagga 客户端利用 Unix 系统或 TCP 协议与 Zserv API 进行通信。正是这些Zserv客户端通常实现一个路由协议和通信路由更新到斑马守护进程。现有的Zserv实现是：
> {% raw %}
<table id="daemons">
<tr><th>IPv4</th><th>IPv6</th></tr>
<tr><td class="daemon" colspan="2">zebra</td><td>- kernel interface, static routes, zserv server</td></tr>
<tr><td class="daemon">ripd</td><td class="daemon">ripngd</td><td>- RIPv1/RIPv2 for IPv4 and RIPng for IPv6</td></tr>
<tr><td class="daemon">ospfd</td><td class="daemondev">ospf6d</td><td>- OSPFv2 and OSPFv3</td></tr>
<tr><td class="daemon" colspan="2">bgpd</td><td>- BGPv4+ (including address family support for multicast and IPv6)</td></tr>
<tr><td class="daemondev" colspan="2">isisd</td><td>- IS-IS with support for IPv4 and IPv6</td></tr>
<tr><td colspan="2"></td><td style="font-size:75%">externally, not part of Quagga package:</td></tr>
<tr><td class="daemonunm">olsrd</td><td></td><td>- OLSR wireless mesh routing through <a href="http://git.nowhere.ws/?p=olsrd-zclient.git;a=summary">a plugin</a> for <a href="http://www.olsr.org/">olsrd</a></td></tr>
<tr><td class="daemonunm" colspan="2">ldpd</td><td>- MPLS Label Distribution Protocol (<a href="https://github.com/rwestphal/quagga-public/tree/mpls/ldpd">forked from OpenBSD ldpd</a>)</td></tr>
<tr><td class="daemonunm" colspan="2">bfdd</td><td>- Bidirectional Forwarding Detection</td></tr>
</table>
    <style type="text/css">
	table#daemons th {
		background-color: #ccccff;
		padding:0pt 9pt;
	}
	table#daemons td {
		padding:0pt 5pt;
	}
	table#daemons td.daemon {
		background-color: #e3e3ff;
		text-align:center;
		padding:0pt 4pt;
		border:1px solid #9999cc;
	}
	table#daemons td.daemondev {
		background-color: #f2f2ff;
		text-align:center;
		padding:0pt 4pt;
		border:1px dotted #9999cc;
	}
	table#daemons td.daemonunm {
		background-color: #f7f7ff;
		text-align:center;
		padding:0pt 4pt;
		border:1px dashed #ccccff;
	}
    </style>
{% endraw %}

必然使用简单的 RIP 协议啦，要注意的是 __ripng only support for IPv6__ ...先弄好这两个文件：

```conf
# /etc/quagga/zebra.conf
hostname net
password net
enable password net

# /etc/quagga/ripd.conf
hostname net
password net
enable password net
```

`#ripd && zebra` 启动之， `ssh net@127.0.0.1 port 2602` 连接到 ripd，配置；）后续的教程就不少了。弄好后，Device 1 与 Device 2 到达 R1 时经过的节点都只有 R2。

### 策略路由

```bash
root@debian~#echo "200 custom" >> /etc/iproute2/rt_tables #添加新路由表项至 rt_tables
root@debian~#ip rule add from 192.168.2.0/24 lookup custom #指定源为网段 192.168.2.0/24 的分组查找路由表 custom
root@debian~#ip route add default via 100.64.0.3 table custom #设置此路由表中的默认路由
```

目标达成，路径如拓扑图所示，撒花。

## 结语

假期无所事事，只学会了它...大囧。原意是利用这些天看一看 Linux 网络栈的实现，[捂脸]，太高看自己了，真心真心看不全懂。看了百多页的《TCP/IP详解 卷2：实现》，感觉没现代的源码参考，遂将《精通Linux内核网络》《Linux环境编程：从应用到内核》买来，加上其他一些资料的参考， 堪堪弄懂了网络通信的大致的流程，然，感觉并没有啥用。

其实，有这个想法，还不是因为 BBR 这个拥塞控制算法嘛？想弄懂它，移植它到其他内核。现在只能这样咯，新学期开始，好好学习编译原理。网络的后续，有时间、有更深层次的领悟后继续写。

喔，写完之后，突然想起一个东西：__主机填写网关后，路由表上表现的信息就是增加一默认路由条目__...

## 参考资料

[静态路由和策略路由的配置实践](https://wsgzao.github.io/post/static-routes/)