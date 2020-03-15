---
title: nmcli实现Network-Teaming
date: 2020-03-10 16:56:49
tags:
- nmcli
- Network-Teaming
categories: 系统
---

网络组：是将多个网卡聚合在一起方法，从而实现冗错和提高吞吐量
网络组不同于旧版中bonding技术，提供更好的性能和扩展性
网络组由内核驱动和teamd守护进程实现.

<!--more-->

多种方式runner
broadcast
roundrobin
activebackup
loadbalance
lacp (implements the 802.3ad Link Aggregation Control Protocol)
![img](nmcli实现Network-Teaming/9dfedabd72360e20bb522aba61fc1f79.png)
这是已经配好的，NetworkManger支持多配置文件，想删时需要先down网卡

```
[root@localhost ~]# nmcli connection show
[root@localhost ~]# nmcli connection down NWteam0
```

![img](nmcli实现Network-Teaming/02097343774069c9fff7f3cf09bcdcf4.png)

```
[root@localhost ~]# nmcli connection delete NWteam0 (删除网卡配置文件)
```

![img](nmcli实现Network-Teaming/308ea3f157106e7685efda867ca74b16.png)

## 1创建网络组接口

```bash
nmcli con add type team con-name CNAME ifname INAME [config JSON]
CNAME 连接名，INAME 接口名
JSON 指定runner方式
格式：'{"runner": {"name": "METHOD"}}'
METHOD 可以是broadcast, roundrobin,
activebackup, loadbalance, lacp
[root@localhost ~]# nmcli connection add con-name OFFICE type team ifname office config '{"runner":{"name":"loadbalance"}}' ipv4.addresses 192.168.153.150 ipv4.method manual
```

![img](nmcli实现Network-Teaming/06a7698d79e26a76d9b0bc6eb556da1d.png)
![img](nmcli实现Network-Teaming/29238011f5d980132ff24660e555146e.png)

## 2创建port接口

```
[root@localhost ~]# nmcli connection add con-name OFFICE-1 type team-slave ifname ens33 master OFFICE
[root@localhost ~]# nmcli connection add con-name OFFICE-2 type team-slave ifname ens38 master OFFICE
```

![img](nmcli实现Network-Teaming/4251b0ef0a7ff72b404bf2806e236dbc.png)
(绿色代表已开启的配置文件)

## 3测试

```
[root@localhost ~]# nmcli connection up OFFICE # 开启OFFICE配置
```