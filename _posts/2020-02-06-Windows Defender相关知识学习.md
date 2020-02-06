---
layout: post
title: Windows Defender相关知识学习
date: 2020-02-06 23:30:00
tags: Windows-Defender
categories: hack 
author: PythonPig
---
* content
{:toc}

记录Windows Defender相关知识和操作。  

{:refdef: style="text-align: center;"}
![windefend](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Windows%20Defender相关知识学习/Windows-Defender.jpg?raw=true) 
{: refdef}




图片来源于https://www.techzine.be/nieuws/security/45304/windows-defender-faalt-malware-scan-na-enkele-seconden/  

### \#0x00 相关操作记录
在渗透测试过程中，经常遇到Windows Defender，最近通过阅读微软官方文档找到了一些通过powershell操作WinDefend的方法。  
渗透测试时，先获取并保存目标原有配置，然后添加ExclusionPath或ExclusionProcess，操作完成后恢复原有配置。  

1、查看Windefend配置：  
```
powershell.exe Get-MpPreference
```
2、添加exclusion路径
```
powershell.exe Set-MpPreference -ExclusionPath "C:\temp", "C:\VMs", "C:\NanoServer"
```
3、添加exclusion程序
```
powershell.exe Set-MpPreference -ExclusionProcess "vmms.exe", "Vmwp.exe"
```
4、删除exclusion路径(Remove Exclusion Path) 
```
powershell.exe Remove-MpPreference -ExclusionPath "C:\temp", "C:\VMs"
```
5、删除exclusion程序(Remove Exclusion Process)
```
powershell.exe Remove-MpPreference -ExclusionProcess "vmms.exe", "Vmwp.exe"
```
6、关闭Windefend实时检测(Disable Real-Time Protection)
```
powershell.exe Set-MpPreference -DisableRealtimeMonitoring $true
```
7、打开Windefend实时检测(Enable Real-Time Protection)
```
powershell.exe Set-MpPreference -DisableRealtimeMonitoring $false
```


### 参考
* [Microsoft Docs - Defender](https://docs.microsoft.com/en-us/powershell/module/defender/?view=win10-ps)
* [HOW TO DISABLE AND CONFIGURE WINDOWS DEFENDER ON WINDOWS SERVER 2016 USING POWERSHELL](https://www.thomasmaurer.ch/2016/07/how-to-disable-and-configure-windows-defender-on-windows-server-2016-using-powershell/)  