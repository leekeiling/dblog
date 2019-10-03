---
title: squid搭建多ip代理服务器
date: 2018-10-03 16:49:10
tags: 正向代理 squid
---
### 原理
客户端发起请求 --> squid代理服务器 --> squid代发请求  
<!--more-->

### 单网卡绑定多个ip
```
cd /etc/sysconfig/network-scripts
cp ifcfg-eth0 ifcfg-eth0:0
vim ifcfg-eth0:0
DEVICE=eth0:0 #需要将原来的eth0改为eth0:0
HWADDR=00:0c:29:8e:a7:06 #这个不能改变
TYPE=Ethernet
UUID=9e7044ec-ef90-4f4d-a6d0-c72f3f168404
ONBOOT=yes
NM_CONTROLLED=yes
BOOTPROTO=none
USERCTL=no
IPV6INIT=no
IPADDR=192.168.1.99 #新的IP地址
NETMASK=255.255.255.0
```

### 服务端squid安装
```
yum install squid -y
```
### 确认内核转发功能
```
 sysctl -a |grep -w ip_forward
```
### squid配置
```
vim /etc/squid/squid.conf
```

```
http_access allow all
http_port 3128
```
### squid多ip出口配置
```
## first ip forward
acl ip_90 myip 192.168.8.90tcp_outgoing_address 192.168.8.90 ip_90
## second ip forward
acl ip_91 myip 192.168.8.91tcp_outgoing_address 192.168.8.91 ip_91

```
### 重启squid
```
squid restart
```

### 客户端测试
```
http_proxy=$your_squid_ip:3128
https_proxy=$your_squid_ip:3128
```



