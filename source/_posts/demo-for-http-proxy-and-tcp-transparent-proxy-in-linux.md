---
title: 一个HTTP代理与Linux下TCP透明代理的演示
date: 2020-10-18 14:10:08
tags: ['network', 'linux']
categories: style
---

## Theory

### HTTP Proxy

有关原理，这里[HTTP 代理原理及实现（一）][1]、[HTTP 代理原理及实现（二）][2]描述得很清楚，总结如下：

- HTTP代理分为两类
  - 普通代理：Web代理服务器扮演中间人角色，对于连接到它的客户端它是服务端，对于要连接的服务端它是客户端
  - 隧道代理：通过Web代理服务器用隧道方式传输基于TCP的协议，即：客户端使用HTTP CONNECT方法告知Web代理服务器的目标地址与TCP端口，随后Web代理服务器在与目标完成TCP三次握手后，返回HTTP给客户端连接就绪报文，此时便建立了原始数据的任意、双向通信，直到连接关闭为止（Web代理服务器转发TCP数据流）

### TCP transparent proxy in Linux

Linux的iptables / nftables （即netifilter）提供了REDIRECT extension，允许在PREROUTING，或者OUTPUT chain将任意端口流量重定向至指定端口，比如将80端口的TCP数据流重定向到1080端口：

```bash
# iptables
iptables -t nat -A PREROUTING -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 1080
# nftables
nft add rule ip nat PREROUTING tcp dport 80 counter redirect to :1080
```

Note: `iptables-translate`可将`iptables`语法翻译成`nftables`语法

同时可以使用`getsockopt()`获取数据流的原目标端口与地址：

```c
// IPv4
struct sockaddr_in {
    __kernel_sa_family_t	sin_family;	/* Address family		*/
    __be16		sin_port;	/* Port number			*/
    struct in_addr	sin_addr;	/* Internet address		*/

    /* Pad to size of `struct sockaddr'. */
    unsigned char		__pad[__SOCK_SIZE__ - sizeof(short int) -
		      	sizeof(unsigned short int) - sizeof(struct in_addr)];
};

getsockopt(<fd>, SOL_IP, SO_ORIGINAL_DST,
            struct sockaddr_in, sizeof(struct sockaddr_in))
// IPv6
struct sockaddr_in6 {
	  unsigned short int	sin6_family;    /* AF_INET6 */
	  __be16			sin6_port;      /* Transport layer port # */
	  __be32			sin6_flowinfo;  /* IPv6 flow information */
	  struct in6_addr		sin6_addr;      /* IPv6 address */
	  __u32			sin6_scope_id;  /* scope id (new in RFC2553) */
};

getsockopt(fd, SOL_IPV6, IP6T_SO_ORIGINAL_DST,
            struct sockaddr_in6, sizeof(struct sockaddr_in6))
```

## Demonstration

一个go代码的演示，使用iptables / nftables的REDIRECT模块捕获TCP 80端口的流量，同时使用HTTP CONNECT方法对其进行代理访问。

### Prepare

[squid](http://www.squid-cache.org/)能够扮演Web代理服务器角色，默认情况下它仅允许发起443端口的CONNECT请求，可以通过注释`http_access deny CONNECT !SSL_ports`允许建立任意端口的隧道代理。

使用`$ curl --proxy http://<host>:<port> <destination>`测试HTTP代理的可用性。

HTTP [CONNECT方法的报文首部[3]如下：

```md
CONNECT <host>:<port> HTTP/1.1\r\n
Host: <host>:<port>\r\n
Proxy-Authorization: <type> <credentials>\r\n
\r\n
```

其中[`Proxy-Authorization`][4]是可选项，用于认证。`basic`类型的`<credentials>`为`<username>:<password>`的base64类型编码。

squid设置认证的配置如下：

```conf
auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/passwords
auth_param basic realm proxy
acl authenticated proxy_auth REQUIRED
http_access allow authenticated
```

Note：

- `basic_ncsa_auth`位置由具体Linux发行版确定，可能位于`/usr/lib/squid/basic_ncsa_auth`下
- `/etc/squid/passwords`含有认证信息，由包名apache2-utils中的`htpasswd`程序创建

### Impelmentation

#### Key

核心技术要点有两处：

1. 获取原始请求的目标地址与端口
2. 构造HTTP CONNECT报文

代码如下：

```go
// get original destination:port
addr, err := syscall.GetsockoptIPv6Mreq(int(file.Fd()), syscall.IPPROTO_IP, SO_ORIGINAL_DST)

// http CONNECT method
httpHeader := fmt.Sprintf(
	"CONNECT %s HTTP/1.1\r\n"+
		"Host: %s\r\n"+
		"%s"+
		"Proxy-Connection: Keep-Alive\r\n"+
		"\r\n",
  origRemote, origRemote, auth)
```

Note: `"Proxy-Connection: Keep-Alive\r\n"`非必须

详细代码在[这里](https://gist.github.com/time-river/210c730a66f5bf62b1fcc3cfc163335c)

#### 踩坑记录

##### one

golang的[`Transport`](https://github.com/golang/go/blob/606d4a38b9ae76df30cc1bcaeee79923a5792e59/src/net/http/transport.go)类型并不好用，文档很少，俩issues([net/http: document how to create client CONNECT requests](https://github.com/golang/go/issues/22554), [net/http: support bidirectional stream for CONNECT method](https://github.com/golang/go/issues/17227))提到了如何使用。
截止2020.10.18，它不支持`Host`字段的格式为`<dest addr>:<port>`，前缀必须为`//` / `http://` / `https://`（基于部分`net/http`源码分析得到的结论），也就是说：__无法使用golang提供的`Transport`建立任意基于HTTP隧道模式的TCP隧道__。

##### two

无法利用以下代码获取正确的端口：

```go
var origAddr net.TCPAddr

addr, err := syscall.GetsockoptIPv6Mreq(int(leftFile.Fd()), syscall.IPPROTO_IP, SO_ORIGINAL_DST)
if err != nil {
		fmt.Println("syscall.GetsockoptIPv6Mreq error: %w", err)
		return origAddr, err
}
origConn := (*syscall.RawSockaddrInet4)(unsafe.Pointer(addr))

origAddr.IP = net.IPv4(origConn.Addr[0], origConn.Addr[1], origConn.Addr[2], origConn.Addr[3])
origAddr.Port = int(origConn.Port)
```

因端口类型为2字节，而int为4字节，在做转换时会出错：

- 对于`80 (0x0050)`端口，会得到`20480 (0x5000)`
- 对于`443 (0x01bb)`端口，会得到`47873 (0xbb01)`

##### three

golang的`net/http/response`的`Response.Header.Write()`不会带上HTTP首部的：

- 第一行的[Status-Line: `HTTP-Version SP Status-Code SP Reason-Phrase CRLF`](https://www.w3.org/Protocols/rfc2616/rfc2616-sec6.html)
- 最后一行的`CRLF`

导致curl在测试时一直等待http报文响应。

## Reference

- [HTTP 代理原理及实现（一）][1]
- [HTTP 代理原理及实现（二）][2]
- [MDN web docs: CONNECT][3]
- [MDN web docs: Proxy-Authorization][4]
- [RFC2616: sec6 Response][5]

[1]: https://imququ.com/post/web-proxy.html
[2]: https://imququ.com/post/web-proxy-2.html
[3]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/CONNECT
[4]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/oxy-Authorization
[5]: https://www.w3.org/Protocols/rfc2616/rfc2616-sec6.html
