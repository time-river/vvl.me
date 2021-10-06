---
title: →_→ iptables
date: 2016-08-04 13:57:06
categories: style
tags: iptables
---

这里记录下 iptables 原理等相关的东西，相信读者们不会看完，我写(抄)了三天，抄完已经歇菜。

## 数据包的流向

### 基本概念

* 表(tables)
  * iptables 包含 5 张表
    * __raw__ 用于配置数据包，__raw__ 中的数据包不会被系统跟踪
    * __filter__ 是用于存放所有与防火墙相关操作的默认表
    * __nat__ 用于[网络地址转换](https://en.wikipedia.org/wiki/Network_address_translation)（例如：端口转发）
    * __mangle__ 用于对特定数据包的修改（参考[损坏数据包](https://en.wikipedia.org/wiki/Mangled_packet)）
    * __security__ 用于[强制访问控制](https://wiki.archlinux.org/index.php/Security#Mandatory_access_control)网络规则（例如： SELinux -- 详细信息参考[该文章](http://lwn.net/Articles/267140/)）
  * 大部分情况仅需要使用 __filter__ 和 __nat__。
* 链(chains)
  * 表由链组成，链是一些按顺序排列的规则的列表
  * 默认的 __filter__ 表包含 __INPUT__, __OUTPUT__ 和 __FORWARD__ 3条内建的链
  * __nat__ 表包含 __PREROUTING__, __POSTROUTING__ 和 __OUTPUT__ 链
  * 链的默认规则通常设置为 __ACCEPT__，如果想确保任何包都不能通过规则集，那么可以重置为 __DROP__
  * 默认的规则总是在一条链的最后生效
* 规则(rules)
  * 数据包的过滤基于__规则__
    * __规则__由一个目标（数据包包匹配所有条件后的动作）和很多匹配（导致该规则可以应用的数据包所满足的条件）指定
    * 一个__规则__的典型匹配事项是数据包进入的端口（例如：eth0 或者 eth1）、数据包的类型（ICMP, TCP, 或者 UDP）和数据包的目的端口
  * 目标(target)使用 __-j__ 或者 __--jump__ 选项指定
    * 目标可以是用户定义的链（例如，如果条件匹配，跳转到之后的用户定义的链被继续处理）、一个内置的特定目标或者是一个目标扩展
    * 如果目标是内置目标，数据包的命运会立刻被决定并且在当前表的数据包的处理过程会停止
      * 内置目标是 __ACCEPT__,  __DROP__, __QUEUE__ 和 __RETURN__, 目标扩展是 __REJECT__ 和 __LOG__
    * 如果目标是用户定义的链，并且数据包成功穿过第二条链，目标将移动到原始链中的下一个规则
    * 目标扩展可以被终止（像内置目标一样）或者不终止（像用户定义链一样）
* 模块(modules)
  * 有许多模块可以用来扩展 iptables，例如 connlimit, conntrack, limit 和 recent。这些模块增添了功能，可以进行更复杂的过滤。

### 表和链的遍历

这里的内容是关于数据包是以什么顺序、如何穿越不同的链和表的。

下图简要描述了网络数据包通过 iptables 的 __nat__ 与 __filter__ 表的过程

```md
                               XXXXXXXXXXXXXXXXXX
                             XXX     Network    XXX
                               XXXXXXXXXXXXXXXXXX
                                       +
                                       |
                                       v
 +-------------+              +------------------+
 |table: filter| <---+        | table: nat       |
 |chain: INPUT |     |        | chain: PREROUTING|
 +-----+-------+     |        +--------+---------+
       |             |                 |
       v             |                 v
 [local process]     |           ****************          +--------------+
       |             +---------+ Routing decision +------> |table: filter |
       v                         ****************          |chain: FORWARD|
****************                                           +------+-------+
Routing decision                                                  |
****************                                                  |
       |                                                          |
       v                        ****************                  |
+-------------+       +------>  Routing decision  <---------------+
|table: nat   |       |         ****************
|chain: OUTPUT|       |               +
+-----+-------+       |               |
      |               |               v
      v               |      +-------------------+
+--------------+      |      | table: nat        |
|table: filter | +----+      | chain: POSTROUTING|
|chain: OUTPUT |             +--------+----------+
+--------------+                      |
                                      v
                               XXXXXXXXXXXXXXXXXX
                             XXX    Network     XXX
                               XXXXXXXXXXXXXXXXXX
```

__第一个路由策略__包括__决定数据包的目的地__是本地主机（这种情况下，数据包穿过 INPUT 链），还是其他主机（数据包穿过 FORWARD 链）；__中间的路由策略__包括决定给传出的数据包__使用那个源地址、分配哪个接口__；__最后一个路由策略__存在是因为先前的 mangle 与 nat 链可能会改变数据包的路由信息。

数据包通过路径上的每一条链时，链中的每一条__规则按顺序匹配__；无论何时__匹配了一条规则__，相应的 target/jump 动作将会执行。最常用的3个 target 是 ACCEPT, DROP ,或者 jump 到用户自定义的链。只有内置的链拥有默认的策略，用户自定义的链没有默认的策略(Policy, 当封包__不在设定的规则之内__时，则该封包的通过与否，是以 Policy 的设定为准)。在 jump 到的用户自定义的链中，若每一条规则都不能提供完全匹配，那么数据包像下面图片描述的一样返回到调用链。

![table subtraverse](/images/16/08/table_subtraverse.jpg)

在任何时候，若 DROP target 的规则实现完全匹配，那么被匹配的数据包会被丢弃，不会进行进一步处理。如果一个数据包在链中被 ACCEPT，那么它也会被所有的父链 ACCEPT，并且不再遍历其他父链。然而，要注意的是，数据包还会以正常的方式继续遍历其他表中的其他链。

更详细的过程参见[这里](#iptables-流程图的详细解释)

## 一些乱七八糟的知识

### [NAT](https://zh.wikipedia.org/wiki/%E7%BD%91%E7%BB%9C%E5%9C%B0%E5%9D%80%E8%BD%AC%E6%8D%A2)

* 分类
  * 静态 NAT(Static NAT)
  * 动态地址 NAT(Pooled NAT)
  * 网络地址端口转换 NAPT(Port-Level NAT)
    * 网络地址端口转换 NAPT(Network Address Port Translation)则是把内部地址映射到外部网络的一个 IP 地址的不同端口上。它可以将中小型的网络隐藏在一个合法的 IP 地址后面。NAPT 与动态地址 NAT 不同，它将内部连接映射到外部网络中的一个单独的 IP 地址上，同时在该地址上加上一个由 NAT 设备选定的端口号
    * 包含 SNAT 与 DNAT
      * 源NAT(Source NAT, SNAT): 修改数据包的源地址。源NAT 改变第一个数据包的来源地址，它永远会在数据包发送到网络之前完成，数据包伪装就是一个 SNAT 的例子
      * 目的NAT(Destination NAT, DNAT): 修改数据包的目的地址。DNAT 刚好与 SNAT 相反，它是改变第一个数据懈的目的地地址，如平衡负载、端口转发和透明代理就是属于 DNAT
* 应用
  * NAT 主要可以实现以下几个功能: 数据包伪装、平衡负载、端口转发和透明代理
    * 数据伪装: 可以将内网数据包中的地址信息更改成统一的对外地址信息，不让内网主机直接暴露在因特网上，保证内网主机的安全。同时，该功能也常用来实现共享上网
    * 端口转发: 当内网主机对外提供服务时，由于使用的是内部私有IP地址，外网无法直接访问。因此，需要在网关上进行端口转发，将特定服务的数据包转发给内网主机
    * 负载平衡: 目的地址转换 NAT 可以重定向一些服务器的连接到其他随机选定的服务器
    * 失效终结: 目的地址转换NAT可以用来提供高可靠性的服务。如果一个系统有一台通过路由器访问的关键服务器，一旦路由器检测到该服务器当机，它可以使用目的地址转换NAT透明的把连接转移到一个备份服务器上
    * 透明代理: NAT 可以把连接到因特网的 HTTP 连接重定向到一个指定的 HTTP 代理服务器以缓存数据和过滤请求。一些因特网服务提供商就使用这种技术来减少带宽的使用而不用让他们的客户配置他们的浏览器支持代理连接
* 原理
  * 地址转换
  * 连接跟踪
  * 端口转换

```md
      Client A        |                 NAT GateWay                 |       Web Server
    192.168.1.2       |      192.168.1.1          202.20.65.5       |      202.20.65.4
+------------------+  |  +------------------+ +------------------+  |  +------------------+
|   HTTP Request   |  |  |   HTTP Request   | |   HTTP Request   |  |  |   HTTP Request   |
| Dst: 202.20.65.4 |---->| Dst: 202.20.65.4 | | Dst: 202.20.65.4 |---->| Dst: 202.20.65.4 |
| Src: 192.168.1.2 |  |  | Src: 192.168.1.2 | | Src: 202.20.65.5 |  |  | Src: 202.20.65.5 |
+------------------+  |  +------------------+ +------------------+  |  +------------------+
                      |            v        SNAT         ^          |            |
                      |            +---------------------+          |            v
+------------------+  |  +------------------+ +------------------+  |  +------------------+
|   HTTP Reply     |  |  |   HTTP Reply     | |   HTTP Reply     |  |  |   HTTP Reply     |
| Dst: 192.168.1.2 |<----| Dst: 192.168.1.2 | | Dst: 202.20.65.5 |<----| Dst: 202.20.65.5 |
| Src: 202.20.65.4 |  |  | Scr: 202.20.65.4 | | Src: 202.20.65.4 |  |  | Src: 202.20.65.4 |
+------------------+  |  +------------------+ +------------------+  |  +------------------+
                      |           ^        DNAT          v          |
                      |           +----------------------+          |
```

上图展示了 NAT 是如何进行__地址转换__的:

1. 私有网中的主机 192.168.1.2 向公共网中的主机 202.20.65.4 发送了1个 HTTP 请求(Dst=202.20.65.4,Src=192.168.1.2)，他是一个 IP 包
2. 当 IP 包经过NAT网关时，NAT Gateway 会将 IP 包的源 IP 转换为 NAT Gateway 的公共 IP 并转发到公共网，此时 IP 包(Dst=202.20.65.4，Src=202.20.65.5)中已经不含任何私有网IP的信息。
3. 由于 IP 包的源 IP 已经被转换成 NAT Gateway 的公共 IP ，Web Server 发出的响应 IP 包(Dst= 202.20.65.5,Src=202.20.65.4)将被发送到 NAT Gateway
4. 这时，NAT Gateway 会将 IP 包的目的 IP 转换成私有网中主机的 IP，然后将 IP 包(Des=192.168.1.2，Src=202.20.65.4)转发到私有网

对于通信双方而言，这种地址的转换过程是完全透明的。如果内网主机发出的请求包未经过 NAT，那么当 Web Server 收到请求包，回复的响应包中的目的地址就是私网IP地址，在 Internet 上无法正确送达，导致连接失败。

在上述过程中，NAT Gateway 在收到响应包后，就需要判断将数据包转发给谁。此时如果子网内仅有少量客户机，可以用静态 NAT 手工指定；但如果内网有多台客户机，并且各自访问不同网站，这时候就需要__连接跟踪(connection track)__。

```md
Client A           NAT Gateway             Server
   |  Dst: 202.20.65.4  |                     |
   |  Src: 192.168.1.2  |                     |
   |------------------->||  Dst: 202.20.65.4  |
   |  SNAT & Record     ||  Src: 202.20.65.5  |
   |                    |+------------------->||
   |                    |                     ||
   |                    |                     ||
   |  Dst: 192.168.1.2 ||<--------------------||
   |  Src: 202.20.65.4 ||   Dst: 202.20.65.5  |
   |<------------------+|   Src: 202.20.65.4  |
   |   DNAT             |                     |
   |     according to   | Track Table:        |
   |       recording    |   192.168.1.2 -> 202.20.65.4
   |                    |                     |
   |                    |                     |
```

还是以上述 Client 访问 Web Server 为例:
当仅有一台客户机访问服务器时，NAT Gateway 只须更改数据包的源 IP 或目的 IP 即可正常通讯。但是如果 Client A 和 Client B 同时访问 Web Server，那么当 NAT Gateway 收到响应包的时候，就无法判断将数据包转发给哪台客户机。此时，NAT Gateway 会在 Connection Track 中加入端口信息加以区分，即__端口转换__。
如果两客户机访问同一服务器的源端口不同，那么在 Track Table 里加入端口信息即可区分，如果源端口正好相同，那么在时行 SNAT 和 DNAT 的同时对源端口也要做相应的转换。

```md
Client A                    Client B NAT Gateway             Server
   |  Dst: 202.20.65.4:80    |                        |
   |  Src: 192.168.1.2:4096  |                        |
   |-------------------------+----------------------->||  Dst: 202.20.65.4:80   |
   |                         |                        ||  Src: 202.20.65.5:4096 |
   |                         |  Dst: 202.20.65.4:80   |+----------------------->||
   |                         |  Src: 192.168.1.3:4096 |                         ||
   |                         |----------------------->||  Dst: 202.20.65.4:80   ||
   |  Dst: 192.168.1.2:4096  |                        ||  Src: 202.20.65.5:4097 ||
   |  Src: 202.20.65.4:80    |                        |+----------------------->||
   |<------------------------+-----------------------+|  Dst: 202.20.65.5:4096  ||
   |                         |                       ||  Src: 202.20.65.4:80    ||
   |                         | Dst: 192.168.1.3:4096 ||<------------------------||
   |                         | Src: 202.20.65.4:80    |                         ||
   |                         |<----------------------+|  Dst: 202.20.65.5:4097  ||
   |  Track Table:           |                       ||  Src: 202.20.65.4:80    ||
   |    192.168.1.2:4096:4096 -> 202.20.65.4         ||<------------------------||
   |    192.168.1.3:4096:4097 -> 202.20.65.4          |                         |
   |    ...                                           |                         |
```

### NAT 表

一段数据流的第一个包会流经这个表，此后其余的包都自动会做相同的处理。实际的 target 分为以下几类:

* DNAT
* SNAT
* MASQUERADE
* REDIRECT

`MASQUERADE`的作用与`SNAT`完全一样。对于每个匹配的包，`MASQUERADE`都会查找可用的 IP 地址，而不像`SNAT`使用的 IP 地址是配置好的。
使用`MASQUERADE`也有好处，就是可以使用通过 PPP、 PPPoE、SLIP 等拨号得到的地址，这些地址可是由 ISP 的 DHCP 随机分配的。

### 状态机制(The state machine)

* 数据包在用户空间的四种状态
* 连接跟踪记录
* TCP / UDP / ICMP 连接
* 缺省的连接操作
* 复杂协议和连接跟踪

#### 数据包在用户空间的四种状态

在内核中，数据包的状态随着协议的不同而不同。但是在内核外部，也就是用户空间里，只有四种状态。

---
State(状态) | Explanation(解释)
--- | ---
NEW | NEW 说明这个包是我们看到的第一个包。意思就是，这是 conntrack 模块看到的某个连接第一个包，它即将被匹配了。比如，我们看到一个SYN 包，是我们所留意的连接的第一个包，就要匹配它。第一个包也可能不是 SYN 包，但它仍会被认为是 NEW 状态。这样做有时会导致一些问题，但对某些情况是有非常大的帮助的。例如，在我们想恢复某条从其他的防火墙丢失的连接时，或者某个连接已经超时，但实际上并未关闭时。
ESTABLISHED | ESTABLISHED 已经注意到两个方向上的数据传输，而且会继续匹配这个连接的包。处于 ESTABLISHED 状态的连接是非常容易理解的。只要发送并接到应答，连接就是 ESTABLISHED 的了。一个连接要从 NEW 变 为ESTABLISHED，只需要接到应答包即可，不管这个包是发往防火墙的，还是要由防火墙转发的。ICMP 的错误和重定向等信息包也被看作是 ESTABLISHED，只要它们是我们所发出的信息的应答。
RELATED | RELATED 是个比较麻烦的状态。当一个连接和某个已处于 ESTABLISHED 状态的连接有关系时，就被认为是 RELATED 的了。换句话说，一个连接要想是 RELATED 的，首先要有一个 ESTABLISHED 的连接。这个 ESTABLISHED 连接再产生一个主连接之外的连接，这个新的连接就是 RELATED 的了，当然前提是 conntrack 模块要能理解 RELATED。ftp 是个很好的例子，FTP-data 连接就是和FTP-control有RELATED的。还有其他的例子，比如，通过 IRC 的 DCC 连接。有了这个状态，ICMP 应答、FTP 传输、DCC 等才能穿过防火墙正常工作。注意，大部分还有一些 UDP 协议都依赖这个机制。这些协议是很复杂的，它们把连接信息放在数据包里，并且要求这些信息能被正确理解。
INVALID | INVALID 说明数据包不能被识别属于 哪个连接或没有任何状态。有几个原因可以产生这种情况，比如，内存溢出，收到不知属于哪个连接的 ICMP 错误信息。一般地，我们 DROP 这个状态的任何东西。
UNTRACKED | 简单的说，若一个包在 raw 表中被表记为 NOTRACK target，那么在状态机制中就会显示 UNTRACKED。这也意味着所有 RELATED 连接不会被看到。因此当处理 UNTRACKED 连接时候，有些东西已定要小心，因为像依赖于 ICMP 等的信息不会在状态机制中显示。

#### 连接跟踪记录

若安装了连接跟踪模块(ip_conntrack)，`#tail /proc/net/ip_conntrack`，则会看到类似的消息:

> tcp      6 117 SYN_SENT src=192.168.1.6 dst=192.168.1.9 sport=32775 \
>     dport=22 [UNREPLIED] src=192.168.1.9 dst=192.168.1.6 sport=22 \
>     dport=32775 [ASSURED] use=2

conntrack模块维护的所有信息(实际出现的信息中，ASSURED 与 UNREPLIED / SYN_SENT 并不同时出现)都包含在这个例子中了，通过塔可以知道特定的连接处在何种状态。
首先显示的是协议，这里是 TCP，接着是十进制的6(TCP协议类型代码是6)，之后的117是这条 conntrack 的生存时间，它会被由规律的消耗，直到收到这个连接的更过的包；此时，这个值就被设为当时那个状态的缺省值。接下来的是这个连接在当前时间点的状态。上面的例子说明这个包处在状态 SYN_SENT，这个值是 iptables 显示的，以便我们好理解，而内部用的值稍有不同。SYN_SENT 说明我们正在观察的这个连接只在一个方向发送了一 TCP SYN 包。再下面是源地址、目的地址、源端口和目的端口。其中有个特殊的词 UNREPLIED，说明这个连接还没有收到任何回应。最后，是希望接收的应答包的信息，他们的地址和端口和前面是相反的。
当连接出现双向流量的时候，conntrack 会擦除 [UNREPLIED] 标志，然后重置它。这个名词([UNREPLIED])告诉我们还没有建立双向连接，建立连接后，它将会被末尾的 [ASSURED] 替代。
[ASSURED] 表示当连接跟踪表满时，这条记录不会被擦除。

#### TCP / UDP / ICMP 连接

* TCP -- 有状态协议
  * 连接的建立
    * 一个 TCP 连接是经过三次握手协商才建立起来。整个会话由一个 SYN 包开始，然后是一个 SYN/ACK 包，最后是一个 ACK 包。此时，会话才建立成功，能够发送数据。
  * 连接的关闭
    * 一般情况下，在发出最后一个 ACK 包之前，连接(双向连接)是不会关闭。
    * 连接关闭后，进入 TIME_WAIT 状态，缺省时间是两分钟(这个时间存在的原因是为了让数据包通过各种规则的检查、通过拥挤的路由器，从而到达目的地)。
    * 如果一个包是被 RST 重置的，就直接变为 CLOSE 了。RST 包是不需要确认的，它会直接关闭连接。
  * 连接状态与内部状态

```md
Client         Firewall         Server
 SYN    --+
          +--     NEW     --+
                            +-- SYN/ACK
          +-- ESTABLISHED --+
 SCK    --+

FIN/ACK --+
          +-- EATABLISHED --+
                            +-- ACK
          +-- ESTABLISHED --+
 CLOSE  --+

                            +-- FIN/ACK
          +-- ESTABLISHED --+
 ACK    --+
          +--    CLOSED   --+
                            +-- CLOSED
```

Ubuntu 16.04.1 LTS下，TCP 连接建立时 conntrack 显示的信息

>     [NEW] tcp      6 120 SYN_SENT src=192.168.0.3 dst=104.236.143.213 sport=53630 dport=80 [UNREPLIED] src=104.236.143.213 dst=192.168.0.3 sport=80 dport=53630
> [UPDATE] tcp      6 60 SYN_RECV src=192.168.0.3 dst=104.236.143.213 sport=53630 dport=80 src=104.236.143.213 dst=192.168.0.3 sport=80 dport=53630
> [UPDATE] tcp      6 432000 ESTABLISHED src=192.168.0.3 dst=104.236.143.213 sport=53630 dport=80 src=104.236.143.213 dst=192.168.0.3 sport=80 dport=53630 [ASSURED]

内部状态

---
State | Timeout value
--- | ---
NONE | 30 minutes
ESTABLISHED | 5 days
SYN_SENT | 2 minutes
SYN_RECV | 60 seconds
FIN_WAIT | 2 minutes
TIME_WAIT | 2 minutes
CLOSE | 10 seconds
CLOSE_WAIT | 12 hours
LAST_ACK | 30 seconds
LISTEN | 2 minutes

这些值并不是绝对的，它们可能随着内核版本的变化而变化，当然也可以通过修改 proc file-system 里面的系统来修改。

* UDP -- 无状态协议
  * 尽管 UDP 无状态，因为它没有任何连接建立和关闭的过程，而且大部分是无序列号的。但内核仍然可以对 UDP 连接设置状态。
  * UDP 连接的状态

```md
  Client           Firewall          Server
UDP Package --+
              +--    NEW      --+
                                +-- UDP Packet
              +-- ESTABLISHED --+
...         --+
```

这是 UPD 连接建立时候显示的信息

>    [NEW] udp      17 30 src=127.0.0.1 dst=127.0.1.1 sport=48970 dport=53 [UNREPLIED] src=127.0.1.1 dst=127.0.0.1 sport=53 dport=48970
> [UPDATE] udp      17 29 src=127.0.0.1 dst=127.0.1.1 sport=48970 dport=53 src=127.0.1.1 dst=127.0.0.1 sport=53 dport=48970
> [UPDATE] udp      17 180 src=127.0.0.1 dst=127.0.1.1 sport=48970 dport=53 src=127.0.1.1 dst=127.0.0.1 sport=53 dport=48970 [ASSURED]

[UNREPLIED] 标记说明还未收到回应。一旦收到第一个包的回应，[UNREPLIED] 标记就会被删除，连接就被认为是 ESTABLISHED，但在记录中并不显示 ESTABLISHED 标记。

* ICMP -- 无状态协议,用来控制而不是建立连接
  * 四种类型两种状态
    * 回显请求和应答(Echo request and reply)
    * 时间戳请求和应答(Timestamp request and reply) -- 废弃
    * 信息请求和应答(Information request and reply) -- 废弃
    * 地址掩码请求和应答(Address mask request and reply)
  * IMCP 连接的状态

```md
 Client          Firewall            Server
ICMP Echo -----     NEW     ----- ICMP Echo Request
Reply     ----- ESTABLISHED ----- Client Processing
```

主机向目标发送一个回显请求，防火墙就认为这个包处于 NEW 状态。目标回应一个回显应答，防火墙就认为包处于 ESTABLISHED 状态了。

>     [NEW] icmp     1 30 src=192.168.0.3 dst=202.117.120.246 type=8 code=0 id=10486 [UNREPLIED] src=202.117.120.246 dst=192.168.0.3 type=0 code=0 id=10486
>  [UPDATE] icmp     1 29 src=192.168.0.3 dst=202.117.120.246 type=8 code=0 id=10486 src=202.117.120.246 dst=192.168.0.3 type=0 code=0 id=10486
> [DESTROY] icmp     1 src=192.168.0.3 dst=202.117.120.246 type=8 code=0 id=10486 src=202.117.120.246 dst=192.168.0.3 type=0 code=0 id=10486

```md
 Client       Firewall       Server
  SYN   -----   NEW   --+
                        +-- ICMP Net Unreachable
 Client ----- RELATED --+
 Aborts

UDP Packet --   NEW   --+
                        +-- ICMP Net Prohibited
Client     -- RELATED --+
Aborts
```

IMCP 一个非常重要的作用是告诉 UDP / TCP 连接或正在建立的连接发生了什么，这时候 ICMP 的应答被认为是 RELATED 的。上图的主机不可达(Network Unreachable, TCP 连接)与 网络被禁止(Net Prohibited, UDP 连接)就是例子。
当发送一个 SYN 包到某一地址，防火墙认为它的状态是 NEW。但是目标网络有问题不可达，路由器就会返回网络不可达的信息，这就是 RELATED 的。
当 UDP 连接遇到问题时，同样也会用响应的 ICMP 信息返回，它们的状态也是 RELATED。

#### 缺省的连接操作

某些情况下，连接追踪机制并不知道如何操作一些特殊的协议，尤其在它不了解这个协议或不知道这个协议如何工作的时候。这种情况下，conntrack使用缺省操作。
缺省操作很像对 UDP 的操作: 第一个包被认作 NEW，其后的应答包等数据都是 ESTABLISHED。

#### 复杂协议和连接跟踪

有些协议连接跟踪机制很难正确地跟踪它们，比如 FTP，它们都在数据包的数据域里携带某些信息，这些信息用于建立其他的连接。因此需要一些特殊的 connection tracking helper 来完成这项工作。
conntrack helper 可以被静态地编译进内核，也可以作为模块，但要该命令临时加载: `modprobe nf_conntrack_*`。若要开机自动加载则需要编辑`/etc/modprobe.d`文件夹下文件。
注意连接跟踪并不处理 NAT，因为要对连接做 NAT 就需要增加相应的模块。
conntrack helper 的命名原则通常为`nf_conntrack_<protocol>`，即`nf_conntrack_+协议`，NAT 模块的命名类似:`nf_nat_<protocol>`。它们的位置在`/lib/modules/<kernel-version>/kernel/net/netfilter/`。

注:

1. 从[这里](http://www.spinics.net/lists/netfilter/msg43535.html)可知: `ip_conntrack_*`已经被重命名为`nf_conntrack_*`。
2. [这里](http://shorewall.net/FTP.html)指出低于2.6.19(包括2.6.19)版本的内核，helper 与 nat 模块的位置在`/lib/modules/<kernel-version>/kernel/net/ipv4/netfilter/`。

## 规则是怎样炼成的

> 抄还是不抄？这是一个问题。
> 已经写(抄)了三天啦，了解了工作原理，语法也就不是那么重要了...

书写规则的语法就这么一句

```bash
iptables [-t table] command [match] [target/jump]
```

* [-t table]
  * 可选的参数，默认为 __filter__
* command
  * 那么多种，常用的也就那几个，`man iptables`好啦
  * `--modprobe` -- 探测并装载所使用的模块
  * `-v, --verbose`
    * 可用的选项与命令: `--list, --append, --insert, --delete, --replace`
    * 这个选项使输出详细化。与`--list`连用时，输出中包括网络接口的地址、规则的选项、TOS掩码、字节和包计数器，其中计数器是以K、M、G(这里用的是10的幂而不是2的幂哦)为单位的。如果想知道到底有多少个包、多少字节，还要用到选项`-x,--exact(精确的)`。如果`-v`和`--append,--insert,--delete,--replace`连用，iptables 会输出详细的信息告诉你规则是如何被解释的、是否正确地插入等等
* [match]
  * 通用匹配: -p / -s / -d / -i / -o / -f,--fragment(匹配被分片的包的第二片或及以后的部分)
  * 隐含匹配
    * 这种匹配操作是自动或隐含地装载入内核的。例如我们使用`--protocol`tcp 时，不需再装入任何东西就可以匹配只有 IP 包才有的一些特点。
    * TCP matches
      * --sport / --dport
        * 不指定此选项，则暗示所有端口
        * 使用服务名或端口号，但名字必须是在`/etc/services`中定义
        * 可以使用连续的端口，如`--source-port 20:80`
        * 可以省略第一个端口号，默认为0，如`--source-port :80`
        * 可以省略第二个端口号，默认为65535
        * 在端口号前加`!`表示取反(注意空格)，如`--source-port ! 22 / --source-port ! 22:80`
        * 注意: 这个匹配操作不能识别不连续的端口列表，如`--source-port ! 22,36,80`。这样的操作需要多端口匹配扩展来完成
      * --tcp-flags
        * 匹配特定的 TCP 标记
    * UDP matches
      * --sport / --dport
    * ICMP matches
      * --icmp-type
    * SCTP matches
      * Stream Control Transmission Protocol
  * 显式匹配
    * 显式匹配必须用`-m,--match`装载
    * addrtype match
    * aH/ESP match
    * comment match
    * connmark match
    * __conntrack__ math
      * 连接追踪匹配
      * --ctstate
        * INVALID / ESTABLISHED / NEW / RELATED / SNAT / DNAT
        * 比如`iptables -A INPUT -p tcp -m conntrack --ctstate RELATED`
      * --ctproto
        * 匹配协议
      * --ctorigsrc
        * conntrack origin source
        * IP 或 网段
      * --ctorigdst
        * conntrack origin destination
      * --ctreplsrc
        * conntrack replace source
      * --ctrepldst
        * conntrack replace destination
      * --ctstatus
        * NONE / EXPECTED / SEEN_REPLY / ASSURED
      * --ctexpire
        * conntrack expiration timer，单位秒
    * dscp match
    * ecn match
    * hashlimit match
    * helper match
      * contrack helper
    * IP range match
      * --src-range
        * `iptables -A INPUT -p tcp -m iprange --src-range 192.168.1.13-192.168.2.19`
      * --dst-range
    * length match
    * __limit__ match
      * limit match 的工作方式就像一个单位大门口的保安，当有人要进入时，需要找他办理通行证。早上上班时，保安手里有一定数量的通行证，来一个人，就签发一个，当通行证用完后，再来人就进不去了，但他们不会等，而是到别的地方去(在 iptables 里，这相当于一个包不符合某条规则，就会由后面的规则来处理，如果都不符合，就由缺省的策略处理)。但有个规定，每隔一段时间保安就要签发一个新的通行证。这样，后面来的人如果恰巧赶上，也就可以进去了。如果没有人来，那通行证就保留下来，以备来的人用。如果一直没人来，可用的通行证的数量就增加了，但不是无限增大的，最多也就是刚开始时保安手里有的那个数量。也就是说，刚开始时，通行证的数量是有限的，但每隔一段时间 就有新的通行证可用。limit match 有两个参数就对应这种情况，--limit-burst 指定刚开始时有多少通行证可用，--limit 指定要隔多长时间才能签发一个新的通行证。要注意的是，我这里强调的是“签发一个新的通行证”，这是以 iptables 的角度考虑的。在你自己写规则时，就要从这个角度考虑。比如，你指定了 --limit 3/minute --limit-burst 5，意思是开始时有5个通行证，用完之后每20秒增加一个(这就是从 iptables 的角度看的，要是以用户的角度看，说法就是每一分钟增加三个或者每分钟只能过三个)。
      * --limit
        * 为 limit match 设置最大平均匹配速率
        * 数值/时间
        * 时间单位 second / minute / hour / day
        * 默认值 3/hour
      * --limit-burst
        * 定义的是 limit match 的峰值，就是在单位时间内最多可匹配几个包
        * 默认值 5
    * __mac__ match
      * --mac-source
    * mark match
    * __multiport__ match
      * --source-port
        * `iptables -A INPUT -p tcp -m multiport --source-port 22,53,80,110`
      * --destination-port
      * --port
        * source port + destination port
    * owner match
      * --cmd-owner / --uid-owner / --gid-owner / --pid-owner / --sid-owner
    * racket type match
    * realm match
    * recent match
    * __state__ match
      * --state
        * INVALID / ESTABLISHED / NEW / RELATED
    * tcpmss match
    * tos match
    * ttl match
    * unclean match
* [target/jump]
  * 决定符合条件的包到何处去
  * jump
    * 目标是一个在同一个表内的链
    * 一个例子
      * 比如在 filter 表中建立一个名为 tcp_packets的链`iptables -N tcp_packets`
      * 然后把它作为 jump 的目标`iptables -A INPUT -p tcp -j tcp_packets`
  * target
    * 目标是一个具体的操作
    * __ACCEPT__ target
    * CLASSIFY target
    * CLUSTERIP target
    * CONNMARK target
    * CONNSECMARK target
    * __DNAT__ target
      * --to-destination
        * `--to-destination 192.168.1.1`
        * `--to-destination 192.168.1.1-192.168.1.10`
        * `--to-destination 192.168.1.1:80`
        * `--to-destination 192.168.1.1:80-100`
        * 只有指定 TCP 或 UDP 协议才能指定端口
    * __DROP__ target
    * DSCP target
    * ECN target
    * __LOG__ target
      * --log-level
        * debug / info / notice / warning / warn / err / error / crit / alert / emerg / panic
      * --log-prefix
        * 添加的前缀
      * --log-tcp-sequence
        * 把包的TCP序列号和其他日志信息一起记录下来
      * --log-tcp-options
        * 记录TCP包头中的不同的选项
      * --log-ip-options
        * 记录 IP 包头中的选项
    * MARK target
    * __MASQUERADE__ target
      * --to-ports
      * 指定 TCP 或 UDP 前提下，设置外出包可使用的端口
      * `--to-ports 1025`
      * `--to- ports 1024-3000`
    * MIRROR target
    * NETMAP target
      * SNAT / DNAT 的新实现，提供了对整个私有网络1:1的 NAT 转换功能
      * --to
        * `iptables -t mangle -A PREROUTING -s 192.168.1.0/24 -j NETMAP --to 10.5.6.0/24`
    * NFQUEUE target
    * NOTRACK target
    * QUEUE target
    * __REDIRECT__ target
      * --to-ports
      * `iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 8080`
    * __REJECT__ target
      * 与 DROP 的区别在于除了阻塞包之外，还向发送者返回错误信息
      * --reject-with
        * icmp-net-unreachable / icmp-host-unreachable / icmp-port-unreachable / icmp-proto-unreachable / icmp-net-prohibited / icmp-host-prohibited
        * 默认 port-unreachable
    * RETURN target
    * SAME target
      * 功能与 SNAT 差不多。不同之处:
        * `iptables -t mangle -A PREROUTING -s 192.168.1.0/24 -j SAME --to 10.5.6.7-10.5.6.9`
          * 一个 /24 网段，3个 IP 地址。如果 192.168.1.20 第一次通过 10.5.6.7 的 IP 地址，那么随后的包也会通过这个地址
      * --to
    * SECMARK target
    * __SNAT__ target
      * --to-source
    * TCPMSS target
    * TOS target
    * TTL target
    * ULOG target

## iptables 流程图的详细解释

### 以本地为目标的包

---
Step | Table | Chain | Comment(注释)
--- | --- | --- | ---
1 | | | 在线路上传输(比如 Internet)
2 | | | 进入接口(比如 eth0)
3 | raw PREROUTING | 连接追踪(connection tracking)起作用前，这条链起作用。例如，它被用于设置一个特定的连接，这条连接不被连接跟踪代码处理
4 | | | 这是当连接跟踪代码起作用的时候，它将会在[The state machine](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#STATEMACHINE)讨论
5 | mangle | PREROUTING | 这条链通常被用于对特定数据包修改(比如，改变 TOS 等)
6 | nat | PREROUTING | 这个链主要用来做 DNAT。不要在这个链做过滤操作，因为某些情况下包会溜过去
7 | | | 路由判断(比如，包是发往本地的，还是需要被转发的）
8 | mangle | INPUT | 在路由之后，被送往本地程序之前，在这里对特定数据包修改
9 | filter | INPUT | 所对这些包的过滤条件就设在这里。有以本地为目的的数据包都要经过这个链，不管他们从哪里来
10 | | | 到达本地进程或应用了(比如服务程序或客户程序)

### 以本地为源的包

(作者认为，由图可知，7与8应该互换位置)

---
Step | Table | Chain | Comment(注释)
--- | --- | --- | ---
1 | | | 本地进程/应用(比如服务程序或客户程序)
2 | | | 路由判断: 要使用哪个源地址(source ip)、外出接口(eth0之类)，还有其他一些信息
3 | raw | OUTPUT | 对于本地生成的包，在连接追踪起作用前，这里是你可以做工作的的地方。例如，你可以为连接做标记以便它们不会被追踪
4 | | | 对本地生成的包，这里是连接追踪起作用的地方。比如状态的改变等等。它将会在[The state machine](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#STATEMACHINE)讨论
5 | mangle | OUTPUT | 在这可以对特定数据包修改。建议不要在这做过滤，可能有副作用哦
6 | nat | OUTPUT | 这个链对从防火墙本身发出的包进行 DNAT 操作
7 | | | 路由判断: 因为先前的 mangle 与 cat 可能改变了数据包的路由信息
8 | filter | OUTPUT | 对本地发出的包过滤
9 | mangle | POSTROUTING | 在离开主机之前，决定对包进行怎样的处理。两种包会经过这里: 防火墙自身产生的包、被转发的包
10 | nat | POSTROUTING | 在这里做SNAT。但不要在这里做过滤，因为有副作用，而且有些包是会溜过去的，即使你用了 DROP 策略
11 | | | 离开接口(比如： eth0)
12 | | | 在线路上传输(比如，Internet)

### 被转发的包

---
Step | Table | Chain | Comment
--- | --- | --- | ---
1 | | | 在线路上传输(比如，Internet)
2 | | | 进入接口(比如， eth0)
3 | raw | PREROUTING | 这里可以进行设置不被连接追踪的操作
4 | | | 这里是非本地连接产生连接追踪的地方，这也会在[The state machine](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#STATEMACHINE)章节讨论
5 | mangle | PREROUTING | 修改数据包，比如改变 TOS 等
6 | nat | PREROUTING |这个链主要用来做 DNAT。不要在这个链做过虑操作，因为某些情况下包会溜过去。稍后会做 SNAT
7 | | | 路由判断，比如，包是发往本地的，还是要转发的
8 | mangle | FORWARD | 包继续被发送至 mangle 表的 FORWARD 链。当我们希望在最初的路由之后、数据包发出前的下一个路由之前修改数据包才需要
9 | filter | FORWARD | 包继续被发送至这条 FORWARD 链。只有需要转发的包才会走到这里，并且针对这些包的所有过滤也在这里进行。注意，所有要转发的包都要经过这里，不管是外网到内网的还是内网到外网的。在你自己书写规则时，要考虑到这一点
  | | | (作者个人认为有必要加上这行: 由图可知，这里是路由决策)
10 | mangle | POSTROUTING | 这个链也是针对一些特殊类型的包(译者注: 参考第8步，我们可以发现，在转发包时，mangle 表的两个链都用在特殊的应用上)。这一步 mangle 是发生在所有路由判断结束之后，但此时包还在本地上
11 | nat | POSTROUTING | 这个链就是用来做 SNAT 的，当然也包括 Masquerade (伪装)。但不要在这儿做过滤，因为某些包即使不满足条件也会通过
12 | | | 离开接口(比如： eth0)
13 | | | 又在线路上传输了(比如，LAN)

### 总结

该流程图描述链了在任何接口上收到的网络数据包是按照怎样的顺序穿过表的交通管制链。

![tables traverse](/images/16/08/tables_traverse.jpg)

## iptables 总结

去它的总结，都歇菜了，哪来的心思写总结←\_←。
主要的资料来源于下面的两本《Iptables Tutorial》。中文版比较旧，而英文版也有一些错误。还参考了鸟哥与 ArchLinux 文档。
讲真，ArchLinux 文档基本概念部分介绍得不错，但是竟然也有错的 =\_= 渣翻译，嫌弃.jpg(于是我把 *遍历链(Traversing Chains)* 部分给修改了下，改正了一些错误，添加上自己的理解。虽然我翻译得也渣...)
完结!
喔，对了，上篇遗留的问题呢？从上面找吧，别打我...

## 参考

[ArchLinux Iptables (简体中文)](https://wiki.archlinux.org/index.php/Iptables_%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)
[鳥哥的 Linux 私房菜 第九章、防火牆與 NAT 伺服器](http://linux.vbird.org/linux_server/0250simple_firewall.php)
[iptables 指南](https://www.frozentux.net/iptables-tutorial/cn/iptables-tutorial-cn-1.1.19.html)
[Iptables Tutorial](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html)
