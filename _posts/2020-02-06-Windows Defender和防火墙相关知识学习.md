---
layout: post
title: Windows Defender和防火墙相关知识学习
date: 2020-02-06 23:30:00
tags: Windows-Defender
categories: Windows 
author: PythonPig
---
* content
{:toc}

记录Windows Defender和防火墙相关知识和操作。  

{:refdef: style="text-align: center;"}
![windefend](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Windows%20Defender相关知识学习/Windows-Defender.jpg?raw=true) 
{: refdef}




图片来源于https://www.techzine.be/nieuws/security/45304/windows-defender-faalt-malware-scan-na-enkele-seconden/  

### \#0x00 Windows Defender相关操作记录
在渗透测试过程中，经常遇到Windows Defender，最近通过阅读微软官方文档找到了一些通过powershell操作WinDefend的方法。  
渗透测试时，先获取并保存目标原有配置，然后添加ExclusionPath或ExclusionProcess，操作完成后恢复原有配置。  

查看Windefend配置：  
```
powershell.exe Get-MpPreference
```

简单的方法查看windefend状态
```
sc query windefend
```

查看状态
```
powershell.exe Get-MpComputerStatus
```

添加exclusion路径
```
powershell.exe Set-MpPreference -ExclusionPath "C:\temp", "C:\VMs", "C:\NanoServer"
```

添加exclusion程序
```
powershell.exe Set-MpPreference -ExclusionProcess "vmms.exe", "Vmwp.exe"
```

删除exclusion路径(Remove Exclusion Path) 
```
powershell.exe Remove-MpPreference -ExclusionPath "C:\temp", "C:\VMs"
```

删除exclusion程序(Remove Exclusion Process)
```
powershell.exe Remove-MpPreference -ExclusionProcess "vmms.exe", "Vmwp.exe"
```

关闭Windefend实时检测(Disable Real-Time Protection)
```
powershell.exe Set-MpPreference -DisableRealtimeMonitoring $true
```

打开Windefend实时检测(Enable Real-Time Protection)
```
powershell.exe Set-MpPreference -DisableRealtimeMonitoring $false
```

彻底关闭winden，重启生效
```
New-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows Defender" -Name DisableAntiSpyware -Value 1 -PropertyType DWORD -Force
```

### \#0x01 Windows 防火墙相关操作记录

1、查看防火墙状态  
```
Netsh Advfirewall show allprofiles

Netsh Advfirewall show domainprofile
Netsh Advfirewall show Privateprofile
Netsh Advfirewall show Publicprofile

netsh firewall show state  //查看添加的规则
```
2、开启防火墙
```
netsh firewall set opmode mode=enable
或
netsh advfirewall set allprofiles state on
```
3、关闭防火墙
```
netsh firewall set opmode mode=disable
或
netsh advfirewall set allprofiles state off
```
4、关闭domainprofile防火墙
```
netsh advfirewall set domainprofile state off
```
5、关闭tcp 445端口
```
netsh advfirewall firewall add rule name=”deny tcp 445″ dir=in protocol=tcp localport=445 action=block
```
6、打开tcp 445端口
```
netsh advfirewall firewall add rule name="allow tcp 445" dir=in localport=445 protocol=tcp action=allow
```
7、设置默认inbound和outbound策略
```
netsh advfirewall set allprofiles firewallpolicy allowinbound,allowoutbound
```
8、删除规则  
```
netsh advfirewall firewall delete rule name=test
```
### 参考
* [Microsoft Docs - Defender](https://docs.microsoft.com/en-us/powershell/module/defender/?view=win10-ps)
* [HOW TO DISABLE AND CONFIGURE WINDOWS DEFENDER ON WINDOWS SERVER 2016 USING POWERSHELL](https://www.thomasmaurer.ch/2016/07/how-to-disable-and-configure-windows-defender-on-windows-server-2016-using-powershell/)  