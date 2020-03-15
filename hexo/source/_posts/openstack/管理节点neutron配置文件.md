---
title: 管理节点neutron配置文件
date: 2020-03-09 15:50:35
tags: openstack
categories: openstack
---

## /etc/neutron/neutron.conf

<!--more-->

```
[DEFAULT]
core_plugin = ml2
service_plugins =
transport_url = rabbit://openstack:openstack123@rabbitmq-server2
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
[agent]
[cors]
[cors.subdomain]
[database]
connection = mysql+pymysql://neutron:neutron123@openstack-vip.qh.net/neutron
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
auth_url = http://openstack-vip.qh.net:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = nova
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

## /etc/neutron/dhcp_agent.ini

```
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
[agent]
[ovs]
```

## /etc/neutron/metadata_agent.ini

```
[DEFAULT]
nova_metadata_ip = openstack-vip.qh.net
metadata_proxy_shared_secret = 20190621
[agent]
[cache]
```

## /etc/neutron/plugin.ini

```
[DEFAULT]
[ml2]
type_drivers = flat,vlan
tenant_network_types = 
mechanism_drivers = linuxbridge
extension_drivers = port_security
[ml2_type_flat]
flat_networks = linux36,linux37
[ml2_type_geneve]
[ml2_type_gre]
[ml2_type_vlan]
[ml2_type_vxlan]
[securitygroup]
enable_security_group = true
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

## /etc/neutron/plugins/ml2/ml2_conf.ini

```
[DEFAULT]
[ml2]
type_drivers = flat,vlan
tenant_network_types = 
mechanism_drivers = linuxbridge
extension_drivers = port_security
[ml2_type_flat]
flat_networks = linux36,linux37
[ml2_type_geneve]
[ml2_type_gre]
[ml2_type_vlan]
[ml2_type_vxlan]
[securitygroup]
enable_security_group = true
```