---
title: OpenStack网络服务部署neutron
date: 2020-03-09 15:55:22
tags: openstack
categories: openstack
---

neutron是openstack的网络组件，是OpenStack的网络服务

<!--more-->

网络类型：

提供者网络：虚拟机桥接到物理机，并且虚拟机必须和物理机在同一个网络范围内。

自服务网络：可以自己创建网络，最终会通过虚拟路由器连接外网

```
1、创建数据库并授权
MariaDB [(none)]> CREATE DATABASE neutron;

MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'neutron123';
2、测试连接数据库
[root@openstack ~]# mysql -uneutron -pneutron123 -h10.20.0.100
Welcome to the MariaDB monitor. Commands end with ; or \g.
Your MariaDB connection id is 3088
Server version: 10.1.20-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| neutron            |
| test               |
+--------------------+
3 rows in set (0.02 sec)
3、创建neutron用户
[root@openstack ~]# openstack user create --domain default --password-prompt neutron
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | e07d0a220bf64db69c68fb128486472a |
| enabled             | True                             |
| id                  | 9a35bc9f0add4decbed162b3314d0146 |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

4、将neutron用户加入service项目，并赋予admin角色的权限
[root@openstack ~]# openstack role add --project service --user neutron admin

5、创建neutron服务实体
[root@openstack ~]# openstack service create --name neutron \
--description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | 5528cd69d011463ea81281fe0182a654 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+

6、注册neutron服务API端点
openstack endpoint create --region RegionOne network public http://openstack-vip.qh.com:9696 
openstack endpoint create --region RegionOne network internal http://openstack-vip.qh.com:9696 
openstack endpoint create --region RegionOne network admin http://openstack-vip.qh.com:9696 



[root@openstack ~]# openstack endpoint create --region RegionOne network public http://openstack-vip.qh.com:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | a930b5f0ee0e44439e211afe67a7a3a2 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 5528cd69d011463ea81281fe0182a654 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://openstack-vip.net:9696    |
+--------------+----------------------------------+
[root@openstack ~]# openstack endpoint create --region RegionOne network internal http://openstack-vip.qh.com:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 37c14697e2da46a5960defff9ccdd729 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 5528cd69d011463ea81281fe0182a654 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://openstack-vip.net:9696    |
+--------------+----------------------------------+
[root@openstack ~]# openstack endpoint create --region RegionOne network admin http://openstack-vip.qh.com:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 04b9771687924f25b0f8b8dc89d3232e |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 5528cd69d011463ea81281fe0182a654 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://openstack-vip.net:9696    |
+--------------+----------------------------------+

7、在控制端安装提供者网络的组件
###安装阿里源
[root@openstack ~]# yum install openstack-neutron openstack-neutron-ml2 \
openstack-neutron-linuxbridge ebtables

8、编辑neutron.conf配置文件
[root@openstack ~]# vim /etc/neutron/neutron.conf
[DEFAULT]
core_plugin = ml2                  #启用ml2插件
service_plugins =                  #禁用其他插件
transport_url = rabbit://openstack:openstack123@rabbitmq.qh.com      #消息队列
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[database]                        #配置数据库访问
connection = mysql+pymysql://neutron:neutron123@openstack-vip.qh.com/neutron

[keystone_authtoken]              #认证服务
auth_uri = http://openstack-vip.qh.com:5000
auth_url = http://openstack-vip.qh.com:35357
memcached_servers = openstack-vip.qh.com:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron

[nova]                   #配置网络服务来通知计算节点的网络拓扑变化
auth_url = http://openstack-vip.qh.com:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = nova

[oslo_concurrency]              #配置锁路径
lock_path = /var/lib/neutron/tmp


9、配置桥接配置
[root@openstack ~]# vim /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
type_drivers = flat,vlan             #启用flat和vlan
tenant_network_types =               #禁用私有网络
mechanism_drivers = linuxbridge      #启用linuxbridge机制
extension_drivers = port_security    #启用端口安全

[ml2_type_flat]                      #配置共有虚拟网络为flat
flat_networks = linux36          #名称，可以更改不加网卡名

[securitygroup]                      #启用ipset增加安全组的方便性
enable_security_group = true


10、配置DHCP代理
[root@openstack ~]# vim /etc/neutron/dhcp_agent.ini
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true

11、配置元数据代理
添加控制端的IP或域名，密码一般填写年月日
[root@openstack ~]# vim /etc/neutron/metadata_agent.ini
[DEFAULT]                             
nova_metadata_ip = openstack-vip.qh.com
metadata_proxy_shared_secret = 20190621

12、在nova配置文件中配置认证账号密码，告诉nova如何找neutron做认证
[root@openstack ~]# vim /etc/nova/nova.conf 
[neutron]                        #配置访问参数，启用元数据代理，并设置密码
url = http://openstack-vip.qh.com:9696
auth_url = http://openstack-vip.qh.com:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
service_metadata_proxy = true
metadata_proxy_shared_secret = 20190621    #密码是上一步的元数据代理设置的密码

13、创建一个软连接
网络服务初始化脚本需要一个软链接，创建出软连接
[root@openstack ~]# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini

14、定义映射关系
[root@openstack ~]# vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[linux_bridge]                 #将公共虚拟网络和公共物理网络接口对应起来
physical_interface_mappings = mylinux:br0      #第9步配置的名称
 与/etc/neutron/plugins/ml2/ml2_conf.ini 中网卡名一样
[vxlan]                        #禁用vxlan
enable_vxlan = false

[securitygroup]                #启用安全组并配置linux桥接iptables防火墙
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
15、初始化数据库
[root@openstack ~]# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
--config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

![img](OpenStack网络服务部署neutron/neutron1.png)

```
查看数据库生成的数据
```

![img](OpenStack网络服务部署neutron/neutron3.png)

```
16、重启计算API服务
[root@openstack ~]# systemctl restart openstack-nova-api.service

17、启动neutron并设置为开机启动
[root@openstack ~]# systemctl enable neutron-server.service \
neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
neutron-metadata-agent.service

[root@openstack ~]# systemctl start neutron-server.service \
neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
neutron-metadata-agent.service
```

![img](OpenStack网络服务部署neutron/neutron4.png)

```
18、配置负载均衡
listen openstack_neutron_port_9696
  bind 10.20.0.40:9696
  mode tcp
  log global
  server 10.20.0.60 10.20.0.60:9696 check inter 3000 fall 2 rise 5
  server 10.20.0.70 10.20.0.70:9696 check inter 3000 fall 2 rise 5 backup
```

**===========================================================================**

## 配置计算节点

```
1、安装neutron组件
[root@note1 ~]# yum install openstack-neutron-linuxbridge ebtables ipset

2、修改配置文件
[root@note1 ~]# vim /etc/neutron/neutron.conf
[DEFAULT]
transport_url = rabbit://openstack:openstack123@rabbitmq.qh.com
auth_strategy = keystone

[keystone_authtoken]
auth_uri = http://openstack-vip.qh.com:5000
auth_url = http://openstack-vip.qh.com:35357
memcached_servers = openstack-vip.qh.com:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp

3、提供者网络配置linuxbridge
[root@note1 ~]# vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[linux_bridge]
physical_interface_mappings = linux36:bond0    #mylinux和控制端设置的保持一致

[vxlan]
enable_vxlan = false

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

4、配置nova
[root@note1 ~]# vim /etc/nova/nova.conf 
[neutron]
url = http://openstack-vip.qh.com:9696
auth_url = http://openstack-vip.qh.com:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
service_metadata_proxy = true
metadata_proxy_shared_secret = 20190621

5、重启计算节点服务
[root@note1 ~]# systemctl restart openstack-nova-compute.service

6、启动Linuxbridge代理并配置它开机自启动
[root@Host1 ~]# systemctl start neutron-linuxbridge-agent.service
[root@Host1 ~]# systemctl enable neutron-linuxbridge-agent.service

7、控制节点测试:确认alive为笑脸存活
[root@openstack ~]# neutron agent-list
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+------------------+------------------+-----------+-------------------+-------+----------------+---------------------+
| id               | agent_type       | host      | availability_zone | alive | admin_state_up | binary              |
+------------------+------------------+-----------+-------------------+-------+----------------+---------------------+
| a0f301e2-2ed0-43 | DHCP agent       | openstack | nova              | :-)   | True           | neutron-dhcp-agent  |
| 42-a005-f71b9d23 |                  |           |                   |       |                |                     |
| e543             |                  |           |                   |       |                |                     |
| dc88fa1e-f79e-   | Metadata agent   | openstack |                   | :-)   | True           | neutron-metadata-   |
| 4fb7-89ed-       |                  |           |                   |       |                | agent               |
| 0dc592fb3058     |                  |           |                   |       |                |                     |
| f0416b07-78b5-4b | Linux bridge     | openstack |                   | :-)   | True           | neutron-            |
| 56-8ee1-1f5ab1ba | agent            |           |                   |       |                | linuxbridge-agent   |
| caa8             |                  |           |                   |       |                |                     |
+------------------+------------------+-----------+-------------------+-------+----------------+---------------------+
```