---
title: 我可以利用TUN/TAP设备写一个IP packet modifer吗？
date: 2018-06-06 16:19:37
tags: ['TUN/TAP', 'network', 'linux']
---

## 引子

2017年末，在SUSE实习的时候`man ip link`，看到了这种奇奇怪怪的东西：

```md
TYPE := [ bridge | bond | can | dummy | hsr | ifb | ipoib | macvlan | macvtap | vcan | veth | vlan | vxlan |
        ip6tnl | ipip | sit | gre | gretap | ip6gre | ip6gretap | vti | nlmon | ipvlan | lowpan | geneve | vrf
        | macsec ]
```

除了`bridge`，别的都没有见过，于是学习了下与之有关的知识。对[图解几个与Linux网络虚拟化相关的虚拟网卡-VETH/MACVLAN/MACVTAP/IPVLAN][1]中的以下观点感到不错：

> 一块网卡就是一道门，一个接口，它上面一般接协议栈，下面一般接介质。最关键的是，你要明确它们确实在上面和下面接的是什么。
> TUN/TAP：用户可以用文件句柄操作的字符设备

前者就是网络中的封装(encapsulation)。而后者，[维基百科][2]介绍说：TUN/TAP是一种虚拟的网络设备。TAP等同于一个以太网设备提供了一种机制，它操作第二层网络数据包如以太网数据帧；TUN模拟了网络层设备，操作第三层数据包比如IP数据包。操作系统通过TUN/TAP设备向绑定该设备的用户空间程序发送数据，反之，用户空间的程序也可以像操作硬件网络设备那样，通过TUN/TAP设备发送数据。

要知道，从用户空间流经内核的网络数据流并不会传递给用户空间的程序，而是直接交给NIC传输。TUN/TAP设备的特性可以将网络数据传递给用户空间程序，使用户空间的程序处理网络数据流成为可能。

那么，能不能实现一种用户空间的程序，不向内核添加一行代码，便可以篡改、监视本机的所有网络流量呢？这与Stack Overflow中的[Can I make a “TCP packet modifier” using tun/tap and raw sockets?][3]不谋而合。

## 想法描述

这中用户空间的程序起名为IP modifer，假如发送个ICMP Request，数据包有着如下的流程：

[![IP-modifer.md.png](https://img.vvl.me/images/2018/06/06/IP-modifer.md.png)](https://img.vvl.me/image/s18)

1. 通过配置路由，使ICMP Request经过TUN设备送出。
2. 数据包流经内核协议栈，将封装有ICMP Request的IP数据包传递给IP packet modifer程序。
3. 数据包经IP packet modifer处理后通过eth0设备送出。
4. 数据包再一次流经内核协议栈，最后交给NIC送出本机。

## 技术难点

想法虽好，但是：

- 如何使数据包流经TUN设备？
- IP packet modifer处理后怎样通过eth0设备送出？

这种需求很容易想到各类VPN软件的实现，为此，我专门在我的腾讯云上部署了ocserv，并调研了一番源码。

### ocserv中的实现

客户端与服务端建立连接后，`ip a`显示了如下的信息：

```md
// Client
9: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1313 qdisc pfifo_fast state UNKNOWN group default qlen 500
    link/none
    inet 172.16.31.3/32 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::65d:48dd:9c60:323/64 scope link flags 800
       valid_lft forever preferred_lft forever

// Server
20: vpns0: <POINTOPOINT,UP,LOWER_UP> mtu 1313 qdisc pfifo_fast state UNKNOWN group default qlen 500
    link/none
    inet 172.16.31.1 peer 172.16.31.3/32 scope global vpns0
       valid_lft forever preferred_lft forever
```

`ip r`信息如下：

```bash
// Client
me@debian:~/l2tp$ ip route
default dev tun0 scope link
119.29.x.25 via 192.168.80.2 dev eth0 src 192.168.80.128
172.16.31.0/24 dev tun0 scope link
192.168.80.0/24 dev eth0 proto kernel scope link src 192.168.80.128

# 注：119.29.x.25为服务端IP。
# 腾讯云的网络采用static NAT方式，一个公网IP地址与一个内网IP地址绑定，
# 因此服务端显示的IP地址为10.135.44.93

// Server
ubuntu@VM-44-93-ubuntu:~$ ip r
default via 10.135.0.1 dev udp0-internal
10.135.0.0/18 dev eth0  proto kernel  scope link  src 10.135.44.93
172.16.31.3 dev vpns0  proto kernel  scope link  src 172.16.31.1
```

同时，Server端通过`tcpdump`可以看到：

```bash
root@VM-44-93-ubuntu:~# tcpdump -i any port 8000 -vvv
tcpdump: listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
18:46:49.345299 IP (tos 0x0, ttl 115, id 28480, offset 0, flags [DF], proto TCP (6), length 52)
    113.140.11.123.23027 > 10.135.44.93.8000: Flags [.], cksum 0xe5cf (correct), seq 4093742334, ack 2560316182, win 417, options [nop,nop,sack 1 {1413:7061}], length 0
18:46:49.345328 IP (tos 0x0, ttl 64, id 20538, offset 0, flags [DF], proto TCP (6), length 894)
    10.135.44.93.8000 > 113.140.11.123.23027: Flags [P.], cksum 0xb75b (incorrect -> 0x8564), seq 8473:9327, ack 0, win 4715, length 854
18:46:49.345338 IP (tos 0x0, ttl 115, id 28481, offset 0, flags [DF], proto TCP (6), length 40)
    113.140.11.123.23027 > 10.135.44.93.8000: Flags [.], cksum 0xf8ce (correct), seq 0, ack 7061, win 417, length 0

root@VM-44-93-ubuntu:~# tcpdump -i vpns0 -vvv
tcpdump: listening on vpns0, link-type RAW (Raw IP), capture size 262144 bytes
18:53:15.185651 IP (tos 0x0, ttl 64, id 64599, offset 0, flags [DF], proto UDP (17), length 52)
    172.16.31.3.40351 > 172.16.31.1.domain: [udp sum ok] 44379+ A? jd.com. (24)
18:53:15.195137 IP (tos 0x0, ttl 64, id 47204, offset 0, flags [DF], proto UDP (17), length 74)
    172.16.31.1.domain > 172.16.31.3.40351: [udp sum ok] 44379 q: A? jd.com. 1/0/0 jd.com. [1m14s] A 106.39.167.118 (46)
18:53:15.222268 IP (tos 0x0, ttl 64, id 64600, offset 0, flags [DF], proto UDP (17), length 52)
    172.16.31.3.40351 > 172.16.31.1.domain: [udp sum ok] 6814+ AAAA? jd.com. (24)
18:53:15.256181 IP (tos 0x0, ttl 64, id 47206, offset 0, flags [DF], proto UDP (17), length 122)
    172.16.31.1.domain > 172.16.31.3.40351: [udp sum ok] 6814 q: AAAA? jd.com. 0/1/0 ns: jd.com. [1m14s] SOA ns1.jdcache.com. apollo.jd.com. 2015126609 10800 3600 604800 38400 (94)
18:53:15.311630 IP (tos 0x0, ttl 64, id 663, offset 0, flags [DF], proto TCP (6), length 60)
    172.16.31.3.59362 > 106.39.167.118.http: Flags [S], cksum 0x8bd4 (correct), seq 3078582050, win 25460, options [mss 1273,sackOK,TS val 3632565 ecr 0,nop,wscale 7], length 0
18:53:15.354061 IP (tos 0x68, ttl 49, id 55915, offset 0, flags [none], proto TCP (6), length 52)
    106.39.167.118.http > 172.16.31.3.59362: Flags [S.], cksum 0x7705 (correct), seq 405060611, ack 3078582051, win 14600, options [mss 1273,nop,nop,sackOK,nop,wscale 7], length 0
18:53:15.390477 IP (tos 0x0, ttl 64, id 664, offset 0, flags [DF], proto TCP (6), length 40)
    172.16.31.3.59362 > 106.39.167.118.http: Flags [.], cksum 0xef5d (correct), seq 1, ack 1, win 199, length 0
18:53:15.477524 IP (tos 0x0, ttl 64, id 665, offset 0, flags [DF], proto TCP (6), length 171)
    172.16.31.3.59362 > 106.39.167.118.http: Flags [P.], cksum 0x1bff (correct), seq 1:132, ack 1, win 199, length 131: HTTP, length: 131
    GET / HTTP/1.1
    User-Agent: Wget/1.18 (linux-gnu)
    Accept: */*
    Accept-Encoding: identity
    Host: jd.com
    Connection: Keep-Alive
    
18:53:15.521598 IP (tos 0x68, ttl 49, id 55916, offset 0, flags [none], proto TCP (6), length 401)
    106.39.167.118.http > 172.16.31.3.59362: Flags [P.], cksum 0x4fab (correct), seq 1:362, ack 132, win 123, length 361: HTTP, length: 361
    HTTP/1.1 302 Moved Temporarily
    Server: JengineD/1.7.2.1
    Date: Wed, 06 Jun 2018 10:53:15 GMT
    Content-Type: text/html
    Content-Length: 165
    Location: http://www.jd.com
    Connection: keep-alive
    
    <html>
    <head><title>302 Found</title></head>
    <body bgcolor="white">
    <center><h1>302 Found</h1></center>
    <hr><center>JengineD/1.7.2.1</center>
    </body>
    </html>
18:53:15.560272 IP (tos 0x0, ttl 64, id 666, offset 0, flags [DF], proto TCP (6), length 40)
    172.16.31.3.59362 > 106.39.167.118.http: Flags [.], cksum 0xed68 (correct), seq 132, ack 362, win 208, length 0
```

根据上面列出的信息，我猜想了这样几件事情：

1. 客户端建立连接后，会修改默认路由为`172.16.31.1`，它定义在ocserv的配置文件中，为服务端IP地址`119.29.x.25`设置静态路由，使数据不经过TUN设备。
2. 客户端将流经TUN设备的数据发往服务端，端口为8000，它同样定义在ocserv的配置文件中。
3. 服务端接收数据后，会向vpns0设备写入由客户端TUN设备捕获的数据。
4. iptables nat表中POSTROUTING链上的SNAT规则使客户端可以与106.39.167.118通信。

在ocserv的源码中，又找到了下列信息：

```c
// src/tun.h
ssize_t tun_write(int sockfd, const void *buf, size_t len);

/*
  root@VM-44-93-ubuntu:/tmp/ocserv-0.10.11# grep -rni tun_write
  src/worker-vpn.c:2130:          ret = tun_write(ws->tun_fd, plain, plain_size);
*/
// src/worker-vpn.c
    case AC_PKT_DATA:
        oclog(ws, LOG_TRANSFER_DEBUG, "writing %d byte(s) to TUN",
              (int)plain_size);
        ret = tun_write(ws->tun_fd, plain, plain_size);
        if (ret == -1) {
            e = errno;
            oclog(ws, LOG_ERR, "could not write data to tun: %s",
                  strerror(e));
            return -1;
        }
        ws->tun_bytes_in += plain_size;
        ws->last_nc_msg = now;
```

确认猜想的第三点使正确的。并且从Stack Overflow上的[What exactly happens to packets written to a TUN/TAP device?][4]回答得知，向TUN设备注入数据包，kernel并不会知道这个数据包到底来自真实的NIC物理设备还是TUN设备。

### 我的实现

并没有采取直接修改`main`路由表的方式来使所有网络流量流经TUN设备，而是使用了策略路由：

- 非来自TUN设备的IP数据包查询路由表100
- 路由表100的优先级比254高

创建TUN设备并设置策略路由的相关命令如下：

```bash
# ip tuntap add mode tun tun0
# ip link set tun0 up
# ip addr add 10.0.0.2/24 dev tun0
# ip route add default via 10.0.0.2 dev tun0 table 100
# ip rule add from all pref 100 lookup 100
```

这样，IP packet modifer可以截获所有的网络流量，参考ocserv的实现可以知道，只需要把数据写回TUN设备便可，那么就要求：

- 来自TUN设备的IP数据包查询路由表254（main）

仅需下面的这个命令：

```bash
# ip rule add from all iif tun0 lookup main
```

但，默认情况下，[任何从非loopback网卡进来的任何数据包的源地址不能是本机地址][5]。若接收源地址为本机地址的数据包需要NIC启用[`accept_local`][6]，与此有关的patch在[这][7]，为kernel v2.6.33中的新增特性。因此，还需要执行以下命令：

```bash
# sysctl -w net.ipv4.conf.tun0.accept_local=1
```

## 实现

据此，我实现了三个简单的示例程序：

- [simple-tun-read-write.py](https://gist.github.com/time-river/f2288ae1dbe8bfa5cc84a841c9a507ad)

通过策略路由，它会截获所有自主机发出的网络流量，并把他们打印出来。

- [ip-datagram.py](https://gist.github.com/time-river/77b8647201284202ca2ca260c671b07c)

与simple-tun-read-write.py类似，但它无需手动配置。

- [icmp-echo](https://github.com/time-river/network-research/tree/master/tun-tap/icmp-echo)

使用rust实现的IP packet modifer，它可以伪造ICMP echo。

相关的源码都可以在[这里](https://github.com/time-river/network-research/tree/master/tun-tap)找到。

## 参考资料

[图解几个与Linux网络虚拟化相关的虚拟网卡-VETH/MACVLAN/MACVTAP/IPVLAN][1]
[TUN與TAP][2]
[Can I make a “TCP packet modifier” using tun/tap and raw sockets?][3]
[What exactly happens to packets written to a TUN/TAP device?][4]
[关于Linux内核引入的accept_local参数的一个问题][5]
[How to configure linux routing/filtering to send packets out one interface, over a bridge and into another interface on the same box][6]
[ipv4 05/05: add sysctl to accept packets with local source addresses][7]

[1]: https://blog.csdn.net/dog250/article/details/45788279 "图解几个与Linux网络虚拟化相关的虚拟网卡-VETH/MACVLAN/MACVTAP/IPVLAN"
[2]: https://zh.wikipedia.org/zh-hk/TUN%E4%B8%8ETAP "TUN與TAP"
[3]: https://stackoverflow.com/questions/2664955/can-i-make-a-tcp-packet-modifier-using-tun-tap-and-raw-sockets "Can I make a “TCP packet modifier” using tun/tap and raw sockets?"
[4]: https://serverfault.com/questions/671516/what-exactly-happens-to-packets-written-to-a-tun-tap-device "What exactly happens to packets written to a TUN/TAP device?"
[5]: https://blog.csdn.net/dog250/article/details/78746198 "关于Linux内核引入的accept_local参数的一个问题"
[6]: https://serverfault.com/questions/411921/how-to-configure-linux-routing-filtering-to-send-packets-out-one-interface-over "How to configure linux routing/filtering to send packets out one interface, over a bridge and into another interface on the same box"
[7]: https://patchwork.ozlabs.org/patch/40152/ "ipv4 05/05: add sysctl to accept packets with local source addresses"
