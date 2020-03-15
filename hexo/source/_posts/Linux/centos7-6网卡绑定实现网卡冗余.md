---
title: centos7.6网卡绑定实现网卡冗余
date: 2020-03-10 16:57:49
tags:
- 网络
categories: 系统
---

常用的绑定驱动模式有:
mode=0平衡负载模式:平时两块网卡均工作，且自动备援，采用Switch支援。
mode=1自动备援模式:平时只有一块网卡工作，故障后自动替换为另外的网卡。
mode=2平衡策略模式:此模式提供负载平衡和容错能力
mode=6:平衡负载模式:平时两块网卡均工作，且自动备援，无须设置Switch支援。

<!--more-->

### 1.添加2块网卡

![img](centos7-6网卡绑定实现网卡冗余/a76550e49899eacebce22fd49f129bbb.png)

### 2.确认网卡正常工作

![img](centos7-6网卡绑定实现网卡冗余/666eec6f73791d7b347a902d368ebc66.png)

### 3.添加新网卡配置文件

#### 现在新添加的两块网卡均无配置文件需手动添加

![img](centos7-6网卡绑定实现网卡冗余/e3f132847ffc2f48066b94bd0ee5ecad.png)

### ifcfg-eth1

```
TYPE=Ethernet
DEVICE=eth1
ONBOOT=yes
BOOTPROTO=none
USERCTL=no
MASTER=bond0
SLAVE=yes
```

### ifcfg-eth2

```
TYPE=Ethernet
DEVICE=eth2
ONBOOT=yes
BOOTPROTO=none
USERCTL=no
MASTER=bond0
SLAVE=yes
```

### ifcfg-bond0

```
TYPE=Bond
BOOTPROTO=none
ONBOOT=yes
USERCTL=no
DEVICE=bond0
IPADDR=192.168.153.130
NETMASK=255.255.255.0
BONDING_OPTS= “miimon=100 mode=0”   //模式0，miimon是用来进行链路监测
```

### 4.bond配置,修改modprobe相关设定文件

需要关闭NetworkManager服务

```
# systemctl stop NetworkManager
# systemctl disable NetworkManager
```

### 5.查看内核是否加载bonding

重启网络服务

```
systemctl restart network
```

查看内核是否加载

```
# lsmod |grep bonding
```

![img](centos7-6网卡绑定实现网卡冗余/172f4b6504a72c5e0b70d99eae95e3ad.png)

### 6.查看是否成功(eth1与eth2 MAC地址已同步说明绑定成功)

![img](centos7-6网卡绑定实现网卡冗余/cd360c0412847ed5fbbc9aafbc511cad.png)
在另一台同网段虚拟机也可以ping通
![img](centos7-6网卡绑定实现网卡冗余/7f4162ecb00ea9a2a636eab215f2b104.png)

### 7.多网卡也只需添加一块网卡写入配置文件即可