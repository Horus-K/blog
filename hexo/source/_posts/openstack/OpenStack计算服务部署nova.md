---
title: OpenStack计算服务部署nova
date: 2020-03-09 16:10:48
tags: openstack
categories: openstack
---

nova是OpenStack最早的组件之一，nova分为控制节点和计算节点，计算节点通过nova computer进行虚拟机创建，通过libvirt调用kvm创建虚拟机，nova之间通信通过rabbitMQ队列进行通信。

<!--more-->

- **API：**负责接收和响应外部请求。

- **Scheduler**：负责调度虚拟机所在的物理机。

- **Conductor**：计算节点访问数据库的中间件。

- **Consoleauth：**用于控制台的授权认证。

- **Novncproxy：**VNC代理，用于显示虚拟机终端操作。

**Nova-API的功能：** Nova-API组件实现了restful API的功能，接收和响应来自最终用户的计算API请求，接收外部的请求并通过message queue将请求发送给其他服务组件，同时也兼容EC2 API，所以也可以使用EC2的管理工具对nova进行日常管理。

**nova scheduler:** nova scheduler模式在OpenStack中的作用是决策虚拟机创建在哪个主机（计算节点）上。决策一个虚拟机应该调度到某物理节点，需要分为两个步骤：

过滤（filter），过滤出可以创建虚拟机的主机

![img](OpenStack计算服务部署nova/nova1.png)

计算权值（weight），根据权重大进行分配，默认根据资源可用空间进行权重排序。

![img](OpenStack计算服务部署nova/nova2.png)

**=============================================================================================**

```
nova控制端服务及高可用

  安装并配置nova控制节点

1、创建数据库
[root@mysql1 ~]# mysql -uroot -p
MariaDB [(none)]> CREATE DATABASE nova_api;
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> CREATE DATABASE nova_cell0;

2、创建nova账户并对nova数据库进行授权
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'nova123';


MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'nova123';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'nova123';
3、测试
```

![img](OpenStack计算服务部署nova/nova3.png)

```
4、在管理端使用admin管理员账号创建nova用户
[root@OpenStack1 ~]# openstack user create --domain default --password-prompt nova
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | 463ba193c3164cf2b0d278198bb424cd |
| enabled             | True                             |
| id                  | cc642044eabd438b9a67ef9548a5a5a3 |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

[root@OpenStack1 ~]# openstack user list
+----------------------------------+--------+
| ID                               | Name   |
+----------------------------------+--------+
| 28bcd9dc73be4443bc88f23b57c4d3b6 | demp   |
| 95e7ef69ef254533b0aa913b45381aa9 | admin  |
| cc642044eabd438b9a67ef9548a5a5a3 | nova   |
| ffab9b49c1b94aa3ad70a109f8c38140 | glance |
+----------------------------------+--------+

5、将nova用户加入service项目，并赋予admin角色的权限
[root@OpenStack1 ~]# openstack role add --project service --user nova admin

6、创建nova服务实体
[root@OpenStack1 ~]# openstack service create --name nova \
--description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | cccd4f84faf74cdeb5e8493a3573e0b2 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+

查看创建的服务实体
[root@OpenStack1 ~]# openstack service list
+----------------------------------+----------+----------+
| ID                               | Name     | Type     |
+----------------------------------+----------+----------+
| 3b1ba4d6a5f645fb883f86ce806ac087 | keystone | identity |
| 7e3ab1a609bf4944a8ffcf01c3ae3d7a | glance   | image    |
| cccd4f84faf74cdeb5e8493a3573e0b2 | nova     | compute  |
+----------------------------------+----------+----------+

7、注册nova服务实体API端点
openstack endpoint create --region RegionOne compute public http://openstack-vip.qh.com:8774/v2.1 
openstack endpoint create --region RegionOne compute internal http://openstack-vip.qh.com:8774/v2.1 
openstack endpoint create --region RegionOne compute admin http://openstack-vip.qh.com:8774/v2.1 



[root@OpenStack1 ~]# openstack endpoint create --region RegionOne compute public http://openstack-vip.qh.com:8774/v2.1
+--------------+------------------------------------+
| Field        | Value                              |
+--------------+------------------------------------+
| enabled      | True                               |
| id           | 85630d2827794726abd51ba8f6c39f76   |
| interface    | public                             |
| region       | RegionOne                          |
| region_id    | RegionOne                          |
| service_id   | cccd4f84faf74cdeb5e8493a3573e0b2   |
| service_name | nova                               |
| service_type | compute                            |
| url          | http://openstack-vip.net:8774/v2.1 |
+--------------+------------------------------------+

[root@OpenStack1 ~]# openstack endpoint create --region RegionOne compute internal http://openstack-vip.qh.com:8774/v2.1
+--------------+------------------------------------+
| Field        | Value                              |
+--------------+------------------------------------+
| enabled      | True                               |
| id           | d8ae3de7f8cc4ae3bc9169831ecb1e04   |
| interface    | internal                           |
| region       | RegionOne                          |
| region_id    | RegionOne                          | 
| service_id   | cccd4f84faf74cdeb5e8493a3573e0b2   |
| service_name | nova                               |
| service_type | compute                            |
| url          | http://openstack-vip.net:8774/v2.1 |
+--------------+------------------------------------+

[root@OpenStack1 ~]# openstack endpoint create --region RegionOne compute admin http://openstack-vip.qh.com:8774/v2.1
+--------------+------------------------------------+
| Field        | Value                              |
+--------------+------------------------------------+
| enabled      | True                               |
| id           | e218e523fd61492f953f3b37f1c3ddc9   |
| interface    | admin                              |
| region       | RegionOne                          |
| region_id    | RegionOne                          |
| service_id   | cccd4f84faf74cdeb5e8493a3573e0b2   |
| service_name | nova                               |
| service_type | compute                            |
| url          | http://openstack-vip.net:8774/v2.1 |
+--------------+------------------------------------+
8、验证注册
[root@OpenStack1 ~]# openstack endpoint list
+----------------------------------+-----------+--------------+--------------+---------+-----------+------------------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                                |
+----------------------------------+-----------+--------------+--------------+---------+-----------+------------------------------------+
| 03005b03299a48608158e44365b079ba | RegionOne | glance       | image        | True    | public    | http://openstack-vip.net:9292      |
| 05cca4d6ae5a4110aa6345284d7f7b7a | RegionOne | glance       | image        | True    | internal  | http://openstack-vip.net:9292      |
| 2c2de07fdbe04f3aa3f7e6113cf28b3f | RegionOne | glance       | image        | True    | admin     | http://openstack-vip.net:9292      |
| 3a5cc4f279f5482990e77261ba57e80f | RegionOne | keystone     | identity     | True    | internal  | http://openstack-vip.net:5000/v3   |
| 85630d2827794726abd51ba8f6c39f76 | RegionOne | nova         | compute      | True    | public    | http://openstack-vip.net:8774/v2.1 |
| c5649de3650e4fdeb262ab3acb71819c | RegionOne | keystone     | identity     | True    | public    | http://openstack-vip.net:5000/v3   |
| d8ae3de7f8cc4ae3bc9169831ecb1e04 | RegionOne | nova         | compute      | True    | internal  | http://openstack-vip.net:8774/v2.1 |
| e218e523fd61492f953f3b37f1c3ddc9 | RegionOne | nova         | compute      | True    | admin     | http://openstack-vip.net:8774/v2.1 |
| f3ac230871b2478f8e4ec1bfd471f90b | RegionOne | keystone     | identity     | True    | admin     | http://openstack-vip.net:35357/v3  |
+----------------------------------+-----------+--------------+--------------+---------+-----------+------------------------------------+
9、创建placement用户
[root@OpenStack1 ~]# openstack user create --domain default --password-prompt placement
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | 463ba193c3164cf2b0d278198bb424cd |
| enabled             | True                             |
| id                  | a59dc6359de040d7a1adea4d2f60f257 |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

[root@OpenStack1 ~]# openstack user list
+----------------------------------+-----------+
| ID                               | Name      |
+----------------------------------+-----------+
| 28bcd9dc73be4443bc88f23b57c4d3b6 | demp      |
| 95e7ef69ef254533b0aa913b45381aa9 | admin     |
| a59dc6359de040d7a1adea4d2f60f257 | placement |
| cc642044eabd438b9a67ef9548a5a5a3 | nova      |
| ffab9b49c1b94aa3ad70a109f8c38140 | glance    |
+----------------------------------+-----------+

10、将placement用户加入service项目，并赋予admin角色的权限
[root@OpenStack1 ~]# openstack role add --project service --user placement admin

11、创建placement服务
[root@OpenStack1 ~]# openstack service create --name placement --description "Placement API" placement
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Placement API                    |
| enabled     | True                             |
| id          | 329ea6dc9e7b4cc8b5797f5c0d7f5d88 |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+

12、注册placement API服务
openstack endpoint create --region RegionOne placement public http://openstack-vip.qh.com:8778 
openstack endpoint create --region RegionOne placement internal http://openstack-vip.qh.com:8778 
openstack endpoint create --region RegionOne placement admin http://openstack-vip.qh.com:8778 


[root@OpenStack1 ~]# openstack endpoint create --region RegionOne placement public http:// openstack-vip.qh.com :8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 4d392925b4b54c56ac96d4563fe6a7a7 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 329ea6dc9e7b4cc8b5797f5c0d7f5d88 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://openstack-vip.net:8778    |
+--------------+----------------------------------+

[root@OpenStack1 ~]# openstack endpoint create --region RegionOne placement internal http:// openstack-vip.qh.com :8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | da1ed4a922ef43f8b4484cb1ea547e6b |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 329ea6dc9e7b4cc8b5797f5c0d7f5d88 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://openstack-vip.net:8778    |
+--------------+----------------------------------+

[root@OpenStack1 ~]# openstack endpoint create --region RegionOne placement admin http:// openstack-vip.qh.com :8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | f31dff31de284ba38fec69b00bf48e5e |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 329ea6dc9e7b4cc8b5797f5c0d7f5d88 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://openstack-vip.net:8778    |
+--------------+----------------------------------+
13、配置负载均衡
root@LB1:~# vim /etc/haproxy/haproxy.cfg 
listen openstack_nova_port_8774
  bind 10.20.0.40:8774
  mode tcp
  log global
  server 10.20.0.60 10.20.0.60:8774 check inter 3000 fall 2 rise 5
  server 10.20.0.70 10.20.0.70:8774 check inter 3000 fall 2 rise 5 backup

listen openstack_nova_port_8778
  bind 10.20.0.40:8778
  mode tcp
  log global
  server 10.20.0.60 10.20.0.60:8778 check inter 3000 fall 2 rise 5
  server 10.20.0.70 10.20.0.70:8778 check inter 3000 fall 2 rise 5 backup

listen openstack_nova_port_6080
  bind 10.20.0.40:6080
  mode tcp
  log global
  server 10.20.0.60 10.20.0.60:6080 check inter 3000 fall 2 rise 5
  server 10.20.0.70 10.20.0.70:6080 check inter 3000 fall 2 rise 5 backup


root@LB1:~# systemctl restart haproxy
root@LB1:~# ss -nlt          #确认监听8774、8778、6080端口
14、在控制端，安装并配置组件
配置yum源，
[centos-openstack-ocata]
name=CentOS-7 - OpenStack ocata
#baseurl=http://mirror.centos.org/centos/7/cloud/$basearch/openstack-ocata/
#baseurl=https://mirrors.aliyun.com/centos/7/cloud/x86_64/openstack-ocata/
baseurl=https://mirrors.aliyun.com/centos/7/cloud/x86_64/openstack-stein/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Cloud
exclude=sip,PyQt4

[root@OpenStack1 ~]# yum install openstack-nova-api openstack-nova-conductor openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler openstack-nova-placement-api -y
编辑配置文件nova.conf
[root@OpenStack1 ~]# vim /etc/nova/nova.conf
[DEFAULT]    
enabled_apis = osapi_compute,metadata          #只启用计算和元数据API
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
transport_url = rabbit://openstack:openstack123@rabbitmq.qh.com

[api_database]                          #配置数据库连接
connection = mysql+pymysql://nova:nova123@openstack-vip.qh.com/nova_api

[database]                              #配置数据库连接
connection=mysql+pymysql://nova:nova123@openstack-vip.qh.com/nova

[api]
auth_strategy=keystone

[keystone_authtoken]                     #认证
auth_uri = http://openstack-vip.qh.com:5000
auth_url = http://openstack-vip.qh.com:35357
memcached_servers = openstack-vip.qh.com:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = nova

[vnc]                             # 配置VNC代理使用控制节点的管理接口IP地址
enabled = true
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address =  openstack-vip.qh.com 

[glance]                          #配置镜像服务 API 的位置
api_servers = http://openstack-vip.qh.com:9292

[oslo_concurrency]                #配置锁路径
lock_path = /var/lib/nova/tmp

[placement]
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://openstack-vip.qh.com:35357/v3
username = placement
password = placement
编辑配置文件00-nova-placement-api.conf
[root@OpenStack1 ~]# vim /etc/httpd/conf.d/00-nova-placement-api.conf
<Directory /usr/bin>              #在底部加入
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>

[root@OpenStack1 ~]# systemctl restart httpd
```

![img](OpenStack计算服务部署nova/image-119.png)

```
15、初始化数据库
nova-api database
[root@OpenStack1 ~]# su -s /bin/sh -c "nova-manage api_db sync" nova

cell0 database
[root@OpenStack1 ~]# su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova

[root@OpenStack1 ~]# su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova

[root@OpenStack1 ~]# su -s /bin/sh -c "nova-manage db sync" nova

验证
[root@OpenStack1 ~]# nova-manage cell_v2 list_cells

16、启动服务
[root@OpenStack1 ~]# systemctl enable openstack-nova-api.service \
openstack-nova-consoleauth.service openstack-nova-scheduler.service \
openstack-nova-conductor.service openstack-nova-novncproxy.service

[root@OpenStack1 ~]# systemctl start openstack-nova-api.service \
openstack-nova-consoleauth.service openstack-nova-scheduler.service \
openstack-nova-conductor.service openstack-nova-novncproxy.service
```

**=============================================================**

## nova计算节点服务

```
  在计算节点服务器部署
安装nova计算服务
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo   #安装阿里最新base源 
yum update

 /etc/yum.repos.d/CentOS-Base.repo文件里添加如下内容 
[Virt]
name=CentOS-$releasever - Base
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra
baseurl=http://mirrors.sohu.com/centos/7.5.1804/virt/x86_64/kvm-common/
#baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[root@note1 ~]# yum install openstack-nova-compute 


  
修改配置文件
[root@note1 ~]# vim /etc/nova/nova.conf 
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:openstack123@rabbitmq.qh.com
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api]
auth_strategy=keystone

[keystone_authtoken]
auth_uri = http://openstack-vip.qh.com:5000
auth_url = http://openstack-vip.qh.com:35357
memcached_servers = openstack-vip.qh.com:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = nova

[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = node1(本机地址)
novncproxy_base_url = http://openstack-vip.qh.com:6080/vnc_auto.html

[glance]
api_servers = http://openstack-vip.qh.com:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://openstack-vip.qh.com:35357/v3
username = placement
password = placement

确认你的计算节点是否支持虚拟机硬件加速
[root@note1 ~]# egrep -c '(vmx|svm)' /proc/cpuinfo

启动服务
[root@note1 ~]# systemctl enable libvirtd.service openstack-nova-compute.service
[root@note1 ~]# systemctl start libvirtd.service openstack-nova-compute.service

=================================================================================

在控制节点上测试
[root@OpenStack1 ~]# openstack hypervisor list
+----+---------------------+-----------------+--------------+-------+
| ID | Hypervisor Hostname | Hypervisor Type | Host IP      | State |
+----+---------------------+-----------------+--------------+-------+
| 1  | note1               | QEMU            | 10.20.0.80   | up    |
+----+---------------------+-----------------+--------------+-------
  定期主动发现计算节点
[root@OpenStack1 ~]# vim /etc/nova/nova.conf
[scheduler]
discover_hosts_in_cells_interval = 300

验证计算节点
[root@OpenStack1 ~]# nova host-list
[root@OpenStack1 ~]# nova service-list
[root@OpenStack1 ~]# nova image-list

列出服务组件是否注册成功
[root@OpenStack1 ~]# openstack compute service list
手动发现计算节点
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
检查cells和placement API是否工作正常
[root@OpenStack1 ~]# nova-status upgrade check

列出keystone服务中的端点
[root@OpenStack1 ~]# openstack catalog list
```

![img](OpenStack计算服务部署nova/image-118.png)