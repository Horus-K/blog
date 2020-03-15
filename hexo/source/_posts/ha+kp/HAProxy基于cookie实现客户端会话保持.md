---
title: HAProxy基于cookie实现客户端会话保持
date: 2020-03-10 17:32:42
tags:
- haproxy
categories: haproxy
---

## 缓存功能

<!--more-->

```bash
[root@CentOS7-2 ~]#vim /etc/haproxy/haproxy.cfg
listen web1
  mode http
  bind 192.168.36.104:80
  option forwardfor
  cookie SERVER-COOKIE insert indirect nocache
  server web1 192.168.36.110:80 cookie web-110 weight 1 check inter 3000 fall 3 rise 5
  server web2 192.168.36.106:80 cookie web-106 weight 1 check inter 3000 fall 3 rise 5

[root@CentOS7-2 ~]#systemctl restart haproxy.service
```