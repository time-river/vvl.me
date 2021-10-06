---
title: '译文: 消失的 ifconfig 手册页'
date: 2016-09-26 04:09:40
categories: style
tags: ['data communications and networking', 'translation']
---

这文章与操作说明(man page)并不相同。假设你对 TCP/IP 网络了解得非常少。对于`ifconfig`的参数调用，操作说明已经写得很详细了，这里仅仅说下`ifconfig`的无参数调用的输出。
阅读它并不需要什么特别的知识，这点的确非常像操作说明。

## ifconfig 是什么

`ifconfig`是类 UNIX 系统的系统管理工具，用于诊断和配置网络接口。尽管一些人声称它已经被`iproute2`所取代(`ip`命令)，但它仍然在被广泛地使用。

### 为什么写了这文章？

不加任何参数和选项地调用`ifconfig`，它会输出大量的信息，可想搞明白这些输出并不容易。

---

下面是调用 GNU `ifconfig`的输出。注意：并没有 Wi-Fi 接口，大多数服务器上亦是如此。

```bash
$ ifconfig
eth0      Link encap:Ethernet  HWaddr 08:00:27:0c:49:47  
          inet addr:192.168.0.121  Bcast:192.168.0.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe0c:4947/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:3461 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3686 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:1778710 (1.7 MB)  TX bytes:821363 (821.3 KB)
          Interrupt:10 Base address:0xd020

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:12 errors:0 dropped:0 overruns:0 frame:0
          TX packets:12 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:720 (720.0 B)  TX bytes:720 (720.0 B)
```

---

## 以太网接口(The Ethernet Interface)

```md
eth0      Link encap:Ethernet  HWaddr 08:00:27:0c:49:47
```

数据被逐步地封装并经过 TCP/IP 堆栈。`Link encap:Ethernet`s指的是来自网络层的 IP 数据报(IP Datagrams)在离这个接口之前将被封装成以太网帧。

`HWaddr 08:00:27:0c:49:47`是 48 比特的媒体访问控制(Media Access Control, MAC)地址。在硬件层面的网络接口上，这个地址是独一无二的。其他设备向这个接口上传输以太网帧之前，会发送 ARP 查询请求询问 MAC 地址，接下来的 ARP(Address Resolution Protocol)应答中会包含这个地址。

### IPv4 地址(The IPv4 address)

`inet addr:192.168.0.121`不需要介绍，它是接口使用的 32 比特长度的 IPv4 地址。调用`ifconfig`也往往是为了获取这个地址。

现代网络使用无类域间路由(Classless Inter-Domain Routing, CIDR)的方法把网络切成更小的部分(划分子网)。也因为子网的划分，所以需要知道一 IP 地址中哪里是网络标识号(Network ID)、哪里是主机标识号(Host ID)。这些信息体现在子网掩码(Network Mask)`Mask:255.255.255.0`上。

`Bcast:192.168.0.255`是接口使用的这段子网的广播地址。从这个地址发出的所有数据包将会被子网中的所有接口接收到。可以通过补充子网掩码`Mask:255.255.255.0`的 1 比特位来获得广播地址，就像这样：

```md
Network Mask:           255 . 255 . 255 .   0

Complement all bits:      0 .   0 .   0 . 255
Original IP address:    192 . 168 .   0 . 121
                        _____________________
OR them bitwise:        192 . 168 .   0 . 255
                        Which is the Broadcast Address
```

### 看看 IPv6 地址

本地 IPv6 地址(local IPv6 address)是根据接口的 MAC 地址取得的:

```md
eth0      inet6 addr: fe80::a00:27ff:fe0c:4947/64 Scope:Link
```

`fe80::a00:27ff:fe0c:4947/64`是这个接口 128 比特长度的 link-local IPv6 地址。`Scope:Link`表示它是一个 link-local 地址。Link-local IPv6 地址用于直接连接网络的信息交换，而不是全球。

```md
This is how all link-local addresses are laid out:

10 bits      | 54 bits    | 64 bits
1111 1110 10 | All Zeroes | Interface Identifier

Let's see whether our IPv6 address conforms to this pattern:

                fe80::a00:27ff:fe0c:4947

  (we replace :: with multiple all-zero double-octets)

      fe80:0000:0000:0000 : 0a00:27ff:fe0c:4947

           PREFIX         |   INTERFACE IDENTIFER
  All these zeroes make a | This looks a lot similiar
  link-local IPv6 address | to the MAC address which
  non-routable            | is '08:00:27:0c:49:47'
```

实际上，接口标识符(The Interface Identifier)经常使用 MAC 地址来构造。它被称为 EUI-64(64 bit Extended Unique Indentifier，64 位拓展唯一标识符)。

```md
  08:00:27:0c:49:47        # Start with the MAC adress

  08:00:27:ff:fe:0c:49:47  # Insert ff:fe in the center
  0a:00:27:ff:fe:0c:49:47  # Invert the 7th MSB starting from the right

  0a00:27ff:fe0c:4947      # Group it into double octets!
```

### 关于接口的更多信息

```md
 eth0     UP BROADCAST RUNNING MULTICAST MTU:1500  Metric:1
```

`UP`意味着网络接口在被使用(具有 IP 地址和路由表)，网络层可以访问它。
`BROADCAST`含义是这个接口支持广播(也因此可以使用 DHCP 来获取 IP 地址)。
`RUNNING`表示网络驱动已经被加载，接口已被初始化。
`MULTICAST`告诉我们这个接口支持多播。
无参数调用的`ifconfig`仅仅输出当前具有`UP`标志的接口。

`MTU 1500`说明当前的`Maximum Transmission Unit`被设为 1500 字节，这是以太网允许的最大值。*如果路由与主机允许的话*，任何比 1500 字节大的 IP 报文会被分成多个以太网帧。否则会得到代码为 4 的 ICMP 报文——`Destination Unreachable`。

最后，`Metric:1`是这个接口与路由相关的参数。通常，Linux 内核不会建立基于 metrics 的路由表。这个值代表兼容性。若尝试改变 metric，该命令可能不会工作。

```bash
$ sudo ifconfig eth0 metric 2
SIOCSIFMETRIC: Operation not supported
```

### 统计(Statistics)

```md
eth0      RX packets:3461 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3686 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:1778710 (1.7 MB)  TX bytes:821363 (821.3 KB)
```

`RX`代表接收(received)，`TX`代表`transmitted`。可于此有关的文档并不多。下面是自己对`GNU inetutils 1.9.1`的源码进行探索的发现：
`RX packets`：收到的分组(packet)数量。
`RX errors`：收到的带有错误的分组的总数。这些错误包括帧太长(too-long-frames errors)，环形缓冲区溢出(ring-buffer overflow errors)，CRC 校验错误(CRC errors)，frame alignment errors，fifo overruns，丢失的分组(missed packets)。
环形缓冲区指的是网络接口卡(NIC)在内核发起的中断请求(IRQ)传输帧之前使用的缓冲区。
`RX overruns`显示了 fifo overruns 错误的数量，这是由于环形缓冲区在内核进行 IO 调度之前无法缓存更多的数据。
`RX frame`告知了有多少进入的帧 misaligned。(`RX frame` accounts for the incoming frames that were misaligned.)

`TX packets`表明了传输的分组总数。
`TX errors`显示了发送的分组有多少出现了错误。包括因传输中断出现的错误，因 carrier、fifo errors、heartbeat errors 导致的错误，和窗口错误。*这个特殊的`struct`在源码中并没有说明。*这些错误被分类为`dropped`，`overruns`和`carrier`。

`collisions`显示的是因 CSMA/CD(Carrier Sense Multiple Access with Collision Detection)而致使传输终止的数目。

最后一行是成功接收与成功传输的数据大小。

### 传输队列长度(Transmit Queue Length)

因为它并不是一个统计量，所以另起一个小标题。
`txqueuelen`显示的是当前发送的队列长度。这个数字限制了在接口设备驱动处等待传输的帧的数量。`txqueuelen`的值可以被`ifconfig`命令改变。

### 中断(Interrupts)

```md
eth0      Interrupt:10 Base address:0xd020
```

`Interrupt:10`对应的是中断请求序号，可以从`/etc/interrupts`中查找`eth0`设备得到。

```md
$ cat /proc/interrupts
               CPU0
      0:        115    XT-PIC-XT-PIC    timer
      1:       3402    XT-PIC-XT-PIC    i8042
      2:          0    XT-PIC-XT-PIC    cascade
      5:          1    XT-PIC-XT-PIC    snd_intel8x0
      8:          0    XT-PIC-XT-PIC    rtc0
      9:          0    XT-PIC-XT-PIC    acpi
=>   10:      53981    XT-PIC-XT-PIC    eth0             <=
     11:       1535    XT-PIC-XT-PIC    ohci_hcd:usb1
     12:        146    XT-PIC-XT-PIC    i8042
     14:      16923    XT-PIC-XT-PIC    ata_piix
     15:      10416    XT-PIC-XT-PIC    ata_piix
```

`53981`是`eth0`设备中断`CPU0`的总数。
第三列告诉了我们可编程中断处理程序(the programmable interrupt handler)的名称，`XT-PIC-XT-PIC`的出现疑似是因为 VirtualBox。

## 环回接口(The Loopback Interface)

```md
lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:12 errors:0 dropped:0 overruns:0 frame:0
          TX packets:12 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:720 (720.0 B)  TX bytes:720 (720.0 B)
```

### 换回接口不是以太网设备

它并不与网络接口卡(或者其他硬件)相连，通过环回接口传递的帧并不存在于主机上的任何层。这完全由软件实现。它也意味着发往这个接口的 IP 数据报并不会封装成以太网帧，这可从`Link encap:Local Loopback`看出。

### IPv4 地址

```md
lo        inet addr:127.0.0.1  Mask:255.0.0.0
```

`Mask:255.0.0.0`告诉我们这是一大块地址。环回地址可以设置为这段子网(`127.0.0.1`至`127.255.255.254`)中的任意一个 IP 地址。默认为`127.0.0.1`。

### IPv6

不同于 IPv4，只有一个地址被保留为环回地址——`0:0:0:0:0:0:0:1`，往往简写为`::1/128`，这是因为多组连续的 0 可用`::`代替。
环回地址的 IPv6 地址为`::1/128`，它与`link-local`的地址范围一起在 RFC 3531 中规定。

### 接口

```md
lo        UP LOOPBACK RUNNING  MTU:16436  Metric:1
```

标志里有同名的`LOOPBACK`，`MTU:16436`与前一个却并不相同。这是因为环回接口并不被物理层面所限制，无论是以太网还是光纤分布式数据接口(FDDI, Fiber Distributed Data Interface)，它的 MTU 可以超过 16KiB。每个数据包可达`16 x 1024 = 16384`字节，即使外加 52 字节的额外信息扔可不被分片。这 52 字节通常用于 TCP 与 IP 的首部。
`Metric`的概念同以太网接口一样。

### 统计与传输队列长度

```md
lo        RX packets:12 errors:0 dropped:0 overruns:0 frame:0
          TX packets:12 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
```

环回地址输出的统计信息与以太网接口一样。但错误与 collisions 有些不同，因为它并不是物理媒介。
`txqueuelen`默认为 0。对`lo`设备来说，它可以被改变，可改变了会有什么作用吗？

## 其他工具

不喜欢 GNU `ifconfig` ，或者计算机上没有它？没关系，毕竟还有其他类似的工具。因为两者都来源于 BSD(the BSD userland)，`netstat -ai`与`ifconfig`也在 Mac OS X 上工作，但输出稍有不同。

使用`iproute2`：

```bash
$ ip --statistics link list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN mode DEFAULT
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    RX: bytes  packets  errors  dropped overrun mcast
    67710      812      0       0       0       0
    TX: bytes  packets  errors  dropped carrier collsns
    67710      812      0       0       0       0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN mode DEFAULT qlen 1000
    link/ether 08:00:27:89:cf:84 brd ff:ff:ff:ff:ff:ff
    RX: bytes  packets  errors  dropped overrun mcast
    10372230   53359    9       0       0       0
    TX: bytes  packets  errors  dropped carrier collsns
    206555     1826     0       0       0       0
```

或者是`netstat`，`ifconfig`的输出实际上是基于它(Or with `netstat`, on which the `ifconfig` output is actually based on)：

```bash
$ netstat --all --interfaces
Kernel Interface table
Iface       MTU Met   RX-OK RX-ERR RX-DRP RX-OVR    TX-OK TX-ERR TX-DRP TX-OVR Flg
eth0       1500 0     56092     10      0 0          3095      0 0      0      BMRU
lo        16436 0       858      0      0 0           858      0 0      0      LRU
```

`Flg`显示的是接口的状态。`BMRU`代表广播(Broadcast)、多播(Multicast)、运行中(Running)和使用(Up)。`LRU`代表环回地址(Loopback)、运行中(Running)和使用(Up)。

## [原文](http://blog.hyfather.com/blog/2013/03/04/ifconfig/)

## 参考文献

1. Cotton, M., & Vegoda, L. (2010). Special Use IPv4 Addresses. Internet Engineering Taskforce RFC 5735.
Domingo, D. & Bailey, L. (Eds.). (2011). Red Hat Enterprise Linux 6 Performance Tuning Guide. Red Hat, Incorporated.
2. Fall, K. R., & Stevens, W. R. (2011). TCP/IP Illustrated, Volume 1: The Protocols (Vol. 1). Addison-Wesley Professional.
3. Hinden, R. M., & Deering, S. E. (2003). Internet protocol version 6 (IPv6) addressing architecture. Internet Engineering Taskforce RFC 3513.
4. Hunt, C. (2002). TCP/IP network administration. O’Reilly Media, Incorporated.
5. Kempen, F., Cox, A., Blundell, P., Kleen, A., Eckenfels, B. (2007). ifconfig(8) Manual Page.
6. Kirch, O., & Dawson, T. (2000). Linux network administrator’s guide. O’Reilly Media, Incorporated.
