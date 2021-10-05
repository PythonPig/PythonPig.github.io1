---
layout: post
title: Linux iptables使用记录
date: 2021-10-05 23:30:00
tags: 域渗透
categories: Linux 
author: PythonPig
---
* content
{:toc}

这篇文章的主要记录iptables的一些常用命令，便于翻阅。
{:refdef: style="text-align: center;"}
![iptables](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Linux%20iptables使用记录/iptables.jpeg?raw=true) 
{: refdef}




图片来源于https://huyangjia.com/computer-technology/899.html
### \#0x00 iptables基础知识

{:refdef: style="text-align: center;"}
![数据包流向](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Linux%20iptables使用记录/数据包流向.png?raw=true) 
{: refdef}

根据上图，数据包的流向：  
到本机某进程的报文：PREROUTING --> INPUT  
由本机转发的报文：PREROUTING --> FORWARD --> POSTROUTING  
由本机的某进程发出报文（通常为响应报文）：OUTPUT --> POSTROUTING  

4表5链：表是相似规则的合集，把链中具有相似性的规则整合到一起形成一张表 
```
filter表、nat表、mangle表、raw表
PREROUTING链、INPUT链、FORWARD链、OUTPUT链、POSTROUTING链
```

{:refdef: style="text-align: center;"}
![链与表的关系](https://github.com/PythonPig/PythonPig.github.io/blob/master/images/Linux%20iptables使用记录/链与表的关系.png?raw=true) 
{: refdef}

以prerouting链为例，prerouting链中的规则存放于三张表中，而这三张表中的规则执行的优先级如下：  
raw --> mangle --> nat  

### \#0x01 iptables使用记录
查看nat表中POSTROUTING链中的规则
```
sudo iptables -t nat -vnL POSTROUTING --line-number
```
查看filter表中INPUT链的规则（不使用-t指定表时，默认为filter表）
```
sudo iptables -t filter -vnL INPUT
或
sudo iptables -vnL INPUT
```

根据编号删除规则：删除nat表中POSTROUTING链中的第3条规则
```
iptables -t nat -D POSTROUTING 3
```
根据匹配条件和动作删除规则：删除时需要匹配规则里的所有条件和动作
```
直接把添加规则时的命令中的-A或者-I替换成-D即可
iptables -t nat -D PREROUTING -p udp -d 192.168.81.2 -s 192.168.81.1 --dport 6666  -j DNAT --to 192.168.81.2:7777
```

禁止访问
```
禁止特定ip访问22端口
iptables -I INPUT -s 10.10.10.10 -p tcp --dport 22 -j DROP

禁止特定ip访问
iptables -I INPUT -s 10.10.10.10 -j DROP
```

允许访问
```
iptables -I INPUT -p tcp --dport 80 -j DROP
iptables -I INPUT -s 192.168.1.0/24 -p tcp --dport 80 -j ACCEPT
```


DNAT
```
由eth1流入且目的为61.240.149.149:80的TCP数据包的目的地址改为192.168.10.6:80
iptables -t nat -A PREROUTING -i eth1 -p tcp -d 61.240.149.149 --dport 80 -j DNAT --to-destination 192.168.10.6:80

所有到达本机6666端口的UDP数据包全部转发至本机的7777端口
iptables -t nat -A PREROUTING -p udp -d 192.168.81.2 --dport 6666  -j DNAT --to 192.168.81.2:7777

源IP为192.168.81.1且端口为6666的UDP数据包转发至本机的7777端口
iptables -t nat -A PREROUTING -p udp -d 192.168.81.2 -s 192.168.81.1 --dport 6666  -j DNAT --to 192.168.81.2:7777

修改数据包的目的端口
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 8080
```

SNAT
```
源地址为192.168.10.0/24的数据包的源地址映射成192.168.1.108
iptables -t nat -I POSTROUTING 1 -j SNAT -s 192.168.10.0/24 --to-source 192.168.1.108

所有数据包的源地址映射成192.168.1.108
iptables -t nat -I POSTROUTING 1 -j SNAT --to-source 192.168.1.108

所有从eth0出去的数据包的源地址映射成eth0的IP地址
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

所有源地址为192.168.10.0/24的数据包动态选择并修改源地址为出口地址
iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -j MASQUERADE
```

使用tcp模块(-m tcp)指定连续的多个端口
```
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m tcp --dport 22:25 -j REJECT
```
使用multiport模块(-m multiport)指定离散的多个端口
```
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m multiport --dports 22,80 -j REJECT
```

linux配置成具有nat功能的路由器：假设eth0连接外网，wlan0连接内网
```
1、首先打开forward功能，/etc/sysctl.conf中net.ipv4.ip_forward=1
打开linux的forward功能其实就是打开了linux的路由功能，linux将会根据目的ip选择将数据包送往下一跳（从哪个网卡送出）

2、sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
打开nat功能，满足从eth0出去的数据包进行nat转换（MASQUERADE表示动态确定nat后的ip地址，即数据包从eth0转发出去时，nat后的ip为eth0的ip地址）

3、sudo iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
当数据包从wlan0进入，通过第1中的路由从eth0出时，允许该数据包通过（这里是设置从内网通向外网的数据包）

4、sudo iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
当数据包满足从eth0入、从wlan0出，且只有状态为RELATED,ESTABLISHED时，数据包才可以通过（这里是设置从外网通向内网的数据包）
```

### 参考
* [iptables详解](https://www.zsythink.net/archives/tag/iptables/)
