---
title: 社区服务器网络维修记录
date: 2017-05-14 20:15:50
categories: style
tags: [network]
---

4.29 日维修过后，有半个月了吧...拖延证有点严重。上月底一次停电，来电后，服务器网络有故障了，专门去了一趟老校区来修理。

到了机房，`ifconfig` 查看网络接口的状态。恩，大致是这样：

```bash
vvl@xdlinux:~$ ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:1a:1b:75:06  
inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
inet6 addr: fe80::1/64 Scope:Link
inet6 addr: fe80::42:1aff:fe1b:7506/64 Scope:Link
inet6 addr: fdbd:6b69:4dc6::1/48 Scope:Global
UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
RX packets:162393390 errors:0 dropped:0 overruns:0 frame:0
TX packets:331323123 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:0
RX bytes:12041875764 (11.2 GiB)  TX bytes:498767843534 (464.5 GiB)

lo        Link encap:Local Loopback  
inet addr:127.0.0.1  Mask:255.0.0.0
inet6 addr: ::1/128 Scope:Host
UP LOOPBACK RUNNING  MTU:65536  Metric:1
RX packets:2238346 errors:0 dropped:0 overruns:0 frame:0
TX packets:2238346 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:0
RX bytes:377411960 (359.9 MiB)  TX bytes:377411960 (359.9 MiB)
```

显然网卡没启动，`/etc/init.d/networking` 走起。=\_= 没有任何输出信息...诶，那就手动启动吧。`ifconfig ethx up` ~再`ifconfig`，大致是这样的输出：

```bash
vvl@xdlinux:~$ ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:1a:1b:75:06  
inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
inet6 addr: fe80::1/64 Scope:Link
inet6 addr: fe80::42:1aff:fe1b:7506/64 Scope:Link
inet6 addr: fdbd:6b69:4dc6::1/48 Scope:Global
UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
RX packets:162393390 errors:0 dropped:0 overruns:0 frame:0
TX packets:331323123 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:0
RX bytes:12041875764 (11.2 GiB)  TX bytes:498767843534 (464.5 GiB)

eth0      Link encap:Ethernet  HWaddr 48:5b:39:b8:37:0d  
UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
RX packets:332151441 errors:0 dropped:66005 overruns:0 frame:0
TX packets:162917713 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:1000
RX bytes:502086864230 (467.6 GiB)  TX bytes:15667890997 (14.5 GiB)

eth1      Link encap:Ethernet  HWaddr 00:23:7d:6d:50:9d  
UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
RX packets:35736452 errors:1 dropped:22061 overruns:0 frame:1
TX packets:129046572 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:1000
RX bytes:2719695714 (2.5 GiB)  TX bytes:192402119248 (179.1 GiB)
Interrupt:19

lo        Link encap:Local Loopback  
inet addr:127.0.0.1  Mask:255.0.0.0
inet6 addr: ::1/128 Scope:Host
UP LOOPBACK RUNNING  MTU:65536  Metric:1
RX packets:2238346 errors:0 dropped:0 overruns:0 frame:0
TX packets:2238346 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:0
RX bytes:377411960 (359.9 MiB)  TX bytes:377411960 (359.9 MiB)
```

ps: 忽略 RX/TX bytes...不要在意细节。再说，我也不知道是不是这样的输出，我是根据`ip addr`看出问题的~！

```bash
2: enp3s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN group default qlen 1000
    link/ether 94:de:80:af:d8:39 brd ff:ff:ff:ff:ff:ff
    inet 10.170.41.48/16 brd 10.170.255.255 scope global dynamic enp3s0
       valid_lft 246001sec preferred_lft 246001sec
    inet6 2001:250:1006:dff0::1:67a2/128 scope global dynamic
       valid_lft 245998sec preferred_lft 159598sec
    inet6 fe80::13e0:ea7:7fe3:d7b3/64 scope link
       valid_lft forever preferred_lft forever
```

输出差不多是上面那样，注意__`<NO-CARRIER,BROADCAST,MULTICAST,UP>`__，__NO-CARRIER__嘛，物理层的问题？...（省略若干被遗忘的东西）总之，网卡插得好好的，没问题。`sudo pon dsl-provider` 成功拨号，手动添加IP 地址、路由，成功。此时就剩下一个问题了：`/etc/init.d/networking` 启动脚本怎么了？

先从自己正常工作的腾讯云上 download 下来一个，diff 二者，没问题。思考，这是怎么了呢？后来想到，`ifupdown`这个工具是用来管理网络接口的，是不是他们出问题了呢？尝试搜了一下这个包：

```md
vvl@xdlinux:~$ apt search ifupdown
Sorting... Done
Full Text Search... Done
guessnet/stable 0.56 amd64
  Guess which LAN a network device is connected to

ifupdown/stable,now 0.7.53.1 amd64 [...]
  high level tools to configure network interfaces

ifupdown-extra/stable 0.25 all
  Network scripts for ifupdown

ifupdown-multi/stable 0.1.1 all
  multiple default gateway support for ifupdown

ifupdown-scripts-zg2/stable 0.6-1 all
  Zugschlus' interface scripts for ifupdown's manual method

netscript-ipfilter/stable 5.4.10 all
  Linux 2.6/3.x iptables management system.
```

哇，还真有这个包，`[...]`内容我忘记了，反正不是正常安装的`[installed]`。重新安装之，重启，搞定。

很奇怪：这个包是怎么损坏的呢？一个玄学...
