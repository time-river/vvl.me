---
title: 路由 [2] —— 网络工具
date: 2017-02-25 21:35:50
categories: style
tags: ['data communications and networking', 'netwok layer', 'routing']
---

> 第二篇谈谈 Linux 下的网络工具吧。

## net-tools VS iproute2

其实有不少关于这二者的文章。简单地说：iproute2 替代了 net-tools。《精通 Linux 内核网络》这本书提到：net-tools 与内核的交互是由`ioctl`实现的，而 iproute2 则是`netlink`，后者提供了更强大的功能(其实我也不太懂...)。这图不错：
![net-tools vs iproute2](/images/17/02/Linux-Nettools-vs-Iproute2.png)

关于`ifconfig`， 曾翻译过一[文章](/2016/09/translation-ifconfig/)，将就着看吧。

这里主要介绍与路由有关的东西。

## route -n

`route`的输出与`route -n`并无太大不同，`-n`参数经常见到，指的是 numerical，数字化的表示。输出的具体含义不妨`man`一下，`OUTPUT`那有详细的解释。值得注意的是：

1. 还记得上篇中提到的`0.0.0.0`吗？掩码(Genmask)为此值则代表默认路由(default route)。
2. 网关(Gateway)地址为`*`则代表未设置，numerical 显示为`0.0.0.0`，为设置就代表分组不需要转发，发送者与接收者在同一个物理网络中咯。
3. 度量(Metric)，上篇也提到过，值越小优先级越高。

```bash
vvl@xdlinux:~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         202.130.120.254 0.0.0.0         UG    0      0        0 eth1
10.170.72.254   0.0.0.0         255.255.255.255 UH    0      0        0 ppp0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
202.130.120.124 0.0.0.0         255.255.255.224 U     0      0        0 eth1
```

这么理解上面的输出：

1. 转发技术采用了下一条方法、特定网络方法与特定主机方法来简化路由表项目；
2. 此主机有三个接口：eth1、ppp0、docker0；
3. 与三个物理网络直接相连：202.130.120.124/27、172.17.0.0/16、10.170.72.254/25，其余网络的分组都通过默认网关(202.130.120.254)来转发；
4. Flag 中的 U 代表此路由有效，G 代表采用此路由条目的分组需要经过网关转发， H 代表目标是一个主机而非一个网络。度量(Metric)都是最大——0，没有优先级的区别；
5. 至于 Ref 与 Use，没懂。

查看 IPv6 的路由信息使用`route -A inet6`，目前，显示的信息并看不太懂。

## ip

关于这个，也可以用`man`，`man ip`、`man ip addr`、`man ip route`...
还有，这[文档](http://linux-ip.net/html/index.html)写得挺好。

### ip addr show

`ip addr show`的输出内容包含了`ip link show`的内容。更多详细的信息参考这里：[链路层](http://www.dsm.fordham.edu/cgi-bin/man-cgi.pl?topic=ip-link&ampsect=8)、[网络层](http://www.dsm.fordham.edu/cgi-bin/man-cgi.pl?topic=IP-ADDRESS&ampsect=8)。找个实例解释下：

```bash
me@vvl:/etc/network$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:16:3c:7e:f5:54 brd ff:ff:ff:ff:ff:ff
    inet 104.160.173.87/27 brd 104.160.173.95 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 2001:470:f2aa:0:216:3cff:fe7e:f554/64 scope global mngtmpaddr dynamic
       valid_lft 2591954sec preferred_lft 604754sec
    inet6 2610:150:c004:9::54/108 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::216:3cff:fe7e:f554/64 scope link
       valid_lft forever preferred_lft forever
```

两个条目，接口设备分别为 lo 与 eth0，前两行的信息是与硬件相关的，即链路层信息。

1. LOOPBACK —— 环回地址；UP —— 设备启用；LOWER_UP —— 链路层的状态；BROADCAST、MULTICAST —— 广播、多播。
2. 接着是最大传输单元(Maximum transmission unit, MTU)、流量控制排队规则(the traffic control queueing discipline, qdisc)、所属的群组(group，/etc/iproute2/group 内存储了当前所有的群组，管理之用)、缓冲区传输队列长度(buffer transmit queue length, qlen)。
3. link/loopback 代表此接口为环回接口，而 link/ether 则是以太网(Ethernet)。
4. 紧跟这的是硬件地址，brd —— broadcast(广播)。有趣的是，全零有时候代表这个硬件地址待填充(比如 ARP 请求)、网络接口卡(NIC)不存或者有毛病，等等，总之就是硬件不正常？环回地址哪来的 NIC 呢？笑～全一代表是本地广播地址(只被当前网络中的主机接收)。

后面的都是网络层信息了。

1. inet —— IPv4，inet6 —— IPv6。
2. scope —— 作用范围。这个`ip route`的输出也能看到。取值：全局(global)、站点内部(site，IPv6 专用，已被废弃)、本设备(link)、主机内部(host)。
3. mngtmpaddr —— 我猜是 manage temporary addresses 的缩写，IPv6 专用。此标志代表这个地址是内核创建的一个临时地址，保护隐私用的。关键词：IPv6 Privacy Extensions 、[RFC3041](https://tools.ietf.org/html/rfc3041)。
4. dynamic —— IPv6 专用，代表该地址是动态的。关键词：IPv6 stateless address configuration。
5. 至于 eth0...
6. valid\_lft —— the valid lifetime of this address，preferred\_lft —— the preferred lifetime of this address。

### ip route

在谈这之前，需要写写 rt\_tables，文件的位置在 /etc/iproute2/rt\_tables。

#### rt_tables

> rt_tables简单来说就是通过给表的命名使得管理简单化。

我们都知道，分组如何传递是需要查找路由表的，但路由表唯一吗？答案是否定的。Linux 内核支持多个路由表，这些路由表由独一无二的非负整数 1 - 255 标记，0 号比较特殊。系统内置了以下的路由表：

* 255 —— 本地路由表(local)。由内核维护的一张特殊的路由表，不能由用户自行添加条目。
* 254 —— 主路由表(main)。如果没有指明路由所属的表，所有的路由都默认添加到这个表里。`route`命令操作的表即是此表。
* 253 —— 默认路由表(default)。也是一张特殊的路由表。然，并不知道特殊在哪，而且初始时它是空的。
* 0   —— unspec。据说操作此表会同时操作其他路由表。[摊手]，并不知道如何使用。

非负整数为路由表的优先级，数值越小优先级越高。其实一般只用到 local 与 main 两个路由表。比如查看本地路由表条目：

```bash
me@vvl:~$ ip route list table local
broadcast 104.160.173.64 dev eth0  proto kernel  scope link  src 104.160.173.87
local 104.160.173.87 dev eth0  proto kernel  scope host  src 104.160.173.87
broadcast 104.160.173.95 dev eth0  proto kernel  scope link  src 104.160.173.87
broadcast 127.0.0.0 dev lo  proto kernel  scope link  src 127.0.0.1
local 127.0.0.0/8 dev lo  proto kernel  scope host  src 127.0.0.1
local 127.0.0.1 dev lo  proto kernel  scope host  src 127.0.0.1
broadcast 127.255.255.255 dev lo  proto kernel  scope link  src 127.0.0.1
```

本地路由表说明了此主机直接与哪些网络相连，以及寻路时路由表对特殊地址的处理方式(比如本地地址与广播地址)。第一列说明此路由信息是针对本地地址还是本机上托管的广播地址，随后是 IP 地址，接着是目的地可通过哪个接口到达。proto 是 protocol 的缩写，kernel 的含义是这个条目是由内核自动配置的，     `/etc/iproute2/rt_protos` 中有更多的选项。scope 已提到。src，source，暗示内核使用此接口出站的数据包选择哪个 IP 地址作为源地址，经常在一设备多 IP 地址的情况下使用。

`ip route`与`ip route show main`等价，显示主路由表的信息。

```bash
me@me-U24:~$ ip route
default via 10.177.255.254 dev wlp1s0  proto static  metric 600
10.177.0.0/16 dev wlp1s0  proto kernel  scope link  src 10.177.96.99  metric 600
169.254.0.0/16 dev docker0  scope link  metric 1000
172.17.0.0/16 dev docker0  proto kernel  scope link  src 172.17.0.1
```

第一条代表默认路由，`via 10.177.255.254`的含义是不满足其他条目的数据包传递给网关`10.177.255.254`，由它进行转发。第一条与第二条的度量都是 600，目的地在`10.177.0.0/16`网络中的数据包到底采用哪个条目呢？我也不知道...

## 其他

至于 IPv6 的种种，看不太懂，不写了。对了，还有一种异常坑爹路由条目出现在 IPv6 路由表中：

```bash
root@OpenWrt:~# ip -6 route
default from 2001:250:1006:dff0::/64 via fe80::96db:daff:fe3e:8fcf dev pppoe-wan  proto static  metric 512
...
```

## 参考资料

[net-tools VS iproute2](http://linoxide.com/linux-command/use-ip-command-linux/)  
[Guide to IP Layer Network Administration with Linux](http://linux-ip.net/html/index.html)  
[ip link 手册](http://www.dsm.fordham.edu/cgi-bin/man-cgi.pl?topic=ip-link&ampsect=8)  
[ip addr 手册](http://www.dsm.fordham.edu/cgi-bin/man-cgi.pl?topic=IP-ADDRESS&ampsect=8)  
['ip addr' command shows 'UP' even there is no address associated with that interface](http://unix.stackexchange.com/questions/153136/ip-addr-command-shows-up-even-there-is-no-address-associated-with-that-inter)  
[iproute2 Link group management](http://baturin.org/docs/iproute2/#Link%20group%20management)  
[What does “proto kernel” means in Unix Routing Table?](http://stackoverflow.com/questions/10259266/what-does-proto-kernel-means-in-unix-routing-table)  