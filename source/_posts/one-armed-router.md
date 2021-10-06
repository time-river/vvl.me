---
title: 单臂路由
date: 2019-10-07 16:49:52
tags: ['network', 'router']
categories: style
---

## 背景

### 初衷

大学时代，WNDR 4300 + OpenWrt陪伴我度过3年。升学后，忆往昔，加之所在实验室分配了个渣台式机(i5-4590 / 4G RAM / 500G HDD)，遂有意DIY——一来建立微型局域网便于实验，二来有了加速局域网内设备海外访问速度的可能。

### 理论

#### 单臂路由的定义

路由器工作在网络层(Layer 3, L3)，可以用来连接至少两个网络。

为了解决因共享同一物理网线而导致的冲突，创造了交换机；又为了解决局域网中因广播导致的效率低下，创造了路由器。所以有句话：交换机隔离了冲突域，路由器隔离了广播域。物理上的局域网可在逻辑上继续划分，构成一个个虚拟局域网(Virtual Local Area Network, VLAN)，VLAN亦可用于隔离广播域。

单臂路由器(one-armed-router)是一种特殊的路由器，它用来在多个虚拟局域网间传递数据包，一个单臂路由器上连接的多个网络都位于同一个物理连接上。

#### 单工、半双工与全双工

数据通信领域中两台设备之间通信，方式有单工、半双工、全双工。

- 单工模式(simplex mode)：通信是单方向的，两台设备只有一台能够发送，另一台只能接受。
- 半双工模式(half-duplex mode)：每台设备均能够发送与接收，但不能同时进行。
- 全双工模式(full-duplex mode)：双方设备均能够同时发送与接收。

标准以太网(10Mbps)、快速以太网(100Mbps)与千兆以太网(1000Mbps)皆支持半双工与全双工。工作在全双工模式下的千兆以太网，收发速度皆为1000Mbps，因此理论上最大速度是2000Mbps。

#### 交换机、端口模式(Access / Trunk / Hybrid)与802.1Q协议(VLAN)

交换机通常指的是二层（工作在数据链路层）交换机。

交换机端口有三种工作模式：Access、Trunk与Hybrid，区别体现在对带有VLAN id的数据包处理行为：

- Access：只能属于一个VLAN，多用于连接终端设备。
- Trunk：允许多个VLAN，可以接收与发送多个VLAN报文，多用于交换机之间连接。
- Hybrid：被设计用于兼容Acess与Trunk模式。

在涉及三种工作模式的具体行为之前，需要了解下802.1Q协议。

IEEE 802.1Q规定了如何在以太网之上支持VLAN。需要关注的是VLAN tagging的概念与规则。
与VLAN tagging有关的名词有：

- VID(VLAN ID)：VLAN内的port可以接收发自这个VLAN的帧。
- PVID(Port VLAN ID)：定义untagged port可以转发哪种帧。UnTag的帧进入port后会被打上VLAN id。
- UnTag：此帧不带Tag，即不带VLAN id。
- Tag：此帧带Tag，即带VLAN id。
- untagged port：由此port转发的帧都没有Tag(untagged)，若有Tag的帧进入交换机，则其经untagged port转发出交换机时，Tag将被去除。(多用于终端设备)
- tagged port：由此port转发出的帧都将有Tag(tagged)。若有UnTag的帧进入交换机，经tagged port转发出交换机时，Tag将被加上。将使用在流入端口上的PVID设定作为Tag的VLAN id。(多用于交换机之间)

> Note：
> 1. 一个port可以具有多个VID，但只能有一个PVID
> 2. VLAN tagging的相关名词，与port对帧的行为(tag / untag, accept / reject)有关
> 3. Access / Trunk / Hybrid工作模式的端口，也就是untagged port / tagged port。

#### Linux下的网络虚拟化、隧道技术与网络数据处理

Linux实现了字符设备`/dev/net/tun`(文档：[Universal TUN/TAP device driver][16])，利用它可以创建虚拟网卡，二层是tun，三层是tap。它是各类用户态网络设备实现的基础，比如QEMU虚拟网卡的后端、VPN中捕获流量所需的网卡。

Linux也原生支持一些虚拟网卡，比如macvlan、ipvlan ([`$ man 8 ip link`][17])，还包括一些隧道协议l2tp、IPSec，他们可用libnl (Netlink Library Suit)来创建。

网络是分层的，通过添加首部封装数据，高层的流量可由低层承载，同样低层的流量也可由高层承载。据此出现了各类隧道技术，还包括层叠网络(Overlay Network)。

Linux提供了网络数据处理的基础设施：BPF(Berkeley Packet Filter)、netfiler。BPF在链路层复制数据，接收来自用户态前端(比如tcpdump)发来的滤包条件，将符合条件的报文复制到用户空间，提供了网络数据观测的可能。Netfilter在网络协议栈处理数据的关键路径上hook必经的函数，根据用户态前端(比如nftables、iptables)发来的规则，将符合规则的数据包修改、丢弃、转发等。

> Note:  
> netfilter的用户态前端工具，发展历史[[13]]：
```md
ipfwadm -> ipchains -> iptables -> nftables
2.0        2.2         2.4         3.13
```
> nftables融合了ebtables、iptables、ip6tables。

## 设计与实现

### 设备与软件

硬件：

- 单网口的台式机 * 1
- 具有VLAN功能的千兆5口交换机 * 1

软件：

- VMware vSphere Hypervisor 6.7

准备工作：

1. 因为VMware vSphere Hypervisor 6.7不支持Realtek网卡，因此需要自定义安装iso，参考[How to make your unsupported NIC work with ESXi 5.x or 6.0][8]。[这里][9]是相关社区的wiki。
2. 某个版本起，VMware vSphere Hypervisor要求RAM不少于4G。在iso种的UPGRADE/PRECHECK.PY中找到函数checkMemorySize()中的MEM_MIN_SIZE = (4 * 1024) * SIZE_MiB，修改之。
3. 为了保证台式机断电后通电自启动，需要在BIOS中设置下电源管理策略。

### 网络拓扑

网络拓扑设计如下：

![network topology](/images/19/10/network-topology.png)

- 对于5口VLAN交换机
  - 划分两个VLAN区域：id为100的代表WAN，IP地址范围为*202.xxx.7.xxx/23*；id为200的代表LAN，IP地址范围为*192.168.0.0/24*
  - VLAN 100有端口1、2；VLAN 200有端口2、3、4、5
  - 端口1、3、4、5为Untagged Port；端口2为Tagged Port
  - 端口1的PVID为100；端口3、4、5的PVID为200；端口2的PVID随意
- 对于单网口的台式机
  - 运行VMware vSphere Hypervisor
  - 在Hypervisor中建立两个vSwitch：一个VLAN id为100，用于连接WAN；一个VLAN id为200，用于连接LAN
  - 名为Router的Virtual Machine运行于Hypervisor中，具有两个虚拟以太网卡，分别与两个vSwitch相连

> Note：  
> - LAN内的网速均为全双工1Gbps，因此无需担心单臂路由中上下行带宽的抢占
> - 交换机与台式机之间，帧的流动如下：
>   1. WAN中的帧进入Switch端口1，被打上PVID 100后经端口2流出
>   2. Hypervisor中的vSwitch 1接收VLAN id为100的帧，经端口c流出时去掉Tag 100
>   3. Router中的port 1接收来自WAN的帧，经处理后通过port 2传输至LAN中
>   4. vSwitch 2接收来自Router中port 2的帧，打上VLAN id 200后经Hypervisor port传输至Switch中
>   5. Switch中的端口2接收到VLAN id为200的帧，去掉Tag 200后，选择端口3、4、5之一送出

### 实现

#### 单臂路由

关于VMware vSphere Hypervisor与交换机的具体配置过程省略，关注Router(OS: Debian)中的配置：

- DHCP Server
- NAT
- mDNS

Router有两个以太网卡：与WAN相连的eth0，与LAN相连的eth1。通过ISP提供的上网方式获得WAN上的IP *202.xxx.7.xxx/23*，绑定在eth0上；在eth1上添加静态IP *192.168.0.3/24*，作为LAN中的网关(network gateway)。

DHCP Server主要负责ip地址的分配、网关与DNS Server的下发，采用isc-dhcp-server提供DHCP服务，[wiki: DHCP_Server][18]详细叙述了如何配置。(只不过是加一个subnet)
NAT是Router的核心，GNU / Linux上，通过内核参数net.ipv4.ip_forward=1启用IPv4协议的数据报转发，同时使用iptables / nftables添加相应的转发策略，旨在将LAN中的数据报通过SNAT正确地传输至WAN上。

mDNS(multicast DNS)的作用类似DNS，端口是UDP 5353，提供了一种在局域网中无需使用DNS Server即可解析域名的方法。由它解析的顶级域名为.local，解析的域名形如`<hostname>.local`。

##### 可能存在的TCP断流

背景：

- TCP是流，承载它的是IP数据报以及以太网帧，因此数据的每次传输存在最大传输单元(maximum transmission unit，MTU)。对于IP数据报，根据标志位或许可将大于MTU的分组分片；对于以太网帧，若大于发送接口的MTU，则直接丢弃。
- 实现了PMTUD(Path MTU Discovery)的TCP/IP协议栈利用ICMP(type 3, code 4)发现链路中的最小MTU。
- TCP在连接建立阶段会协商MSS(maximum segment size)，避免IP分组的分片。

原因：

- 很多网站或路由器粗暴低禁止了ICMP type 3 code 4，导致PMTUD失效。

解决：

- 添加规则来动态地调整TCP MSS
- 因UDP是无连接的，无法协商MSS大小，因此在PMTUD失效的情况下它或许无法正常工作。可通过禁止IP首部中的DF(Disable Fragment )比特位来强制为任何UDP数据包分片

```bash
# #iptables调整TCP MSS规则
# iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
# #禁止IP首部中的DF，同时也禁止了PMTUD
# echo 1 >/proc/sys/net/ipv4/ip_no_pmtu_disc
```

##### LAN中的IPv6

###### IPv6 NAT[[11]]

- 内核参数net.ipv6.conf.all.forwarding=1用于启用IPv6的SNAT
- 设置net.ipv6.conf.{all|default}.accept_ra=2以便在启用了IPv6 forwarding的情况下仍会获取地址
- 根据网络状态决定是否配置net.ipv6.conf.{all|default}.use_tempaddr=0来禁用隐私拓展

使用radvd或者isc-dhcp-server可以配置DHCPv6。

###### Native IPv6

背景：

- IPv6地址的地址配置：
  - 两种协议：
    - RFC 8415定义的DHCPv6
    - RFC 4862定义的IPv6 Stateless Address Autoconfiguration
  - 三种模式：
    - SLAAC，Stateless Auto Address Configuration
    - Stateless DHCPv6
    - Stateful DHCPv6

条件：拥有/64的IPv6网段

原理：

- 通过桥接虚拟网卡，利用ebtalbes的broute表设置规则
  - 将来自WAN的IPv6流量桥接(DROP)，使之不经过三层路由，而是经二层交换直接抵达终端设备
  - 将来自WAN的IPv6流量路由(ACCEPT)，经过NAT后抵达终端设备

实现[[19]]：

```bash
# #桥接
# brctl addif <bridge> <wan>
# #过滤IPv4流量 
# ebtables -t broute -A BROUTING -p IPv6 -j ACCEPT
# ebtables -t broute -A BROUTING -p ! IPv6 -i <wan> -j DROP
```

#### 隧道

为了加速海外的访问速度，使用[ss-redir][14]在路由与海外服务器之间建立TCP/UDP隧道——通过iptables对符合规则的流量重定向，经隧道转发后使用海外服务器代理访问。同时采用[overture][15]优化DNS解析地址。

- 利用ipset快速匹配符合规则的IP地址
- 利用iptables将符合规则的TCP流量重定向至ss-redir监听的端口
- 因为linux自身的[原因][20]，UDP流量的处理需要借助TPROXY (xt_TPROXY.ko) + 策略路由来实现

> Note：
> - TPROXY只能用于PREROUTING中，文档有[标明][21]。因此无法对本地发出的UDP流量进行重定向
> - 至于DNS服务overture，参考[README.md][22]

## Reference

- [Wikipedia: Gigabit Ethernet][1]
- [Wikipedia: Autonegotiation][2]
- [华为技术支持：交换机端口与C友商模式横向对比][3]
- [CSDN: vlan与交换机端口模式Access，Hybrid，Trunk][4]
- [Wikipedia: IEEE 802.1Q][5]
- [What is PVID][6]
- [Switch基本觀念- PVID、VID、Tag\Untag][7]
- [How to make your unsupported NIC work with ESXi 5.x or 6.0][8]
- [the V-Front Online Depot for VMware ESXi][9]
- [MTU woes in IPsec tunnels and how you can fix it][10]
- [知乎专栏：IPv6 --- 动态地址配置][12]
- [知乎：宿舍的网线接口能上IPV6，但是如果连无线路由再连电脑就无法上IPV6，有解决方法吗？][19]

[1]: https://en.wikipedia.org/wiki/Gigabit_Ethernet
[2]: https://en.wikipedia.org/wiki/Autonegotiation
[3]: https://support.huawei.com/enterprise/zh/knowledge/EKB1000060549
[4]: https://blog.csdn.net/JesseYoung/article/details/40047749
[5]: https://en.wikipedia.org/wiki/IEEE_802.1Q
[6]: https://www.megajason.com/2018/04/30/what-is-pvid/
[7]: https://weihanit.wordpress.com/2017/07/27/switch%E5%9F%BA%E6%9C%AC%E8%A7%80%E5%BF%B5-pvid%E3%80%81vid%E3%80%81taguntag/amp/
[8]: https://www.v-front.de/2014/12/how-to-make-your-unsupported-nic-work.html
[9]: https://vibsdepot.v-front.de/wiki/index.php/Welcome
[10]: https://www.zeitgeist.se/2013/11/26/mtu-woes-in-ipsec-tunnels-how-to-fix/
[11]: https://vvl.me/2017/09/Linux-PPPoE/#IPv6%E9%85%8D%E7%BD%AE
[12]: https://zhuanlan.zhihu.com/p/79405231
[13]: https://www.tldp.org/HOWTO/IP-Masquerade-HOWTO/iptables-vs-ipchains-vs-ipfwadm.html
[14]: https://manpages.debian.org/unstable/shadowsocks-libev/ss-redir.1.en.html
[15]: https://github.com/shawn1m/overture
[16]: https://www.kernel.org/doc/Documentation/networking/tuntap.txt
[17]: https://manpages.debian.org/buster/iproute2/ip-link.8.en.html
[18]: https://wiki.debian.org/DHCP_Server
[19]: https://www.zhihu.com/question/31699421/answer/63285066
[20]: https://vvl.me/2018/06/from-ss-redir-to-linux-nat/
[21]: http://man7.org/linux/man-pages/man8/iptables-extensions.8.html
[22]: https://github.com/shawn1m/overture/blob/master/README.md