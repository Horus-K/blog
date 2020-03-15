---
title: keepalived双主服务器
date: 2020-03-10 14:20:02
tags:
- keepalived
categories: 高可用
---

node1：虚拟200，201
node2：虚拟202，203

<!--more-->

vip1 在node1为主 优先级100 node2为备 优先级80
vip2 在node2为主 优先级100 node1为备 优先级80

![img](keepalived双主服务器/screenshot_20190607_180440.png)node1

![img](keepalived双主服务器/screenshot_20190607_180453.png)node2