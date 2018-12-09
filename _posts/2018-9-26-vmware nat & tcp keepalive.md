---
layout: post
title: 在vmware nat模式下使用tcp keepalive的坑（Vmware nat的实现方式）
date: 2017-09-26 23:30:00
tags: vmware nat keepalive
categories: TCP/IP
author: PythonPig
---
* content
{:toc}

### \#0x00 写在前面
最近在做项目时用到了Vmware虚拟机，主机是windows，虚拟机是ubuntu，网络模式选的是NAT，由于通信两端的网络状况非常不好，为了检测网络是否中断以便客户端及时重连，使用了TCP keepalive机制。但是发现，虚拟机发送的keepalive包并没有到达对端服务器，但是虚拟机收到了来自对端服务器的keepalive response（response的源IP为服务器端IP地址），如下图：
![](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/vmware%20nat/keepalive.png?raw=true)




写了个简单的python代码产生TCP keepalive包(ubuntu14.04 x64 python2.7)：
```python
#usage:python tcp_keepalive.py
#>"test"

#-*- coding:utf-8 -*-

import socket

HOST ='192.168.11.2'
PORT = 57212
BUFFSIZE=2048
ADDR = (HOST,PORT)

def set_keepalive_linux(sock, after_idle_sec=1, interval_sec=3, max_fails=5):
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
    sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, after_idle_sec)
    sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPINTVL, interval_sec)
    sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPCNT, max_fails)

tctimeClient = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
set_keepalive_linux(tctimeClient,5,1,50)
tctimeClient.connect(ADDR)

while True:
    data = input(">")
    if not data:
        break
    tctimeClient.send(data.encode())
    data = tctimeClient.recv(BUFFSIZE).decode()
    if not data:
        break
    print(data)
tctimeClient.close()
```

### \#0x01 tcp keepalive  
TCP虽然是面向连接的，但是TCP链路中断时两端终端并不一定立即感知到。在长连接的情况下，需要心跳保活机制感知对端是否存活。业务层面有心跳机制，TCP协议也提供了心跳保活机制，就是今天讲的TCP keepalilve机制。

TCP keepalive机制并不是标准规范，但是大多数操作系统的TCP协议栈都支持keepalive机制，该机制在操作系统中是默认关闭的，使用时需要应用层打开。

TCP keepalive包不包含任何数据，ACK置位，且满足SEG.SEQ = SND.NXT-1，即TCP保活探测报文序列号为下一个要发送包的序列号减1，也即上一个收到确认消息中的ACK-1。SND.NXT = RCV.NXT，即下一次发送正常报文序号等于ACK序列号。参考上图。

### \#0x02 vmware nat  
为了找到其中原因，在国外论坛也没有找到系统介绍vmware nat实现的资料，但是在CSDN上找到了一篇文章，感谢作者dog250（如何在VMWare的NAT模式下使用traceroute(解析vmnat的行为)），受到了启发，做了如下实验。
1、使用scapy在虚拟机中发送TCP SYN包，然后分别在宿主机和虚拟机中使用wireshark进行抓包。

使用scapy执行：
```python
sr1(IP(dst="192.168.11.2")/TCP(dport=57212,flags="S"))，
```
如下图：
![](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/vmware%20nat/scapy.png?raw=true)

宿主机抓包如下图：
![](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/vmware%20nat/%E5%AE%BF%E4%B8%BB%E6%9C%BA%E6%8A%93%E5%8C%85.jpeg?raw=true)

虚拟机抓包如下图：
![](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/vmware%20nat/%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%8A%93%E5%8C%85.jpeg?raw=true)

通过上面抓包可以发现，在虚拟机只发送TCP SYN包的情况下，宿主机已经完成了与对端服务器的TCP连接（三次握手），**因此可以得出结论：Vmware nat其实是通过代理实现的，而不是通过修改socket 5元组实现的，也就是说Vmware nat 是一个应用层的nat**。Vmware nat在windows上其实是通过vmnat.exe这个应用实现的，其他系统类似。

### \#0x03 TCP keepalive遇到Vmware nat  
通过上面对Vmnat的分析其实就比较容易理解文章开头遇到的问题了，因为vmnat是一个代理(proxy)，而proxy只代理用户的数据部分（对TCP的数据部分进行解包-重新封包-转发），而TCP keepalive发送的是没有数据的ACK包，因此TCP keepalive无法穿透vmnat；另一方面，vmnat作为proxy分别与对端服务端和虚拟机建立了TCP连接，因此，vmnat会以服务端的身份回复虚拟机keepalive response！

### \#0x04 参考  
[RFC1122(TCP keepalive)](https://tools.ietf.org/html/rfc1122#section-4.2.3.6)  
[VMWare的NAT模式下使用traceroute(解析vmnat的行为)](https://blog.csdn.net/dog250/article/details/52234859)
