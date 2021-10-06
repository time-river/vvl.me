---
title: 初识 iptables
date: 2016-08-02 15:42:16
categories: style
tags: iptables
---

初识 iptables，[用iptables搭建一套强大的安全防护盾](http://www.imooc.com/learn/389)课程总结。

## Netfilter & Hook point

* Netfilter
  * Netfilter 是 Linux 操作系统核心层内部的一个数据包处理模块。
* Hook point
  * 数据包在 Netfilter 中的挂载点。
  * 五个挂载点: PRE_ROUTING / INPUT / OUTPUT / FORWARD / POST_ROUTING

Netfilter 与 iptables

```md
  +----------+        +--------------+
  | 系统调用 |        | iptables命令 |  用户层
  +----------+        +--------------+
       |     setsockopt |    |
-------v----------------v----^----------------
       |                |    | getsockopt
  +---------------------------------+
  |          内核接口               | 内核层
  +----^-------------------^--------+
       |                   |
  +----v-----+------+------v---------+
  | TCP UDP  | hook |iptables内核模块|
  +----^-----+------+----------------+
       |
  +----v-------+
  | 网络接口层 |
  +----^-------+
       |
-------+--------------------------------------
       |
  +----v-----+
  | 设备驱动 |
  +----------+
```

## iptables

* iptables 规则组成
  * 四张表 + 五条链(Hook point) + 规则
  * 表: filter / nat / mangle / raw
    * Filter: 访问控制、规则匹配
    * Nat: 地址转发
  * 链: PREROUTING / INPUT / OUTPUT / FORWARD / POSTROUTING
  * 规则
    * 数据包访问控制: ACCEPT / DROP / REJECT
    * 数据包修改: SNAT / DNAT
    * 信息记录: LOG

数据包在规则表、链匹配流程

```md
    | IN |                       /   \
     \  /                       | OUT |
 +----++----+                +-----------+
 |PREROUTING|                |POSTROUTING|
 |----------|                |-----------|
 |   nat    |                | nat       |
 |  mangle  |                | raw       |
 |   raw    |                | mangle    |
 +----------+                +--+---+----+
      |           +-------+     |   |
 +----+------+    |FORWARD|     |   |
 |Destination| no |-------|------+  |
 |=localhost?|----+ filter|    +-+--+-+
 +-----------+    | mangle|    |OUTPUT|
    |yes|         +-------+    |------|
     \ /                       |filter|
 +----+------+                 |nat   |
 |   INPUT   |    +---------+  |mangle|
 |-----------|----+localhost|--+raw   |
 | filter    |    +---------+  +------+
 | mangle    |
 +-----------+
```

一些常用的参数

```md
         | table     | command      | chain       | Paramater & Xmatch | target
-----------------------------------------------------------------------------
iptables | -t filter | -A(--append) | INPUT       | -p(--protocol) tcp | -j(--jump) ACCEPT
         |    nat    | -D(--delete) | FORWARD     | -s(--source)       |            DROP
         |           | -L(--list)   | OUTPUT      | -d(--destination)  |            REJECT
         | 默认值    | -F(--flush)  | PREROUTING  | --sport            |            DNAT
         |    filter | -P(--policy) | POSTROUTING | --dport            |            SNAT
         |           | -I(--insert) |             | --dports           |
         |           | -R(--replace)|             | -m(--match)tcp     |
         |           | -n(--numeric)|             |    state           |
         |           |              |             |    multiport       |
```

## 几个关于 filter 表的场景

### 场景一

* 规则1: 对所有的地址开放本机 tcp 协议的 80, 22, 10-21端口的访问
* 规则2: 允许对所有的地址开放本机的基于 ICMP 协议的数据包访问
* 规则3: 其他未被允许的端口则禁止访问

```bash
#iptables -I INPUT -p tcp --dport 80 -j ACCEPT
#iptables -I INPUT -p tcp --dport 22 -j ACCEPT
#iptables -I INPUT -p tcp --dport 10:21 -j ACCEPT
#iptables -I INPUT -p icmp -j ACCEPT
#iptables -A INPUT -j REJECT
```

* 存在两个问题
  * 不能访问环回地址
  * 无法进行外部访问

解决方法如下

```bash
#iptables -I INPUT -i lo -j ACCEPT
#iptables -I INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

[iptables: difference between NEW, ESTABLISHED and RELATED packets](http://serverfault.com/questions/371316/iptables-difference-between-new-established-and-related-packets)
> Consider a NEW packet a telephone call before the receiver has picked up. An ESTABLISHED packet is their, "Hello." And a RELATED packet would be if you were calling to tell them about an e-mail you were about to send them. (The e-mail being RELATED.)
> In case my analogy isn't so great, I personlly think the man pages handles it well:
>> NEW -- meaning that the packet has started a new connection, or otherwise associated with a connection which has not seen packets in both directions, and
>> ESTABLISHED -- meaning that the packet is associated with a connection which has seen packets in both directions,
>> RELATED -- meaning that the packet is starting a new connection, but is associated with an existing connection, such as an FTP data transfer, or an ICMP error.

* 附加规则: 允许指定 ip 访问

```bash
#iptables -I INPUT -p tcp -s 10.10.188.233 --dport 80 -j ACCEPT
```

### 场景二

* [ftp](https://zh.wikipedia.org/wiki/%E6%96%87%E4%BB%B6%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE) 主动模式下 iptables 的规则配置
* [ftp](https://zh.wikipedia.org/wiki/%E6%96%87%E4%BB%B6%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE) 被动模式下 iptables 的规则配置

![ftp active mode](/images/16/08/ftp-active-mode.png)
![ftp passive mode](/images/16/08/ftp-passive-mode.png)

ftp 主动模式 + ssh port + ICMP protocol 配置

```bash
#iptables -I INPUT -p tcp --dport 21 -j ACCEPT
#iptables -I INPUT -p tcp --dport 22 -j ACCEPT
#iptables -I INPUT -P icmp -j ACCEPT
#iptabels -I INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
#iptabels -A INPUT -j REJECT
```

思考: 为什么不需要打开 22 端口？下一篇解答。

ftp 被动模式

* 方法一 -- 需监听高端口
  * 1 `#iptables -I INPUT -p tcp --dport 21 -j ACCEPT`
  * 2 配置 vsftp 的被动随机端口是 50000 --- 60000
    * pasv_min_port=50000
    * pasv_max_port=60000
  * 3 `#iptables -I INPUT -p tcp --dprot 50000:60000 -j ACCEPT`

* 方法二 --使用链接追踪模块
  * 内核启用 nf_conntrack_ftp 模块
    * 临时启用 `#modprobe nf_conntrack_ftp`
    * 添加内核模块启动参数 `IPTABLES_MODULES="nf_conntrack_ftp"`
  * `#iptables -I INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT`
  * `#iptables -I INPUT -p tcp --dport 21 -j ACCEPT`

### 场景三

* 要求一: 员工在公司内部(10.10.155.0/24, 10.10.188.0/24)能访问服务器上的任何服务。
* 要求二: 外网 === 拨号到 ==> VPN 服务器 ====> 内网 FTP / SAMBA / NFS / SSH
* 要求三: HTTP 服务需要允许公网访问

* 允许本地访问，外网 + 本地环回地址
* 允许已监听状态数据包通过，10.10.155.0/24 / 10.10.188.0/24
* 允许规则中允许的数据包通过，PPTP 端口 1723 / HTTP 端口 80 / ICMP 数据包
* 拒绝未被允许的数据包

```bash
#iptables -I INPUT -i lo -j ACCEPT
#iptables -I INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
#iptables -A INPUT -s 10.10.155.0/24 -j ACCEPT
#iptables -A INPUT -s 10.10.188.0/24 -j ACCEPT
#iptables -A INPUT -p tcp --dport 80 -j ACCEPT
#iptables -A INPUT -p tcp --dport 1723 -j ACCEPT
#iptables -I INPUT -p icmp -j ACCEPT
#iptables -A INPUT -j REJECT
```

## iptables nat 表规则配置

---
分类 | 功能 | 作用链
--- | --- | ---
SNAT | 源地址转换 | 出口 POSTROUTING
DNAT | 目标地址转换 | 进口 PREROUTING

SNAT 转发

* 三台机器 A(Client, 10.10.177.233) B(Gatway, eth0: 10.10.188.232 / eth1:10.10.177.232) C(server, 10.10.188.173)
* B 设置
  * 内核 `net.ipv4.ip_forward=1`
  * `#iptables -t nat -A POSTROUTING -s 10.10.177.0/24 -j SNAT --to 10.10.188.232`
* A 设置
  * 添加 10.10.177.232 为网关

DNAT 转发

* 10.10.188.232:80 <==> 10.10.177.233:80
  * `#iptables -t nat -A PREROUTING -d 10.10.188.232 -p tcp --dport 80 -j DNAT --to 10.10.177.233:80`

思考: 本地端口(不经过网卡的数据包)如何转发？依旧下一篇。

## 防止 CC 攻击

* connlimit模块
  * 作用: 限制每一个客户端 ip 的并发连接数
  * 参数: `--connlimit-above n` n 为限制并发个数
  * 例子
    * `#iptables -I INPUT -p tcp --syn --dport 80 -m connlimit --connlimit-above 100 -j REJECT`
  * 控制并发http访问
    * `#iptables -I INPUT -p tcp --dport 80 -s 10.10.163.232 -m connlimit --connlimit-above 10 -j REJECT`

* limit模块
  * 作用: 限速，控制访问
  * 例子
    * `#iptables -A INPUT -m limit --limit 3/hour`
      * `--limit-burst` 默认值为 5
    * 允许10个，超过10个分钟允许1个 icmp 数据包
      * `#iptables -A INPUT -p icmp -m limit --limit 1/m --limit-burst 10 -j ACCEPT`
      * `#iptables -A INPUT -p icmp -j DROP`

## 结语

还差个完整规则实例这段视频的笔记，并不打算记录了。可能自己还是比较适合看书吧，绝大部分视频是以二倍的速度播放的，虽然有几个视频反复播放了好多遍。老师讲得还行，但不是我期望的样子：背后的原理并没有说出来。下篇，记录些遇到的疑问与解释。
