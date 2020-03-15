---
title: keepalived非抢占模式
date: 2020-03-10 14:20:34
tags:
- keepalived
categories: 高可用
---

--------------------------

<!--more-->

```
nopreempt #关闭VIP抢占，需要VIP state都为BACKUP
vrrp_instance VI_1 {
state  BACKUP  ####
interface eth0
virtual_router_id 80
priority 100
advert_int 1
nopreempt   ########## 配在优先级高的机器上
vrrp_instance VI_1 {
state  BACKUP  ######
interface eth0
virtual_router_id 80
priority 90
advert_int 1
nopreempt ##########33
```

