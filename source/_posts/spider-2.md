---
title: 之一：从 url 谈起
date: 2016-04-15 11:10:18
categories: style
tags: spider
---

这里记录一些必要的 HTTP 知识。不过似乎从上句话开始就离题了。

## HTTP 协议简介

HTTP 是超文本传输协议（HyperText Transfer Protocol）的缩写。它是基于 TCP 的连接方式。  

HTTP url 在下面记录。
HTTP 请求由三部分组成：请求行，请求报头，请求正文。
在浏览器中，请求行与请求报头统称为 Request Headers。请求正文是可选的，使用时，正文与请求头直接用空行隔开。

* 请求行
  * 请求行以一个方法符号开头，以空格分开，后面跟着请求的 URI 和协议的版本
  * 格式： `Method  Request-URI  HTTP-Version  CRLF`  
    * *Method* 表示请求方法
    * *Request-URI* 是一个统一资源标识符
    * *HTTP-Version* 表示请求的HTTP协议版本
    * *CRLF* 表示回车和换行（除了作为结尾的 *CRLF* 外，不允许出现单独的 *CR* 或 *LF* 字符）
* 请求报头在下面记录
* 请求正文，一般是表单数据（Form Data），也在下面记录。

HTTP 响应也是由三个部分组成：状态行、消息报头、响应正文。
Response Headers 是浏览器中对请求行与请求报头的统称。

* 状态行
  * 格式：`HTTP-Version  Status-Code  Reason-Phrase  CRLF`
  * *Status-Code* 表示服务器发回的响应状态代码
  * *Reason-Phrase* 表示状态代码的文本描述
* 消息报头在下面记录
* 相应正文就是请求的数据啦

这里 [HTTP协议简介] 算是一个不错的使用浏览器开发者工具的教程。

## HTTP 统一资源定位符

url 是 Uniform Resource Locator（统一资源定位符）的缩写。一个完整的 url 形式是这样子的：

```url
scheme:[//[user:password@]host[:port]][/]path[?query][#fragment]
```

* *scheme* 之意为协议
  * 流行的协议有 *http*, *ftp*, *mailto*, *file*, *data*
  * 在`:`符号之前
* *//*
  * 部分协议需要的东西
  * 跟在`:`之后。
* *user:password@host:port* 突然意识到，这东西与 *ssh* 命令很像嘛
  * *user:password* 使用格式如其意，位于`@`符号之前
  * *host* 为域名，由 DNS 服务器解析到相应的 ip 地址
  * *port* 为端口号。`http`协议默认80，`https`默认`443`
* *path* 就是文件位置啦
  * `/` 代表根地址
  * 使用 *nix 系统的相比对这不陌生
* *query* 也就是 query string parameters（查询字符参数）
  * `?`符号把 *path* 与 *query* 分开
  * 以键值对方式链接：`key=value`
  * 多个查询参数由`;`或`&`分开
* *fragment*
  * 页内锚点，写过 HTML 应该会熟悉
  * 跟在`#`后面

```url
# 一些栗子
http://www.example.com/example.html # 不带查询参数的 url

http://www.example.com/example.html?id=1 # 查询参数为 id，值为1

https://user:passwd@api.delicious.com/v1/posts/get?tag=webdev&meta=yes # 带有验证信息的 url，用户名为 user，密码为 passwd

http://www.example.com/example.html#id # 页内超链接，跳至该命名为 id 锚
```

曾经捣鼓过 Nginx，从 Nginx 的配置文件看下`host[:port][/]path`

```nginx
server {
    listen    1080;
    server_name vvl.me;

    location / {
        root    /var/www/html;
        index   index.html;
    }
}
```

协议没指定，Nginx 默认的是 http 协议，访问端口是1080， 默认页面为 index.html。也就是说，浏览器中输入`http://vvl.me:1080`实际上访问的是`http://vvl.me:1080/index.html`，这个资源在服务器上的绝对路径为`/var/www/html/index.html`。

## HTTP 方法

HTTP 方法（也经常被叫做“谓词”）告知服务器，客户端想对请求的页面做些什么。下面的都是非常常见的方法：

* GET
  * 浏览器告知服务器：只**获取**页面上的信息并发给我。这是最常用的方法。
* HEAD
  * 浏览器告诉服务器：欲获取信息，但是只关心**消息头** 。应用应像处理 GET 请求一样来处理它，但是不分发实际内容。
* POST
  * 浏览器告诉服务器：想在 URL 上**发布**新信息。并且，服务器必须确保数据已存储且仅存储一次。这是 HTML 表单通常发送数据到服务器的方法。
* PUT
  * 类似 POST 但是服务器可能触发了存储过程多次，多次覆盖掉旧值。你可能会问这有什么用，当然这是有原因的。考虑到传输中连接可能会丢失，在 这种 情况下浏览器和服务器之间的系统可能安全地第二次接收请求，而不破坏其它东西。因为 POST 它只触发一次，所以用 POST 是不可能的。
* DELETE
  * **删除**给定位置的信息。
* OPTIONS
  * 给客户端提供一个敏捷的途径来弄清这个 URL 支持哪些 HTTP 方法。

## HTTP 消息头

除了 HOST 为必须，其他都是可选的。下面写几个常用的：

* HOST -- 当一个 Web Server 部署了多个域名时，就靠它了
* UA（User-Agent）-- 一篇有趣的故事：[浏览器野史 UserAgent 列传 上] & [浏览器野史 UserAgent 列传 下]
* Cookies -- 为了辨别用户身份而储存在用户本地终端上的数据 - 模拟登录用到的东西 -
* Referer -- 表示从哪儿链接到目前的网页，采用的格式是URL
* Accept-Encoding -- 能够接受的编码方式列表

延伸一下`Accept-Encoding`，我也是捣鼓过的人 =_=。它通常默认为`gzip, deflate`，`gzip`是一种压缩格式，服务器使用压缩后可以用同样大小的数据传输更多的信息。但是 Python 中`urllib`可不支持gzip压缩哦。Nginx 中相关配置如下：

```nginx
    gzip  on;
    gzip_min_length         1024;
    gzip_buffers            40      4k;
    gzip_comp_level                 2;
    gzip_disable            msie6;
    gzip_types              application/atom+xml application/javascript application/json application/rss+xml application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/svg+xml image/x-icon text/css text/plain text/x-component;
```

逃之~

## 表单

在爬虫中，提交表单通常意味着模拟登录。提交的表单数据是以键值对形式组合的，并按照某种编码方式组合起来，存储在 HTTP 的请求正文中。

表单的数据格式与 url 中的 query很相似。不同的是表单数据是存储于请求正文中，而 query 数据在 消息报头中（确切的说是因为使用了 POST 方法，而不是 GET 方法）。请求正文中的数据大小没有限制，而 query 则相反。

这是一个表单：

```html
<form action="" method="post" class="form" role="form">
  <input type='hidden' name='csrfmiddlewaretoken' value='5cWBAVpqzJVxM74Wk4yWlWh0u0Dpz1JF' />
  <div class="form-group">
    <label for="id_username">昵称:</label>
    <input class="form-control" id="id_username" maxlength="40" name="username" size="40" type="text" />
  </div>
  <div class="form-group">
    <label for="id_password">密码:</label>
    <input class="form-control" id="id_password" maxlength="40" name="password" size="40" type="password" />
  </div>
  <button id='id_submit' type="submit" class="btn btn-default">登录</button>
  <button id='id_register' type='button' class="btn btn-default" onclick='javascrtpt:window.location.href="/accounts/register"'>注册</button>
</form>
```

发送的表单数据是这样的：

```html
# 输入的用户名是 21，密码是 1212
csrfmiddlewaretoken=5cWBAVpqzJVxM74Wk4yWlWh0u0Dpz1JF&username=21&password=1212

# 为了便于观察，写成键值对形式
csrfmiddlewaretoken:5cWBAVpqzJVxM74Wk4yWlWh0u0Dpz1JF
username:21
password:1212
```

csrf 是用来防御跨站域请求攻击的。form 中的数据是以`name=value`形式组合后存储在请求正文中发送的。

## HTTP 响应状态

是对 Status-Code（响应状态代码）与 Reason-Phrase（状态代码的文本描述）的记录。

状态代码有三位数字组成，第一个数字定义了响应的类别，且有五种可能取值：

* 1xx：指示信息--表示请求已接收，继续处理
* 2xx：成功--表示请求已被成功接收、理解、接受
* 3xx：重定向--要完成请求必须进行更进一步的操作
* 4xx：客户端错误--请求有语法错误或请求无法实现
* 5xx：服务器端错误--服务器未能实现合法的请求

常见状态代码、状态描述与说明：

* 200 OK
  * 请求已成功，请求所希望的响应头或数据体将随此响应返回
* 301 Moved Permanently
  * 永久重定向
* 302 Found
  * 临时重定向
* 400 Bad Request
  * 客户端请求有语法错误，不能被服务器所理解
* 401 Unauthorized
  * 请求未经授权
* 403 Forbidden
  * 服务器收到请求，但是拒绝提供服务
* 404 Not Found
  * 请求资源不存在
* 500 Internal Server Error
  * 服务器发生不可预期的错误
* 503 Server Unavailable
  * 由于临时的服务器维护或者过载，服务器当前无法处理请求。这个状况是临时的，并且将在一段时间以后恢复

## 总结

没啥好总结的唉，HTTP 知识浅薄，错了的话就提出吧，感谢！

## 参考资料

[Uniform Resource Locator](https://en.wikipedia.org/wiki/Uniform_Resource_Locator)  
[HTTP 方法](http://docs.jinkan.org/docs/flask/quickstart.html#http)  
[Http协议详解](http://www.jianshu.com/p/e83d323c6bcc)  
[Hypertext Transfer Protocol](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol)  
[HTTP协议简介](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001432011939547478fd5482deb47b08716557cc99764e0000)  
[浏览器野史 UserAgent 列传 上](http://litten.github.io/2014/09/26/history-of-browser-useragent/)  
[浏览器野史 UserAgent 列传 下](http://litten.github.io/2014/10/05/history-of-browser-useragent2/)  
[Cookie](https://zh.wikipedia.org/wiki/Cookie)  
[HTTP来源地址](https://zh.wikipedia.org/wiki/HTTP%E5%8F%83%E7%85%A7%E4%BD%8D%E5%9D%80)  
[细说 Form (表单)](http://www.cnblogs.com/fish-li/archive/2011/07/17/2108884.html)  
[HTTP状态码](https://zh.wikipedia.org/wiki/HTTP%E7%8A%B6%E6%80%81%E7%A0%81)  

[浏览器野史 UserAgent 列传 上]: http://litten.github.io/2014/09/26/history-of-browser-useragent/
[浏览器野史 UserAgent 列传 下]: http://litten.github.io/2014/10/05/history-of-browser-useragent2/
[HTTP协议简介]: http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001432011939547478fd5482deb47b08716557cc99764e0000
