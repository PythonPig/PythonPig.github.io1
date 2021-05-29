---
layout: post
title: non-built-in Administrator使用管理员权限远程执行命令
date: 2021-05-29 23:30:00
tags: privilege_escalation
categories: hack 
author: PythonPig
---
* content
{:toc}

在域渗透的时候经常使用impacket的secretsdump.py来获取域内主机甚至域控上的hash值，secretsdump可以通过多种方法获取{sam, secrets, cached and ntds}中保存的用户凭证。 
最近在使用impacket的wmiexec.py对非域内主机进行渗透时遇到了rpc_s_access_denied、WBEM_E_ACCESS_DENIED的问题，同时发现新添加的管理员对系统默认的共享文件夹（admin$和c$）没有远程访问权限。为了解决上述问题，花了些时间找了下原因，这里做个记录。   
  
{:refdef: style="text-align: center;"}
![access denied](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/non-built-in%20Administrator使用管理员权限远程执行命令/access_denied.jpeg?raw=true)
{: refdef}   





图片来源于https://www.login-as.no/win10-uac-remote-restrictions/

根据用户所属的用户组的不同可以将用户分为built-in Administrator、non-built-in administrative accounts、non-administrative accounts。  
built-in Administrator是系统本身自带的管理员用户，其SID为500，具有系统默认的管理员权限；non-built-in administrative accounts是新添加的管理员用户，是administrators组中的用户，其SID非500;non-administrative accounts是不在administrators组中的用户，即非管理员用户。  

下面主要针对non-built-in administrative accounts、non-administrative accounts 2类用户在横向渗透中使用sc类(psexec,scshell)、wmi类(wmiexec.wmic)、smb类(smbexec,ipc,smbclient)、winrm类等工具可能遇到的权限问题进行讨论。  

分文主要以wmiexec.py使用过程中的权限问题进行讨论，本文讨论的方法在windows 7 x64和windows 10 x64下测试通过。  



### \#0x00 Remote UAC
windows在windows Vista之后引入了一种默认开启的Remote UAC机制，计算机的任何非SID为500本地管理员帐户，无法执行管理任务。  
关于Remote UAC更多内容参见本文最后的“参考”部分。  

Remote UAC主要限制了on-administrative accounts这类用户的权限，对于域管用户、存在本地管理员组的域用户无影响。


### \#0x01 非built-in管理员用户使用wmi
在横向渗透的过程中会遇到这样的问题，新添加的管理员用户在使用wmiexec.py时，提示没有权限，原因就是Remote UAC限制了非built-in管理员用户的远程管理权限。  
对于非built-in管理员用户的这用问题，解决方法很简单，关闭Remote UAC即可。通过新增注册表项LocalAccountTokenFilterPolicy并赋值为1即可，可通过如下命令行完成。
```
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" /v "LocalAccountTokenFilterPolicy" /t REG_DWORD /d 1 /f
```


### \#0x02 非管理员用户使用wmi
非管理员用户使用wmi远程执行命令时可能遇到的问题包括rpc_s_access_denied、WBEM_E_ACCESS_DENIED和命令执行无回显等问题，下面依次分析如何解决这3个问题。  

1、rpc_s_access_denied  
出现上述问题的原因是用户没有DCOM用户权限，解决方法如下：  
```
run dcomcnfg > Component Services > Computers > My Computer > (right click) properties > COM security > (Access Permissions and Launch and Activation Permissions)Edit Limits. Add Users and allow remote access, remote launch, and remote activation.
```
2、WBEM_E_ACCESS_DENIED  
出现上述问题的原因是没有设置命名空间权限，解决方法如下：
```
启动 wmimgmt.msc
在WMI控制窗格中，右键单击WMI控制，选择属性，然后选择安全。
高亮ROOT，点击安全。
添加用户，并赋予所有权限（其实赋予远程权限即可）。
点击高级，选择新增的用户，编辑，应用于本目录和子目录。
```
3、命令执行无回显
出现这个问题的原因是非管理员用户没有读写admin$的权限。  
因为wmiexec.py回显是通过将结果写入ADMIN$然后读回文件内容实现的，若用户没有读写ADMIN$的权限，则wmiexec.py无法顺利执行。  
解决该问题有两种方法  
第一种是使用-nooutput参数，使用该参数后wmiexec.py可以顺利完成命令执行，但是无回显。  
第二种方法是寻找或新建一个隐蔽的文件夹并赋予新建用户读写权限，如：  
```
properties > sharing > network file and folder sharing 中给admin赋予权限并共享
properties > sharing > advanced sharing中可以修改共享名字，如改为public
```
使用时的命令如下：
wmiexec.py admin@10.10.10.10 -share public
这种方法虽然可以使用wmiexec.py执行部分系统命令，但是由于用户是非管理员用户，其权限较低，对于高权限的操作和目录访问均受到限制。


### 参考
* [Description of User Account Control and remote restrictions in Windows Vista](https://docs.microsoft.com/en-us/troubleshoot/windows-server/windows-security/user-account-control-and-remote-restriction)
* [Which permissions/rights does a user need to have WMI access on remote machines?](https://serverfault.com/questions/28520/which-permissions-rights-does-a-user-need-to-have-wmi-access-on-remote-machines)  
* [配置对 WMI 的用户访问](https://www.dell.com/support/manuals/zh-cn/omimssc-sccm-scvmm-v7.1/omimssc-v7.1-sccm-scvmm-bpg/配置对-wmi-的用户访问?guid=guid-7b6e205a-a1ac-4658-a2ed-729880afb50c&lang=zh-cn)  
* [How to Disable a Remote UAC for a non-built-in Administrator](https://documentation.arcserve.com/Arcserve-UDP/Available/V6.5/ENU/Bookshelf_Files/HTML/Solutions%20Guide/UDPSolnGuide/udp_disable_uac_remt_admin.htm)  
* [windows横向渗透中的令牌完整性限制](https://www.secpulse.com/archives/145590.html)  
* [Managing Administrative Shares (Admin$, IPC$, C$, D$) in Windows 10](http://woshub.com/enable-remote-access-to-admin-shares-in-workgroup/)
* [高级域渗透技术之传递哈希已死-LocalAccountTokenFilterPolicy万岁](https://blog.csdn.net/systemino/article/details/89716729)
