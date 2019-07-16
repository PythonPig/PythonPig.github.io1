---
layout: post
title: Windows hash dump之secretsdump
date: 2019-07-16 23:30:00
tags: 域渗透
categories: hack 
author: PythonPig
---
* content
{:toc}

在域渗透的时候经常使用impacket的secretsdump.py来获取域内主机甚至域控上的hash值，secretsdump通过多种方法获取{sam, secrets, cached and ntds}中保存的用户凭证。  
  

![](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Windows%20hash%20dump之secretsdump/stupid-hashdump.jpg?raw=true)   
图片来源于http://blog.extremehacking.org/blog/2017/06/19/make-hashdump-module-work-windows-10-sam-mode/




今天并不讲secretsdump.py的实现方法，也许某天不忙的时候会分析一下其具体实现细节。今天主要借助secretsdump.py了解一下用户凭证在windows系统中是如何存储的。
本文的主要内容：  
1、secretsdump.py的使用  
2、用户凭证在windows系统如何存储

### \#0x00 secretsdump.py的使用:
直接查看帮助是便捷且有效的方法  

``` 
secretsdump.py -h:
Impacket v0.9.20-dev - Copyright 2019 SecureAuth Corporation

usage: secretsdump.py [-h] [-debug] [-system SYSTEM] [-bootkey BOOTKEY]
                      [-security SECURITY] [-sam SAM] [-ntds NTDS]
                      [-resumefile RESUMEFILE] [-outputfile OUTPUTFILE]
                      [-use-vss] [-exec-method [{smbexec,wmiexec,mmcexec}]]
                      [-just-dc-user USERNAME] [-just-dc] [-just-dc-ntlm]
                      [-pwd-last-set] [-user-status] [-history]
                      [-hashes LMHASH:NTHASH] [-no-pass] [-k]
                      [-aesKey hex key] [-dc-ip ip address]
                      [-target-ip ip address]
                      target

Performs various techniques to dump secrets from the remote machine without
executing any agent there.

positional arguments:
  target                [[domain/]username[:password]@]<targetName or address>
                        or LOCAL (if you want to parse local files)

optional arguments:
  -h, --help            show this help message and exit
  -debug                Turn DEBUG output ON
  -system SYSTEM        SYSTEM hive to parse
  -bootkey BOOTKEY      bootkey for SYSTEM hive
  -security SECURITY    SECURITY hive to parse
  -sam SAM              SAM hive to parse
  -ntds NTDS            NTDS.DIT file to parse
  -resumefile RESUMEFILE
                        resume file name to resume NTDS.DIT session dump (only
                        available to DRSUAPI approach). This file will also be
                        used to keep updating the session's state
  -outputfile OUTPUTFILE
                        base output filename. Extensions will be added for
                        sam, secrets, cached and ntds
  -use-vss              Use the VSS method insead of default DRSUAPI
  -exec-method [{smbexec,wmiexec,mmcexec}]
                        Remote exec method to use at target (only when using
                        -use-vss). Default: smbexec

display options:
  -just-dc-user USERNAME
                        Extract only NTDS.DIT data for the user specified.
                        Only available for DRSUAPI approach. Implies also
                        -just-dc switch
  -just-dc              Extract only NTDS.DIT data (NTLM hashes and Kerberos
                        keys)
  -just-dc-ntlm         Extract only NTDS.DIT data (NTLM hashes only)
  -pwd-last-set         Shows pwdLastSet attribute for each NTDS.DIT account.
                        Doesn't apply to -outputfile data
  -user-status          Display whether or not the user is disabled
  -history              Dump password history, and LSA secrets OldVal

authentication:
  -hashes LMHASH:NTHASH
                        NTLM hashes, format is LMHASH:NTHASH
  -no-pass              don't ask for password (useful for -k)
  -k                    Use Kerberos authentication. Grabs credentials from
                        ccache file (KRB5CCNAME) based on target parameters.
                        If valid credentials cannot be found, it will use the
                        ones specified in the command line
  -aesKey hex key       AES key to use for Kerberos Authentication (128 or 256
                        bits)

connection:
  -dc-ip ip address     IP Address of the domain controller. If ommited it use
                        the domain part (FQDN) specified in the target
                        parameter
  -target-ip ip address
                        IP Address of the target machine. If omitted it will
                        use whatever was specified as target. This is useful
                        when target is the NetBIOS name and you cannot resolve
                        it
```
基本使用方法：
```
python secretsdump.py domain/username@10.10.10.10 -hashes LM HASH:NT HASH 
```
secretsdump.py主要从SAM、LSA secrets(包括 cached creds)和域控的NTDS.dit三处获取用户凭证，唯一的一点是不能dump LSASS进程在内存中的数据。  

### \#0x02 用户凭证在windows系统如何存储:
查看微软官方的说明
![用户凭证在windows系统中的存储情况](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Windows%20hash%20dump之secretsdump/用户凭证在windows系统中的存储情况%20copy.png?raw=true) 

根据微软官网的介绍，用户凭证一般存储在SAM、LSA secrets、NTDS.DIT和LSASS进程的内存中，另外系统缓存中也有可能存在用户凭证，参见：  
[Cached Domain Credentials](https://moyix.blogspot.com/2008/02/cached-domain-credentials.html)  
[Interactive logon: Number of previous logons to cache (in case domain controller is not available)](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/jj852209%28v%3dws.11%29)  


secretsdump可以从SAM、LSA secrets(包括 cached creds)和域控的NTDS.dit获取用户凭证，LSASS进程的内存数据通过[绕过杀软导出域内用户hash的方法记录](https://pythonpig.github.io/2018/12/13/绕过杀软导出域内用户hash方法记录/)获取。  


### 参考
* [Cached Domain Credentials](https://moyix.blogspot.com/2008/02/cached-domain-credentials.html)  
* [Interactive logon: Number of previous logons to cache (in case domain controller is not available)](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/jj852209%28v%3dws.11%29)  
* [绕过杀软导出域内用户hash的方法记录](https://pythonpig.github.io/2018/12/13/绕过杀软导出域内用户hash方法记录/)