---
title: criu restore时sendto EPERM返回值的debug记录
date: 2021-09-20 01:48:49
tags: ["debug", "linux"]
---

## Background

criu带有参数`--tcp-established`恢复进程时，报错：

```log
(02.120130)  46208: open file flags:1
(02.120136)  46208: inet:       Restore: family AF_INET    type SOCK_STREAM    proto IPPROTO_TCP      port 41810 state TCP_CLOSE_WAIT   src_addr 127.0.0.1 dst_addr 127.0.0.1
(02.120163)  46208: sockets:    Binding socket to lo dev
(02.120168)  46208: tcp: Restoring TCP connection
(02.120174)  46208: tcp: Restoring TCP connection id 2a2 ino 1c13a
(02.120190)  46208: Debug:      Setting 1 queue seq to 238346158
(02.120197)  46208: Debug:      Setting 2 queue seq to 3717721167
(02.120210)  46208: Debug:      Restoring TCP options
(02.120215)  46208: Debug:              Will turn SAK on
(02.120221)  46208: Debug:              Will set snd_wscale to 7
(02.120227)  46208: Debug:              Will set rcv_wscale to 7
(02.120233)  46208: Debug:              Will turn timestamps on
(02.120238)  46208: Debug: Will set mss clamp to 65495
(02.120245)  46208: Debug:      Restoring TCP 1 queue data 1078 bytes
(02.122296)  46208: Error (soccr/soccr.c:690): Unable to send a fin packet: libnet_write_raw_ipv4(): -1 bytes written (Operation not permitted)
(02.122329)  46208: Error (criu/files.c:1372): Unable to open fd=469 id=0x2a2
(02.555720) Error (criu/cr-restore.c:1634): 46208 killed by signal 9: Killed
```

## Debug

### Step 1: 问题发生时的场景

criu checkpoint日志显示：

```log
(00.424489) inet:       Collected: ino  0x1c13a family AF_INET    type SOCK_STREAM    port    41810 state TCP_ESTABLISHED  src_addr 127.0.0.1

(09.312339) 46208 fdinfo 469: pos:                0 flags:             4002/0x1
(09.312345) fdinfoEntry fd: 469
(09.312347) sockets: Searching for socket 0x1c13a family 2
(09.312358) sockets:    Dumping lo bound dev for sk
(09.312360) sockets: No filter for socket
(09.312361) inet: Dumping inet socket at 469
(09.312363) inet:       Dumping: ino  0x1c13a family AF_INET    type SOCK_STREAM    port    41810 state
TCP_ESTABLISHED  src_addr 127.0.0.1
(09.312366) inet:       Dumped: family AF_INET    type SOCK_STREAM    proto IPPROTO_TCP      port 41810 state 0                src_addr 127.0.0.1 dst_addr 127.0.0.1
(09.312369) tcp: Dumping TCP connection
(09.312370) tcp:        Turning repair on for socket 1c13a
(09.312372) Locked 127.0.0.1:41810 - 127.0.0.1:30018 connection
(09.312410) tcp: Done
(09.312419) write fdinfoEntry fd=469 id=674
```

对比criu restore时日志，发现该TCP socket dump时显示的状态是`TCP_ESTABLISHED`，但该TCP socket restore时的状态是`TCP_CLOSE_WAIT`。**Q1: 相同的TCP，为什么dump时与restore时显示的状态不同？**

TCP在关闭链接时有如下的状态机：

```md
# soccr/soccr.c in criu
The TCP transition diagram for half closed connections
------------
FIN_WAIT1    \ FIN
                     ---------
             / ACK   CLOSE_WAIT
-----------
FIN_WAIT2
                     ----------
             / FIN   LAST_ACK
-----------
TIME_WAIT    \ ACK
                     ----------
                     CLOSED
```

criu对TCP socket dump调用路径如下：

```c
# Collect TCP info
collect_sockets
- inet_receive_one
  - inet_collect_one
    - d->state = m->idiag_state # TCP status
    - show_one_inet("Collected", d); # print `Collected` info

# record TCP info
do_dump_one_inet_fd
- lookup_socket_ino
  - gen_uncon_sk
    - getsockopt(lfd, SOL_TCP, TCP_INFO, &info, &aux) # get socket info
    - sk->state = TCP_CLOSE # TCP status
- show_one_inet("Dumping", sk); # print `Dumping` info
- show_one_inet_img("Dumped", &ie); # print `Dumped` info
  - dump_one_tcp
    - dump_tcp_conn_state
      - libsoccr_save
        - refresh_sk
          - getsockopt(sk->fd, SOL_TCP, TCP_INFO, ti, &olen) # get socket info
          - data->state = ti->tcpi_state # TCP statue
      - pb_write_one # write TCP info to img
```

可以看到，criu dump TCP时显示的TCP信息并非最终写入img的，推断出此状态发生的场景：

1. criu freeze application
2. criu get TCP socket by the controled application
3. criu detects TCP socket status, the following TCP info is from this detecting
4. TCP socket is closed by the opposite end, the TCP socket status helded by criu is changed to `TCP_CLOSE_WAIT`
5. criu detects TCP socket status once again
6. criu write TCP socket information to img

### Step 2: `errno`的来源

criu对`TCP_CLOSE_WAIT`的socket，会使用libnet构造TCP finish ack包来关闭链接。libnet使用的是raw socket，通过系统调用`sendto`发送，而`sendto`报错`EPERM`（即打印文字*Operation not permitted*），并不属于man page[[1]]中所描述的返回值之一，搜索发现当netfilter中的链接追踪模块conntrack连接数满时会返回`EPERM`[[2]]，内核调用路径如下：

```c
-> nf_hook_slow
   -> ipv4_conntrack_local
      -> nf_conntrack_in
         -> resolve_normal_ct
            -> init_conntrack
               -> __nf_conntrack_alloc
         <------- return -ENOMEM, and print "nf_conntrack: table full, dropping packet\n"
   <---- return NF_DROP
<- return -EPERM
```

证实的确netfilter是`sendto`返回`-EPERM`的原因，但问题复现时，环境中的连接追踪数量（即 */proc/sys/net/netfilter/nf_conntrack_count* 值）未见明显异常，同时`dmesg`也无 *nf_conntrack: table full, dropping packet* 相关打印。遂排除了conntrack模块的影响。

### Step 3: 使用function graph tracer + kprobe event观测

得知问题出现的环境运行了Kubernetes，而Kubernetes使用了若干条iptables规则的，初步怀疑Kubernetes的netfilter规则影响了criu设置的改进后的netfilter规则，于是使用function graph观测criu执行时有关`raw_sendmsg`在内核中的调用路径，相关命令如下：

```bash
## 在A bash中执行以下命令
# echo 1 > /sys/kernel/debug/tracing/options/function-fork # 开启ftrace追踪子进程的功能
# echo $$ > /sys/kernel/debug/tracing/set_ftrace_pid # 设置当前的bash进程pid为ftrace追踪的pid
# echo raw_sendmsg > /sys/kernel/debug/tracing/set_graph_function # 设置要追踪调用路径的函数
# echo function_graph > /sys/kernel/debug/tracing/current_tracer # 设置tracer
# criu restore ...
```

```log
  96)               |  raw_sendmsg() {
...
  96)               |    ip_route_output_flow() {
...
  96)               |    nf_hook_slow() {
...
  96)               |      nft_do_chain_inet [nf_tables]() {
  96)               |        nft_do_chain [nf_tables]() {
  96)   1.520 us    |          nft_meta_get_eval [nf_tables]();
  96)   1.280 us    |          nft_cmp_eval [nf_tables]();
  96)   0.850 us    |          nft_meta_get_eval [nf_tables]();
  96)   0.860 us    |          nft_cmp_eval [nf_tables]();
  96) + 11.780 us   |        }
  96) + 14.630 us   |      }
  96)               |      iptable_filter_hook [iptable_filter]() {
  96)               |        ipt_do_table [ip_tables]() {
...
  96)   0.830 us    |          comment_mt [xt_comment]();
  96)   0.930 us    |          mark_mt [xt_mark]();
  96) ! 172.990 us  |      }
  96)               |      kfree_skb() {
...
  96) ! 311.190 us  |    }
  96)   3.630 us    |    dst_release();
  96) ! 552.410 us  |  }
```

追踪到的日志如下，`nft_do_chain_inet`的调用流程证明criu改进后的netfilter规则执行正常，可奇怪的是`mark_xt`执行得似乎有点问题。于是设置kprobe event来打印`xt_mark`的执行结果：

```bash
# echo 1 > /sys/kernel/debug/tracing/options/event-fork
# echo $$ > /sys/kernel/debug/tracing/set_event_pid
# echo 'p:r_mark_mt mark_mt retval=$retval' > /sys/kernel/debug/tracing/kprobe_events
# echo 1 > /sys/kernel/debug/tracing/events/kprobe/enable
```

```log
  96)               |      iptable_filter_hook [iptable_filter]() {
  96)               |        ipt_do_table [ip_tables]() {
...
  96)   0.830 us    |          comment_mt [xt_comment]();
  96)   0.930 us    |          mark_mt [xt_mark]();
  96)               |          trampoline_probe_handler() {
  96)   0.910 us    |            kretprobe_hash_lock();
  96)               |            /* r_mark_mt: (ipt_do_table+0x358/0x6f0 [ip_tables] <- mark_mt) retval=0x1 */
...
```

得到的结果如上，确认是mark规则影响了。根据function graph打印出来的调用关系，在iptables规则中的OUTPUT hook上的filter chain中看到：

```md
-A OUTPUT -j KUBE-FIREWALL
-A KUBE-FIREWALL -m comment --comment "kubernetes firewall for dropping marked packets" -m mark --mark 0x8000/0x8000 -j DROP
```

符合OUTPUT hook + filter chain、comment match + mark match、drop target，怀疑上述的规则是导致`sendto`的罪魁祸首，遂删除该规则重新执行criu，问题不再发生。

### Reason

阅读代码得知，criu设置的mark `0xC114`与掩码`0x8000`执行`&`预算，结果是`0x8000`，匹配上了Kubernetes设置的netfilter规则，遂丢弃了该包。

[1]: https://man7.org/linux/man-pages/man3/sendto.3p.html
[2]: https://blog.cloudflare.com/conntrack-tales-one-thousand-and-one-flows/
