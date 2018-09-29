---
layout: post
title: HTTP KeepAlive && TCP KeepAlive原理与对比
date: 2017-09-28 23:30:00
tags: HTTP TCP keepalive
categories: TCP/IP
author: PythonPig
---
* content
{:toc}

##### \#0x00 写在前面
前面写了点关于TCP KeepAlive的文章[在vmware nat模式下使用tcp KeepAlive的坑（Vmware nat的实现方式）](https://pythonpig.github.io/2017/09/26/vmware-nat-&-tcp-keepalive/)，今天聊聊HTTP KeepAlive。




##### \#0x01 HTTP KeepAlive
HTTP协议是一种无状态的“请求-应答”协议，传输层基于TCP协议，在HTTP 1.0版本时，默认不使用长连接，即：客户端与服务端每一次“请求-应答”都需要新建一个TCP连接，传输完数据后断开连接，过程如下图所示，若要保持长连接，需要在请求头中加上Connection: keep-alive，而HTTP/1.1默认是支持长连接的。

![](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/HTTP%20KeepAlive%20TCP%20KeepAlive/%E9%9D%9Ekeepalive%E4%B8%A4%E6%AC%A1%E8%AF%B7%E6%B1%82.jpeg?raw=true)

这种一次HTTP“请求-应答”建立一个TCP连接的方式显然会带来一些问题，比如HTTP会频繁的“新建-断开”TCP连接，影响服务器性能（更少的tcp连接意味着更少的系统内核调用,socket的accept()和close()调用），另外，主动发起TCP关闭请求（主动发起FIN）的一方可能是客户端也可能是服务端，如果服务端大量发起TCP连接关闭请求，服务器会产生大量的TIME_WAIT状态的连接，占用服务器资源。因此，HTTP从1.1版本之后默认打开了KeepAlive来解决上面的问题，**HTTP KeepAlive的作用就是在一个TCP连接上传输多个HTTP数据包**。过程如下图所示。

![](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/HTTP%20KeepAlive%20TCP%20KeepAlive/keepalive%E4%B8%A4%E6%AC%A1%E8%AF%B7%E6%B1%82.jpeg?raw=true)

从上图中可以看出TCP建立（三次握手）之后，完成了两次HTTP“请求-应答”后关闭了TCP连接（四次挥手）。

凡事有利就有弊，HTTP虽然可以减少TCP建立次数，但是也带来了如下问题：

##### 1、客户端/服务端协商及支持情况
前面讲到客户端和服务端都可以关闭一条TCP连接，因此如何使客户端和服务端都不关闭一条需要KeepAlive的连接呢？HTTP协议在协议头中提供了特定的字段来解决这个问题，这个字段就是"Connection"，如果Connection的值是“Keep-Alive”时，就表示这条连接是需要KeepAlive的。但并不是消息头中包含"Connection: Keep-Alive"字段的HTTP连接就一定可以被KeepAlive，前提条件是客户端和服务端都支持KeepAlive，如下图所示，虽然HTTP消息中包含"Connection: Keep-Alive"字段，但由于客户端不支持，在完成HTTP“请求-应答”后，客户端发起了关闭TCP连接的请求（FIN包）。

![](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/HTTP%20KeepAlive%20TCP%20KeepAlive/%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%B8%8D%E6%94%AF%E6%8C%81%EF%BC%8Ckeepalive%E5%A4%B4%E4%B8%8D%E8%B5%B7%E4%BD%9C%E7%94%A8.jpeg?raw=true)

##### 2、KeepAlive占用TCP连接资源问题
当HTTP连接都使用KeepAlive之后，HTTP服务器会产生大量的TCP连接从而占用服务器资源，别有用心的人甚至可以利用这个特性进行DOS攻击。为了解决这个问题，HTTP服务器都有一个KeepAlive timeout的参数，该参数表示：从客户端/服务端最后一次通信开始，经过timeout秒之后，服务端主动关闭该TCP连接。Apache/2.4.23 (Win32) 中KeepAlivetimeout的值默认为5秒，5秒过后服务端主动关闭TCP连接，如下图所示。

![](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/HTTP%20KeepAlive%20TCP%20KeepAlive/%E6%9C%8D%E5%8A%A1%E7%AB%AFkeepalive%E8%B6%85%E6%97%B6%E5%85%B3%E9%97%AD%E8%BF%9E%E6%8E%A5-5s.jpeg?raw=true)

##### 3、如何知道HTTP数据传送完毕
在不使用长连接时，每接受一次数据TCP连接都会断开（客户端读数据时会返回EOF（-1）），因此客户端很容易知道判断数据是否接收完毕，但使用长连接时判断数据是否接收完毕就出现了困难，现在有两种解决办法：
1）Content-Length：xxx
如果客户端请求的是静态资源（服务器在response时已经知道资源的大小），则在HTTP response的头部携带Content-Length字段，表明本次回复数据的长度，客户端解析这个字段就可以判断数据是否传输完毕。
2）Transfer-Encoding: chunked
如果用户请求的资源是动态生成的（服务器在回复第一个包时并不知道资源的大小），则在HTTP response的头部携带Transfer-Encoding: chunked，表明数据是分块传输的。以分块传输一段文本内容：“人的一生总是在追求自由的一生 So easy”来说明分块传输的过程，如下图所示。

![](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/HTTP%20KeepAlive%20TCP%20KeepAlive/%E5%88%86%E5%9D%97%E4%BC%A0%E8%BE%93.png?raw=true)

图中每个分块的第一行是分块内容的大小，十六进制表示，后面跟CRLF(\r\n)，第一行本身以及分块内容末尾的CRLF不计入大小。第二行是分块内容，后面也跟CRLF。最后一个分块虽然大小为零，但是必不可少，表示分块的结束，后面也跟CRLF，同时内容为空。最后，响应体以CRLF结束。

##### \#0x02 HTTP KeepAlive与TCP KeepAlive

TCP KeepAlive是探测/保活机制，用于探测对端的状态及网络情况，当然也有保活功能，如防止nat超时；HTTP KeepAlive是共用TCP连接的机制，两者的作用和实现机制都不同。

对于linux系统，TCP KeepAlive有三个相关的参数配置，都在目录/proc/sys/net/ipv4/下
'''  
tcp_keepalive_time  7200 当keepalive启用的时候，首次发送keepalive消息的间隔是2小时
tcp_keepalive_intvl  75  当探测没有确认时，重新发送探测的频度。缺省是75秒
tcp_keepalive_probes  9  探测尝试的次数。如果第1次探测包就收到响应了，则后8次的不再发
'''
HTTP KeepAlive相关的参数只有一个keepalive timeout，超过timeout时间没有数据传输后则关闭当前连接。

##### \#0x03 总结

虽然HTTP底层的传输协议是TCP，但HTTP KeepAlive与TCP KeepAlive是两个不同的概念，HTTP KeepAlive timeout的时间一般是几秒到几十秒，完全没有到触发TCP KeepAlive发送保活探测包的条件（默认传输完最后一个数据包到发送第一个保活探测包的时间间隔是7200s）。

##### \#0x04 参考
* [HTTP Keep-Alive模式](http://www.cnblogs.com/skynet/archive/2010/12/11/1903347.html)
* [HTTP keep-alive详解](https://blog.csdn.net/xiaoduanayu/article/details/78386508)
* [wireshark抓包简单查看HTTP keep-alive原理](https://blog.csdn.net/Kingson_Wu/article/details/72512825)
* [HTTP Keep-Alive 学习](http://fengzhenbing98.iteye.com/blog/2117529)
