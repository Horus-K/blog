---
title: haproxy配置文件
date: 2020-03-09 15:51:34
tags: openstack
categories: openstack
---

haproxy配置文件

<!--more-->

```
listen MYSQL_PORT_3306
        bind 192.168.220.248:3306
        mode tcp
        balance roundrobin
        server mysql 192.168.50.102:3306 weight 1 check port 3306 inter 3s fall 3 rise 5

listen OPENSTACK_RABBIT
        bind 192.168.220.248:5672
        mode tcp
        balance roundrobin
        server rabbie 192.168.50.102:5672 weight 1 check port 5672 inter 3s fall 3 rise 5
#        server web1 192.168.50.101:5672 weight 1 check port 3306 inter 3s fall 3 rise 5 backup

listen OPENSTACK_RABBIT_15672
        bind 192.168.220.248:15672
        mode tcp
        balance roundrobin
        server rabbie 192.168.50.102:15672 weight 1 check port 15672 inter 3s fall 3 rise 5
listen OPENSTACK_RABBIT_25672
        bind 192.168.220.248:25672
        mode tcp
        balance roundrobin
        server rabbie 192.168.50.102:25672 weight 1 check port 25672 inter 3s fall 3 rise 5
listen OPENSTACK_memcache
        bind 192.168.220.248:11211
        mode tcp
        balance roundrobin
#        server web1 192.168.50.101:11211 weight 1 check port 3306 inter 3s fall 3 rise 5
        server mem 192.168.50.102:11211 weight 1 check port 11211 inter 3s fall 3 rise 5
listen openstack_keystone_5000
        bind 192.168.220.248:5000
        mode tcp
        balance roundrobin
        server keystone5000 192.168.50.202:5000 weight 1 check port 5000 inter 3s fall 3 rise 5

listen openstack_keystone_35357
        bind 192.168.220.248:35357
        mode tcp
        balance roundrobin
        server keystone1 192.168.50.202:35357 weight 1 check port 35357 inter 3s fall 3 rise 5



listen openstack_glance_api
        bind 192.168.220.248:9292
        mode tcp
        balance roundrobin
        server api1 192.168.50.202:9292 weight 1 check port 9292 inter 3s fall 3 rise 5


listen openstack_glance
        bind 192.168.220.248:9191
        mode tcp
        balance roundrobin
        server glance1 192.168.50.202:9191 weight 1 check port 9191 inter 3s fall 3 rise 5


listen openstack_nova_8774
        bind 192.168.220.248:8774
        mode tcp
        balance roundrobin
        server glance1 192.168.50.202:8774 weight 1 check port 8774 inter 3s fall 3 rise 5


listen openstack_nova_8775
        bind 192.168.220.248:8775
        mode tcp
        balance roundrobin
        server glance1 192.168.50.202:8775 weight 1 check port 8775 inter 3s fall 3 rise 5

listen openstack_nova_8778
        bind 192.168.220.248:8778
        mode tcp
        balance roundrobin
        server glance1 192.168.50.202:8778 weight 1 check port 8778 inter 3s fall 3 rise 5


listen openstack_vnc
        bind 192.168.220.248:6080
        mode tcp
        balance roundrobin
        server glance1 192.168.50.202:6080 weight 1 check port 6080 inter 3s fall 3 rise 5

listen openstack_nova_9696
        bind 192.168.220.248:9696
        mode tcp
        balance roundrobin
        server glance1 192.168.50.202:9696 weight 1 check port 9696 inter 3s fall 3 rise 5

listen openstack_dashboard
        bind 192.168.220.248:80
        mode tcp
        balance roundrobin
        server glance1 192.168.50.202:80 weight 1 check port 80 inter 3s fall 3 rise 5
root@ubuntu:~# cat /etc/sysctl.conf 
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
cp /usr/share/doc/keepalived/samples/keepalived.conf.sample /etc/keepalived/keepalived.conf
global_defs {
   notification_email {
     acassen
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   vrrp_iptables
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    interface eth0
    virtual_router_id 50
    nopreempt

    priority 100
    advert_int 1
    virtual_ipaddress {
       192.168.220.248 dev eth0 label eth0:0
    }
}
```