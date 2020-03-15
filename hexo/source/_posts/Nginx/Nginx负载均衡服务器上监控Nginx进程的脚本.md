---
title: Nginx负载均衡服务器上监控Nginx进程的脚本
date: 2020-03-07 17:10:40
tags:
  - nginx
  - 脚本
categories:
  - nginx
---

企业负载均衡层如果用到Nginx+Keepalived架构，而Keepalived无法进行Nginx服务的实时切换，所以这里用了一个监控脚本check_nginx_pid.sh，每隔5秒就监控一次Nginx的运行状态，如果发现有问题就关闭本机的Keepalived程序，让VIP切换到从Nginx负载均衡器上。

<!--more-->

```shell
shell> vim check_nginx_pid.sh

#!/bin/bash
while :
do
nginxpid='ps -C nginx --no-header | wc -l'
if ［$nginxpid -eq 0 ］;then
　　ulimit -SHn 65535
　　/usr/local/nginx/sbin/nginx
sleep 5
　nginxpid='ps -C nginx --no-header | wc -l'
　if ［$nginxpid -eq 0 ］;then
　/etc/init.d/keepalived stop
　fi
fi
sleep 5
done
```

