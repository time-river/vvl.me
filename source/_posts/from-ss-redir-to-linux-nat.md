---
title: 从ss-redir的实现到Linux NAT
date: 2018-06-09 14:36:54
tags: ['nat', 'network', 'linux']
---

## 引子

今年4月，在家的时候意外看到了[ss-redir 透明代理][1]，对其中的详细说明持有怀疑态度：

> 由于笔者才疏学浅，刚开始居然以为 TCP 透明代理和 UDP 透明代理是一样的，只要无脑 REDIRECT 到 ss-redir 监听端口就可以了。
> ...
> 但是，上面这种情况只针对 TCP；对于 UDP，如果你做了 DNAT，就无法再获取数据包的原目的地址和目的端口了。

于是对此专门做了一番调研。整篇分为三部分：第一部分是我对上述叙述的调研结果，第二部分讨论TPROXY，第三部分叙述一些NAT的知识。

## ss-redir中的UDP REDIRECT问题

ss-redir的原理很简单：使用iptables对PREROUTING与OUTPUT的TCP/UDP流量进行REDIRECT（REDIRECT是DNAT的特例），ss—redir在捕获网络流量后，通过一些*技术手段*获取REDIRECT之前的目的地址（dst）与端口（port），连同网络流量一起转发至远程服务器。

针对TCP连接，的确是因为Linux Kernel连接跟踪机制的实现才使获取数据包原本的dst和port成为可能，但这种连接跟踪机制并非只存在于TCP连接中，UDP连接同样存在，`conntrack -p udp`便能看到UDP的连接跟踪记录。内核中有关TCP与UDP的NAT源码[/net/netfilter/nf_nat_proto_tcp.c][2]和[/net/netfilter/nf_nat_proto_udp.c][3]几乎一模一样，都是根据NAT的类型做SNAT或DNAT。

那这究竟是怎么一回事？为什么对于UDP连接就失效了呢？

回过头来看看ss-redir有关获取TCP原本的dst和port的源码，核心函数是`getdestaddr`：

```c
static int
getdestaddr(int fd, struct sockaddr_storage *destaddr)
{
    socklen_t socklen = sizeof(*destaddr);
    int error         = 0;

    error = getsockopt(fd, SOL_IPV6, IP6T_SO_ORIGINAL_DST, destaddr, &socklen);
    if (error) { // Didn't find a proper way to detect IP version.
        error = getsockopt(fd, SOL_IP, SO_ORIGINAL_DST, destaddr, &socklen);
        if (error) {
            return -1;
        }
    }
    return 0;
}
```

在内核源码中搜了下有关`SO_ORIGINAL_DST`的东西，看到了[getorigdst][4]：

```c
/* Reversing the socket's dst/src point of view gives us the reply
   mapping. */
static int
getorigdst(struct sock *sk, int optval, void __user *user, int *len)
{
    const struct inet_sock *inet = inet_sk(sk);
    const struct nf_conntrack_tuple_hash *h;
    struct nf_conntrack_tuple tuple;

    memset(&tuple, 0, sizeof(tuple));

    lock_sock(sk);
    tuple.src.u3.ip = inet->inet_rcv_saddr;
    tuple.src.u.tcp.port = inet->inet_sport;
    tuple.dst.u3.ip = inet->inet_daddr;
    tuple.dst.u.tcp.port = inet->inet_dport;
    tuple.src.l3num = PF_INET;
    tuple.dst.protonum = sk->sk_protocol;
    release_sock(sk);

    /* We only do TCP and SCTP at the moment: is there a better way? */
    if (tuple.dst.protonum != IPPROTO_TCP &&
        tuple.dst.protonum != IPPROTO_SCTP) {
        pr_debug("SO_ORIGINAL_DST: Not a TCP/SCTP socket\n");
        return -ENOPROTOOPT;
    }
```

*We only do TCP and SCTP at the moment*。Oh，shit！只针对TCP与SCTP才能这么做，并非技术上不可行，只是人为地阻止罢了。

## TPROXY

为了在redirect UDP后还能够获取原本的dst和port，ss-redir采用了TPROXY。Linux系统有关TPROXY的设置是以下三条命令：

```bash
# iptables -t mangle -A PREROUTING -p udp -j TPROXY --tproxy-mark 0x2333/0x2333 --on-ip 127.0.0.1 --on-port 1080
# ip rule add fwmark 0x2333/0x2333 pref 100 table 100
# ip route add local default dev lo table 100
```

获取原本的dst和port的相关源码如下:

```c
static int
get_dstaddr(struct msghdr *msg, struct sockaddr_storage *dstaddr)
{
    struct cmsghdr *cmsg;

    for (cmsg = CMSG_FIRSTHDR(msg); cmsg; cmsg = CMSG_NXTHDR(msg, cmsg)) {
        if (cmsg->cmsg_level == SOL_IP && cmsg->cmsg_type == IP_RECVORIGDSTADDR) {
            memcpy(dstaddr, CMSG_DATA(cmsg), sizeof(struct sockaddr_in));
            dstaddr->ss_family = AF_INET;
            return 0;
        } else if (cmsg->cmsg_level == SOL_IPV6 && cmsg->cmsg_type == IPV6_RECVORIGDSTADDR) {
            memcpy(dstaddr, CMSG_DATA(cmsg), sizeof(struct sockaddr_in6));
            dstaddr->ss_family = AF_INET6;
            return 0;
        }
    }

    return 1;
}

int
create_server_socket(const char *host, const char *port)
{
    ...
#ifdef MODULE_REDIR
        if (setsockopt(server_sock, SOL_IP, IP_TRANSPARENT, &opt, sizeof(opt))) {
            ERROR("[udp] setsockopt IP_TRANSPARENT");
            exit(EXIT_FAILURE);
        }
        if (rp->ai_family == AF_INET) {
            if (setsockopt(server_sock, SOL_IP, IP_RECVORIGDSTADDR, &opt, sizeof(opt))) {
                FATAL("[udp] setsockopt IP_RECVORIGDSTADDR");
            }
        } else if (rp->ai_family == AF_INET6) {
            if (setsockopt(server_sock, SOL_IPV6, IPV6_RECVORIGDSTADDR, &opt, sizeof(opt))) {
                FATAL("[udp] setsockopt IPV6_RECVORIGDSTADDR");
            }
        }
#endif
    ...
}
```

大意就是在mangle表的PREROUTING中为每个UDP数据包打上0x2333/0x2333标志，之后在路由选择中将具有0x2333/0x2333标志的数据包投递到本地环回设备上的1080端口；对监听0.0.0.0地址的1080端口的socket启用`IP_TRANSPARENT`标志，使IPv4路由能够将非本机的数据报投递到传输层，传递给监听1080端口的ss-redir。[`IP_RECVORIGDSTADDR`][5]与`IPV6_RECVORIGDSTADDR`则表示获取送达数据包的dst与port。

可问题来了：要知道mangle表并不会修改数据包，那么TPROXY是如何做到在不修改数据包的前提下将非本机dst的数据包投递到换回设备上的1080端口呢？

与之有关的内核源码我没有完全看懂。根据[TProxy - Transparent proxying, again][6]和[2.6.26时代的patch set][7]，在netfilter中的PREROUTING阶段，将符合规则的IP数据报skb_buff中的成员sk（[它表示数据包从属的套接字][10]）给[assign_sock][11]，这个sock就是利用iptables TPROXY的target信息找到的：

```c
// /net/netfilter/xt_TPROXY.c
static unsigned int
tproxy_tg4(struct net *net, struct sk_buff *skb, __be32 laddr, __be16 lport,
       u_int32_t mark_mask, u_int32_t mark_value)
{
    ...
    /* UDP has no TCP_TIME_WAIT state, so we never enter here */
    if (sk && sk->sk_state == TCP_TIME_WAIT)
        /* reopening a TIME_WAIT connection needs special handling */
        sk = tproxy_handle_time_wait4(net, skb, laddr, lport, sk);
    else if (!sk)
        /* no, there's no established connection, check if
         * there's a listener on the redirected addr/port */
        sk = nf_tproxy_get_sock_v4(net, skb, hp, iph->protocol,
                       iph->saddr, laddr,
                       hp->source, lport,
                       skb->dev, NFT_LOOKUP_LISTENER);

    /* NOTE: assign_sock consumes our sk reference */
    if (sk && tproxy_sk_is_transparent(sk)) {
        /* This should be in a separate target, but we don't do multiple
           targets on the same rule yet */
        skb->mark = (skb->mark & ~mark_mask) ^ mark_value;

        pr_debug("redirecting: proto %hhu %pI4:%hu -> %pI4:%hu, mark: %x\n",
             iph->protocol, &iph->daddr, ntohs(hp->dest),
             &laddr, ntohs(lport), skb->mark);

        nf_tproxy_assign_sock(skb, sk);
        return NF_ACCEPT;
    }
    ...
}
```

sock是根据四元组*saddr*, *sport*, *daddr*, *dport*来选择的，其中*saddr*与*sport*来自skb_buff，另外俩为target所定义。没搞懂的地方在于：在`ip_rcv_finish`中，是怎样将数据包投递到上层协议以及指定端口的？

目前的猜测如下：

```md
// kernel version 4.17
- ip_route_input_noref
  - ip_route_input_rcu
    - ip_route_input_slow
      - fib_lookup
        - fib_table_lookup
          - res->type = fa->fa_type;
    - if (res->type == RTN_LOCAL) {
            ...
            goto local_input;
      }
    - skb_dst_set_noref(skb, &rth->dst);
      - rth = rt_dst_alloc(l3mdev_master_dev_rcu(dev) ? : net->loopback_dev,
               flags | RTCF_LOCAL, res->type,
               IN_DEV_CONF_GET(in_dev, NOPOLICY), false, do_cache);
          - if (flags & RTCF_LOCAL)
               rt->dst.input = ip_local_deliver;
```

通过查找路由表确定`res-type`的类型为`RTN_LOCAL`，goto到local_input，进而调用`rt_dst_alloc`，形参参数`(flag & RTCF_LOCAL) == true`，设置了`rt->dst.input`是`ip_local_deliver`。`ip_local_deliver`中使用协议回调函数`handler`来进一步处理数据包。

进入传输层后，对IPv4下的UDP协议来说，它的`handler`为[`udplite_rcv`（v4.17)][12]，通过调用`skb_steal_sock`来获取sock，这个sock与TPROXY中在`nf_tproxy_get_sock_v4`获取到的sock是一致的。sock的判断是根据`compute_score`计算的得分来选择的，分高者赢。

```md
// UDP
.handler = udplite_rcv

- udplite_rcv
  - __udp4_lib_rcv
    - sk = skb_steal_sock(skb);
      ...
      ret = udp_queue_rcv_skb(sk, skb);

// TPROXY
- nf_tproxy_get_sock_v4
  - udp4_lib_lookup
    - __udp4_lib_lookup
      - __udp4_lib_lookup_skb
        - __udp4_lib_lookup
          - udp4_lib_lookup2

// /net/ipv4/udp.c: get sock
static struct sock *udp4_lib_lookup2(struct net *net,
                 __be32 saddr, __be16 sport,
                 __be32 daddr, unsigned int hnum,
                 int dif, int sdif, bool exact_dif,
                 struct udp_hslot *hslot2,
                 struct sk_buff *skb)
{
    struct sock *sk, *result;
    int score, badness;
    u32 hash = 0;

    result = NULL;
    badness = 0;
    udp_portaddr_for_each_entry_rcu(sk, &hslot2->head) {
        score = compute_score(sk, net, saddr, sport,
                      daddr, hnum, dif, sdif, exact_dif);
        if (score > badness) {
            if (sk->sk_reuseport) {
                hash = udp_ehashfn(net, daddr, hnum,
                           saddr, sport);
                result = reuseport_select_sock(sk, hash, skb,
                            sizeof(struct udphdr));
                if (result)
                    return result;
            }
            badness = score;
            result = sk;
        }
    }
    return result;
}
```

有趣的是，在查找资料过程中，我还看到了这篇文章：[TPROXY之殇-NAT设备加代理的恶果][13]。

最后来回到原点，谈一谈NAT。

## NAT

根据[RFC 2663][16]，NAT分为基本网络地址转换（Basic NAT，also called a one-to-one NAT）和网络地址端口转换（NAPT（network address and port translation），other names include port address translation (PAT), IP masquerading, NAT overload and many-to-one NAT）两类。基本网络地址转换仅支持地址转换，不支持端口映射，要求每一个内网地址都对应一个公网地址；网络地址端口转换支持端口的映射，允许多台主机共享一个公网地址。支持端口转换的NAT又可以分为两类：源地址转换（SNAT）和目的地址转换（DNAT）。前一种情形下发起连接的计算机的IP地址将会被重写，使得内网主机发出的数据包能够到达外网主机。后一种情况下被连接计算机的IP地址将被重写，使得外网主机发出的数据包能够到达内网主机。

Linux下，iptables的SNAT除了SNAT target外，还有MASQUERADE、NETMAP。MASQUERADE target与SNAT差不多，区别主要是MASQUERADE能够自动选择出口网卡的动态IP地址，而NETMAP则是只转换IP地址。DNAT的target有DNAT、REDIRECT，区别是REDIRECT只进行端口转换而IP地址并不会改变。还有一类target叫做NETMAP，只转换IP地址，同时拥有SNAT与DNAT的功能。讨论Linux kernelNAT实现的文章不少，比如[iptables深入解析-nat篇][17]，在此不想谈论这些，而是其他的一些东西。

>> 时隔一个半月，继续...

- REDIRECT只进行端口映射，并不改变IP地址。这点可以在[源码](https://elixir.bootlin.com/linux/v4.17/source/net/netfilter/nf_nat_redirect.c)中看到

```md
// /net/netfilter/nf_nat_redirect.c, function: nf_nat_redirect_ipv4
    /* Local packets: make them go to loopback */
    if (hooknum == NF_INET_LOCAL_OUT) {
        newdst = htonl(0x7F000001);
    } else {
        struct in_device *indev;
        struct in_ifaddr *ifa;

        newdst = 0;

        rcu_read_lock();
        indev = __in_dev_get_rcu(skb->dev);
        if (indev && indev->ifa_list) {
            ifa = indev->ifa_list;
            newdst = ifa->ifa_local;
        }
        rcu_read_unlock();

        if (!newdst)
            return NF_DROP;
    }
```

### NAT穿透

[NAT穿透][20]是比较常见的一个问题，在P2P中被广泛应用。在了解NAT穿透之前需要了解NAT的种类，Wikipedia上面给了很详细的[说明][15]。[STUN][21]是一种网络协议，它允许位于NAT（或多重NAT）后的客户端找出自己的公网地址，查出自己位于哪种类型的NAT之后以及NAT为某一个本地端口所绑定的Internet端端口。这些信息被用来在两个同时处于NAT路由器之后的主机之间建立UDP通信。该协议由RFC 5389定义。UPnP是由“通用即插即用论坛”推广的一套网络协议。该协议的目标是使家庭网络（数据共享、通信和娱乐）和公司网络中的各种设备能够相互无缝连接，并简化相关网络的实现。UPnP通过定义和发布基于开放、因特网通讯网协议标准的UPnP设备控制协议来实现这一目标，也是NAT穿透的标准之一。

#### IPSec中的NAT

提起NAT的源于一篇gist[朴素VPN：一个纯内核级静态隧道][22]，上面提到的东西在这里不提。值得注意的是，IPSec本身就有UDP封装的配置，也有响应的RFC规定了如何穿透NAT，但这里为什么要多此一举呢？简单地说，这在RFC有关IPSec的地方中有提及。

> 结束与08月01日18点38分，太懒了，不写了。

## 参考资料

[1]: https://www.zfl9.com/ss-redir.html "ss-redir 透明代理"
[2]: https://elixir.bootlin.com/linux/v4.9/source/net/netfilter/nf_nat_proto_tcp.c
[3]: https://elixir.bootlin.com/linux/v4.9/source/net/netfilter/nf_nat_proto_udp.c
[4]: https://elixir.bootlin.com/linux/v4.17/source/net/ipv4/netfilter/nf_conntrack_l3proto_ipv4.c#L220
[5]: https://patchwork.ozlabs.org/patch/8552/ "[TPROXY] implemented IP_RECVORIGDSTADDR socket option"
[6]: https://people.netfilter.org/hidden/nfws/nfws-2008-tproxy_slides.pdf "TProxy - Transparent proxying, again"
[7]: https://people.netfilter.org/hidden/tproxy/iptables-tproxy-200710091749.diff
[8]: https://patchwork.ozlabs.org/patch/8552/
[9]: https://gist.github.com/qsorix/4066589 "qsorix/udp_socket_addr.cc"
[10]: https://blog.csdn.net/wenqian1991/article/details/46700177 "【Linux 内核网络协议栈源码剖析】网络栈主要结构介绍（socket、sock、sk_buff，etc）"
[11]: https://elixir.bootlin.com/linux/v4.17/source/net/netfilter/xt_TPROXY.c#L350
[12]: https://elixir.bootlin.com/linux/v4.17/source/net/ipv4/udplite.c#L33
[13]: https://blog.csdn.net/dog250/article/details/13161945 "TPROXY之殇-NAT设备加代理的恶果"
[14]: https://beginlinux.wordpress.com/tag/iptables-nat/ "Understanding Network Address Translation, NAT"
[15]: https://en.wikipedia.org/wiki/Network_address_translation "Network address translation"
[16]: https://tools.ietf.org/html/rfc2663 "RFC 2663: IP Network Address Translator (NAT) Terminology and Considerations"
[17]: http://blog.chinaunix.net/uid-20786208-id-5145525.html "iptables深入解析-nat篇"
[18]: https://blog.chionlab.moe/2018/02/09/full-cone-nat-with-linux/ "从DNAT到netfilter内核子系统，浅谈Linux的Full Cone NAT实现"
[19]: https://blog.chionlab.moe/2018/03/31/full-cone-nat-with-linux-2/ "Linux 内核态实现 Full Cone NAT（2）"
[20]: https://zh.wikipedia.org/wiki/NAT%E7%A9%BF%E9%80%8F "NAT穿透"
[21]: https://zh.wikipedia.org/wiki/STUN "STUN"
[22]: https://gist.github.com/klzgrad/5661b64596d003f61980 "朴素VPN：一个纯内核级静态隧道"

- [ss-redir 透明代理][1]
- [[TPROXY] implemented IP_RECVORIGDSTADDR socket option][5]
- [TProxy - Transparent proxying, again][6]
- [qsorix/udp_socket_addr.cc][9]
- [【Linux 内核网络协议栈源码剖析】网络栈主要结构介绍（socket、sock、sk_buff，etc）][10]
- [TPROXY之殇-NAT设备加代理的恶果][13]
- [Understanding Network Address Translation, NAT][14]
- [Network address translation][15]
- [RFC 2663: IP Network Address Translator (NAT) Terminology and Considerations][16]
- [iptables深入解析-nat篇][17]
- [从DNAT到netfilter内核子系统，浅谈Linux的Full Cone NAT实现][18]
- [Linux 内核态实现 Full Cone NAT（2）][19]
- [NAT穿透][20]
- [STUN][21]
- [朴素VPN：一个纯内核级静态隧道][22]