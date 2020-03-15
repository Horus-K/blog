---
title: glance配置文件
date: 2020-03-09 15:26:58
tags: openstack
categories: openstack
---

## /etc/glance/glance-api.conf

<!--more-->

```
[DEFAULT]
[cors]
[cors.subdomain]
[database]
connection = mysql+pymysql://glance:glance@openstack-vip.qh.net/glance
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
[image_format]
[keystone_authtoken]
auth_uri = http://openstack-vip.qh.net:5000
auth_url = http://openstack-vip.qh.net:35357
memcached_servers = openstack-vip.qh.net:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance
[matchmaker_redis]
[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[paste_deploy]
flavor = keystone
flavor = keystone
[profiler]
[store_type_location_strategy]
[task]
[taskflow_executor]
```

## /etc/glance/glance-registry.conf

```
[DEFAULT]
[database]
connection = mysql+pymysql://glance:glance@openstack-vip.qh.net/glance
[keystone_authtoken]
auth_uri = http://openstack-vip.qh.net:5000
auth_url = http://openstack-vip.qh.net:35357
memcached_servers = openstack-vip.qh.net:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance
[matchmaker_redis]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_policy]
[paste_deploy]
flavor = keystone
[profiler]
```