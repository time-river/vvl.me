---
title: 在Linux上拨号上网(PPPoE)
date: 2017-09-30 21:05:55
categories: style
tags: 'network'
---

> 16年10月便有写这篇文章的想法，还记得那时候刚刚学完PPP协议，已经快一年啦。
> 这篇文章同时发布在西电开源社区的Wiki上面[在Linux上拨号上网(PPPoE)](https://wiki.xdlinux.info/network/PPPoE)。

这篇文章用来指导初学者设置ADSL Internet Connection，即拨号上网。若ISP（Internet Service Provider）支持IPv6网络，则也可以通过IPv6接入因特网。

## 简介

上世纪末，因特网建立在电话线之上，电话线传输的是带宽有限的模拟信号。不管网络接口卡（Network Interface Controller，简称NIC，一般称为网卡）发送的是数字信号还是模拟信号，电话线都无法提供相应的频段与带宽，因此出现了猫。Modem（俗称猫）就是用来解决上述需求的——压缩带宽、搬移频谱，以便通过电话线网络互联。

ADSL（Asymmetric Digital Subscribe Line，非对称数字用户线） Internet Connection，俗称拨号上网。实际上这种上网方式几乎不存在了，因为它的使用完全依赖电话网络，而且因为技术原因，上下行带宽并不对等。

拨号上网现在泛指一种上网的方式：ISP提供用户名（username）与密码（passowrd），用户使用某些软件认证身份、配置好网络，进而可以使用因特网。这些软件采用的是PPPoE（Point-to-Point Protocol Over Ethernet，以太网上的点对点协议）协议，它是一个允许在以太网广播域中的两个以太网接口间创建点对点隧道的协议。它可以认为是在PPP协议的基础上进行了封装了，以便能够利用以太网协议进行数据传输，PPP协议则提供了身份认证与网络配置。

Note：

* 本质上，传输数字信号等价于传输模拟信号，这点可以在学习《信号与系统》这门课后会理解。
* ADSL 的原理、不对称原因在《数据通信与网络》这门课中会提到。
* PPP协议同样在《数据通信与网络》中提到。

## PPPoE过程

PPPoE过程参考维基百科[PPPoE](https://zh.wikipedia.org/wiki/PPPoE)。

## Linux网络管理工具

Linux与网络设置相关工具有两套：Network与NetworkManager。

Network是iproute-*bundle*系列的集合，比如*iproute2*、*ppp*。*ifupdown*是对此系列工具的一个封装。相应的配置文件，DEB系列的发行版位于`/etc/etc/network/`目录下，RPM系列发行版在`/etc/sysconfig/network`目录下。可以`$man interfaces`或`$man ifcfg`查看更多信息。

NetworkManager有两个组成部分：

1. NetworkManager守护进程，其为实际管理连接并汇报网络状态及变更的软件。
2. 多种不同外观的图形前端，包含了GNOME Shell、GNOME Panel、KDE Plasma Workspaces、Cinnamon等等。

NetworkManager配置文件一般位于`/etc/NetworkManager`目录下，更多信息`$man NetworkManager.conf`。

Network与NetworkManager可以说是两套不同的网络管理工具，前者是命令行形式的，一般用于服务器，后者是图形界面形式的，一般用于桌面。两者共同使用时，NetworkManager是否会接管Network所管理的设备由配置文件所决定，比如下面是Debian文档中的描述：

> NetworkManager 不管理任何定义在`/etc/network/interfaces`中的接口。
> 未管理设备指的是NetworkManager并不操作这些网络设备，当下面所提到的两种情况都被满足时候：

> 1. `/etc/network/interfaces`中所设置的任何接口，比如：
>> allow-hotplug eth0
>> iface eth0 inet dhcp
> 2. 同时`/etc/NetworkManager/NetworkManager.conf`包含： 
>> [main]
>> plugins=ifupdown,keyfile
>> 
>> [ifupdown]
>> managed=false

## 建立连接

### 使用NetworkManager

在图形界面下，找到相应的位置，戳戳戳就成。懒得写了。

### 使用Network

这种是基于命令行的管理方式，相应的软件包是*pppoeconf*（DEB发行版）或*rp-pppoe*（RPM发行版）。

#### PPPoE软件包

可使用以下方式查看是否安装相应的软件：

```md
# DEB包发行版
$dpkg -s pppoeconf
Package: pppoeconf
Status: install ok installed # 说明已经安装

# RPM包发行版
$rpm -q rp-pppoe
rp-pppoe-3.12-1.1.x86_64 # 说明已经安装
```

如果没有安装，则：

```md
# DEB包发行版
$sudo apt install pppoeconf

# OpenSUSE系列
$sudo zypper in rp-pppoe 
```

#### PPPoE设置

可根据向导配置PPPoE选项，通过如下命令：

```md
# For pppoeconf
$sudo pppoeconf

# For rp-pppoe
$sudo pppoe-setup
```

在配置过程中出现的术语参见维基百科。

生成的配置文件会存放于`/etc/ppp/peers`目录下，配置选项的含义参考`$man pppd`。若安装的是*pppoeconf*，则会改动`/etc/network/interfaces`；若是*rp-pppoe*， 则为`/etc/sysconfig/network/ifcfg-<ppp device name>`。

用户名（username）与密码（password）存储在`/etc/ppp/chap-secrets`文件中。

#### 手动控制连接

```md
# For pppoeconf
$sudo pon [configuration file name] # 建立连接
$sudo poff [configuration file name] # 关闭连接

# For rp-pppoe
$sudo pppoe-connect [configuration file path] # 建立连接
$sudo pppoe-stop [configuration file path # 关闭连接
```

#### 连接日志

```md
# For pppoeconf
$sudo plog

# For rp-pppoe
$sudo pppoe-status
```

#### IPv6配置

`$man pppd`可以看到有这么一行：
> +ipv6  Enable the IPv6CP and IPv6 protocols.

若打算启用IPv6，添加`+ipv6`至`/etc/ppp/peers/<file name>`是必须的。

IPv6地址由两个逻辑部分组成：一个64位的网络前缀和一个64位的主机地址，主机地址通常根据物理地址自动生成，叫做EUI-64（或者64-位扩展唯一标识）。IPv6地址的默认分配方式是无状态地址自动配置（Stateless Address Autoconfiguration，SLAAC），简单地理解*地址=前缀+MAC地址*。恰恰是因为SLAAC，一个地区的网络前缀是公开的，当知道一台主机的MAC地址的时候此主机的IPv6地址就暴露了，会带来隐私问题。RFC 3041展开了对此问题的讨论，便有了IPv6隐私扩展标准。使用这个隐私扩展，内核会从原本的IPv6地址计算生成一个临时地址。在连接远程服务器时，系统会优先选择这个地址以隐藏原来的地址。也就是说会同时存在多个IPv6地址。

但西电的校园网比较特殊，一个IPv4地址只能拥有一个IPv6地址。而隐私扩展在大部分发行版中是默认启用的，因此需要关闭它，与此有关的是`use_tempaddr`选项：

```md
use_tempaddr - INTEGER
    Preference for Privacy Extensions (RFC3041).
      <= 0 : disable Privacy Extensions
      == 1 : enable Privacy Extensions, but prefer public
             addresses over temporary addresses.
      >  1 : enable Privacy Extensions and prefer temporary
             addresses over public addresses.
    Default:  0 (for most devices)
         -1 (for point-to-point devices and loopback devices)
```

如果开启了IPv6内核转发，会获取不到IPv6地址，这与Router Advertisements这个术语有关，相关配置是`accept_ra`：

```md
accept_ra - INTEGER
    Accept Router Advertisements; autoconfigure using them.

    It also determines whether or not to transmit Router
    Solicitations. If and only if the functional setting is to
    accept Router Advertisements, Router Solicitations will be
    transmitted.

    Possible values are:
        0 Do not accept Router Advertisements.
        1 Accept Router Advertisements if forwarding is disabled.
        2 Overrule forwarding behaviour. Accept Router Advertisements
          even if forwarding is enabled.

    Functional default: enabled if local forwarding is disabled.
                disabled if local forwarding is enabled.
```

因此这么设置就稳妥了：

```md
net.ipv6.conf.default.accept_ra=2
net.ipv6.conf.default.use_tempaddr=0
net.ipv6.conf.all.accept_ra=2
net.ipv6.conf.all.use_tempaddr=0
```

Note：

* DHCP分配地址的方式是存在的，但一般并不使用。

## 更近一步

需求：想使用校园网的IPv6，又想使用易迅中IPv4的无限流量，有没有一种方法实现呢？
解决：
了解过PPP协议的话，知道PPP的过程有三步，第三步进行有关于网络层的设置。*pppd*的文档中有这些设置的描述：

```md
defaultroute
       Add a default route to the system routing tables, using the peer
       as the gateway, when IPCP negotiation is successfully completed.
       This entry is removed when the PPP connection is  broken.   This
       option is privileged if the nodefaultroute option has been spec‐
       ified.

replacedefaultroute
       This option is a flag to the defaultroute  option.  If  default‐
       route  is set and this flag is also set, pppd replaces an exist‐
       ing default route with the new default route.

nodefaultroute
       Disable the defaultroute option.  The system administrator who wishes to prevent users from adding a default route with pppd can  do  so  by
       placing this option in the /etc/ppp/options file.
```

上述选项的含义是在设置网络层参数的时候，是否更改默认路由，对于DEB发行版来说，仅仅需要这么做：

```md
...
# defaultroute
# replacedefaultroute
nodefaultroute
...
```

## Reference

* [Ubuntu Community： ADSLPPPoE](https://help.ubuntu.com/community/ADSLPPPoE)
* [xdlinux google group： IPv6 Over PPPoE](https://groups.google.com/forum/#!topic/xidian_linux/p3XW0jJz4Wo)
* [知乎：拨号和宽带有什么区别？](https://www.zhihu.com/question/48988005)
* [Wikipedia：PPPoE](https://zh.wikipedia.org/wiki/PPPoE)
* [Wikipedia：NetworkManager](https://zh.wikipedia.org/wiki/NetworkManager)
* [Debian Wiki：NetworkManager](https://wiki.debian.org/NetworkManager)
