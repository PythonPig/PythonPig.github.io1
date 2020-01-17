---
layout: post
title: Cobaltstrike SMB beacon(命名管道相关知识) 
date: 2018-1-17 23:30:00
tags: 域渗透
categories: hack 
author: PythonPig
---
* content
{:toc}


### \#0x00 写在前面 
在使用CS的SMB beacon的过程中来学习一下命名管道相关的知识，本文中的内容大部分来自网络，谢谢各位大牛，参考文章见最后的【参考】部分。  

{:refdef: style="text-align: cener;"}
![smb beacon](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Cobaltstrike%20SMB%20beacon(命名管道相关知识)%20/smb%20beacon.png?raw=true)
{: refdef}
图片来源于:https://blog.cobaltstrike.com/2013/12/06/stealthy-peer-to-peer-cc-over-smb-pipes/


### \#0x01 SMB beacon简介
SMB Beacon使用命名管道通过父级Beacon进行通讯，当两个Beacons链接后，子Beacon从父Beacon获取到任务并发送。
因为链接的Beacons使用Windows命名管道进行通信，此流量封装在SMB协议中，所以SMB Beacon相对隐蔽，绕防火墙时可能发挥奇效(系统防火墙默认是允许445的端口与外界通信的，其他端口可能会弹窗提醒，会导致远程命令行反弹shell失败)。  
这张图很好的诠释了SMB beacon的工作流程。  

![smb beacon](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Cobaltstrike%20SMB%20beacon(命名管道相关知识)%20/smb%20beacon.png?raw=true)

### \#0x02 命名管道简介
“命名管道” 又名 “命名管线”，但是通常都叫命名管道，是一种简单基于 SMB 协议的进程间通信（Internet Process Connection - IPC）机制。 在计算机编程里，命名管道可在同一台计算机的不同进程之间或在跨越一个网络的不同计算机的不同进程之间，支持可靠的、单向或双向的数据通信传输。  

和一般的管道不同，命名管道可以被不同进程以不同的方式方法调用（可以跨语言、跨平台）。只要程序知道命名管道的名字，任何进程都可以通过该名字打开管道的另一端，根据给定的权限和服务器进程通信。  

默认情况下，我们无法使用命名管道来控制计算机通信，但是微软提供了很多种 Windows API 函数，例如 ：  

用于实例化命名管道的服务器端函数是 CreateNamedPipe  
接受连接的服务器端功能是 ConnectNamedPipe.aspx)  
客户端进程通过使用 CreateFile 或 CallNamedPipe 函数连接到命名管道  

命名规范  
命名管道的命名是采用的 UNC 格式：\\Server\Pipe\[Path]Name 的。  

第一部分\\Server指定了服务器的名字，命名管道服务即在此服务器创建，其字符串部分可表示为一个小数点(表示本机)、星号(当前网络字段)、域名或是一个真正的服务；第二部分 \Pipe 与邮槽的 \Mailslot 一样是一个不可变化的硬编码字串，以指出该文件是从属于 NTFS；第三部分[Path]Name则使应用程序可以唯一定义及标识一个命名管道的名字，而且可以设置多级目录。  
 

### \#0x03 参考
* [Cobaltstrike系列教程(三)beacon详解](https://blog.csdn.net/qq_26091745/article/details/98103437)
* [【知识回顾】命名管道](https://rcoil.me/2019/11/%E3%80%90%E7%9F%A5%E8%AF%86%E5%9B%9E%E9%A1%BE%E3%80%91%E5%91%BD%E5%90%8D%E7%AE%A1%E9%81%93/)
* [Windows 命名管道研究初探](https://www.anquanke.com/post/id/190207#h2-0)
* [【知识回顾】深入了解 PsExec](https://rcoil.me/2019/08/%E3%80%90%E7%9F%A5%E8%AF%86%E5%9B%9E%E9%A1%BE%E3%80%91%E6%B7%B1%E5%85%A5%E4%BA%86%E8%A7%A3%20PsExec/)
