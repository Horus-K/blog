---
title: 计算节点nova配置文件
date: 2020-03-09 15:48:34
tags: openstack
categories: openstack
---

## /etc/nova/nova.conf

<!--more-->

```
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:openstack123@rabbitmq-server2
my_ip = 192.168.50.105
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
[api]
auth_strategy=keystone
[api_database]
connection = mysql+pymysql://nova:nova123@openstack-vip.qh.net/nova
[barbican]
[cache]
[cells]
[cinder]
[cloudpipe]
[conductor]
[console]
[consoleauth]
[cors]
[cors.subdomain]
[crypto]
[database]
[ephemeral_storage_encryption]
[filter_scheduler]
discover_hosts_in_cells_interval=3000
[glance]
api_servers = http://openstack-vip.qh.net:9292
[guestfs]
[healthcheck]
[hyperv]
[image_file_url]
[ironic]
[key_manager]
[keystone_authtoken]
auth_uri = http://openstack-vip.qh.net:5000
auth_url = http://openstack-vip.qh.net:35357
memcached_servers = openstack-vip.qh.net:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = nova
[libvirt]
virt_type = qemu
[matchmaker_redis]
[metrics]
[mks]
[neutron]
url = http://openstack-vip.qh.net:9696
auth_url = http://openstack-vip.qh.net:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
service_metadata_proxy = true
metadata_proxy_shared_secret = 20190621
[notifications]
[osapi_v21]
[oslo_concurrency]
lock_path=/var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[pci]
[placement]
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://openstack-vip.qh.net:35357/v3
username = placement
password = placement
[quota]
[rdp]
[remote_debug]
[scheduler]
[serial_console]
[service_user]
[spice]
[ssl]
[trusted_computing]
[upgrade_levels]
[vendordata_dynamic_auth]
[vmware]
[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://openstack-vip.qh.net:6080/vnc_auto.html
[workarounds]
[wsgi]
[xenserver]
[xvp]
```