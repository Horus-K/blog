---
title: Keeplived单播
date: 2020-03-10 14:21:35
tags:
- keepalived
categories: 高可用
---

[root@localhost ~]#tcpdump -i eth0 host -nn 224.0.0.18

<!--more-->

![img](Keeplived单播/screenshot_20190607_164251.png)

![img](Keeplived单播/screenshot_20190607_172025.png)前提

![img](Keeplived单播/screenshot_20190607_170705.png)node1配置

```
vrrp_instance VIP1 {
    state MASTER
    interface eth0
    virtual_router_id 1
    priority 100
    advert_int 3
    unicast_src_ip 192.168.64.110
        unicast_peer {
            192.168.64.120
        }
    authentication {
        auth_type PASS
        auth_pass linux
    }
    virtual_ipaddress {
        192.168.64.200/24 dev eth0 label eth0:0
    }
}
```

![img](Keeplived单播/screenshot_20190607_170743.png)node2配置

```
vrrp_instance VIP1 {
    state BACKUP
    interface eth0
    virtual_router_id 1
    priority 80
    advert_int 3
    unicast_src_ip 192.168.64.120
        unicast_peer {
            192.168.64.110
        }
    authentication {
        auth_type PASS
        auth_pass linux
    }
    virtual_ipaddress {
        192.168.64.200/24 dev eth0 label eth0:0
    }
}
```

![img](Keeplived单播/screenshot_20190607_172507.png)抓包