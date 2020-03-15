---
title: 计算节点neutron配置文件
date: 2020-03-09 15:30:46
tags: openstack
categories: openstack
---

## /etc/neutron/neutron.conf

<!--more-->

```
[DEFAULT]
transport_url = rabbit://openstack:openstack123@rabbitmq-server2
auth_strategy = keystone
[agent]
[cors]
[cors.subdomain]
[database]
[keystone_authtoken]
auth_uri = http://openstack-vip.qh.net:5000
auth_url = http://openstack-vip.qh.net:35357
memcached_servers = openstack-vip.qh.net:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron
[matchmaker_redis]
[nova]
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[qos]
[quotas]
[ssl]
```

## /etc/neutron/plugins/ml2/linuxbridge_agent.ini

```
[DEFAULT]
[agent]
[linux_bridge]
physical_interface_mappings = linux36:bond0,linux37:bond1
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
[vxlan]
enable_vxlan = false
```