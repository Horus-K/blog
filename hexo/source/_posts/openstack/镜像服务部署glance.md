---
title: 镜像服务部署glance
date: 2020-03-09 16:11:43
tags: openstack
categories: openstack
---

Glance是OpenStack镜像服务组件，glance服务默认监听在9292端口，其接受REST API请求，然后通过其他模块（glance-registry及image store）来完成诸如镜像的获取、上传、删除等操作。Glance提供restful API可以查询虚拟机镜像的metadata，并且可以获得镜像，通过Glance，虚拟机镜像可以被存储到多种存储上，比如简单的文件存储或者 对象存储（OpenStack中swift项目）是在创建虚拟机的时候，需要先把镜像上传到glance，对镜像的列出镜像、删除镜像和上传镜像都是通过glance进行处理。glance有两个主要的服务，一个是glance-api接受镜像的删除和读取，一个是glance-Registry。

<!--more-->

glance-registy负责与mysql数据交互，用于存储或获取镜像的元数据（metadata），提供镜像元数据相关的REST接口，通过glance-registry可以向数据库中写入或获取镜像的各种数据，glance-registry监听的端口是9191，glance数据库中有两张表，一张是glance表，一张是imane property表，image表保存了镜像格式、大小等信息，image property表保存了镜像的定制化信息。

image store 是一个存储的接口层，通过这个接口glance可以获取镜像，image store支持的存储有Amazon的S3、本身的swift、还有ceph、glusterFS、sheepdog等分布式存储，image store是镜像保存与读取的接口，但是它只是一个接口，具体的实现需要外部的支持，glance不需要配置消息队列，但是需要配置数据库和keystone。

```
官方文档
https://docs.openstack.org/ocata/zh_CN/install-guide-rdo/glance-install.html
1、登录数据库 
[root@mysql1 ~]# mysql -uroot -p 

2、创建glance数据库 
MariaDB [(none)]> CREATE DATABASE glance;
 Query OK, 1 row affected (0.011 sec)  

3、创建glance账号并授权glance数据库 
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glance123';
 Query OK, 0 rows affected (0.016 sec) 
 4、测试是否可以使用glance连接数据库 
mysql -uglance -pglance123 -hopenstack-vip.qh.com
```

![img](镜像服务部署glance/glance1.png)

```
5、使用admin管理员账号创建glance账号

[root@OpenStack1 ~]# openstack user create --domain default --password-prompt glance
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | 463ba193c3164cf2b0d278198bb424cd |
| enabled             | True                             |
| id                  | ffab9b49c1b94aa3ad70a109f8c38140 |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

[root@OpenStack1 ~]# openstack user list
+----------------------------------+--------+
| ID                               | Name   |
+----------------------------------+--------+
| 28bcd9dc73be4443bc88f23b57c4d3b6 | demp   |
| 95e7ef69ef254533b0aa913b45381aa9 | admin  |
| ffab9b49c1b94aa3ad70a109f8c38140 | glance |
+----------------------------------+--------+

6、将glance用户加入service项目，并赋予admin角色的权限
[root@OpenStack1 ~]# openstack role add --project service --user glance admin

7、创建glance服务实体
[root@OpenStack1 ~]# openstack service create --name glance \
--description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 7e3ab1a609bf4944a8ffcf01c3ae3d7a |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+

查看创建服务实体
[root@OpenStack1 ~]# openstack service list
+----------------------------------+----------+----------+
| ID                               | Name     | Type     |
+----------------------------------+----------+----------+
| 3b1ba4d6a5f645fb883f86ce806ac087 | keystone | identity |
| 7e3ab1a609bf4944a8ffcf01c3ae3d7a | glance   | image    |
+----------------------------------+----------+----------+

8、注册glance服务实体API端点
openstack endpoint create --region RegionOne  image public http://openstack-vip.qh.com:9292 
openstack endpoint create --region RegionOne  image internal http://openstack-vip.qh.com:9292
openstack endpoint create --region RegionOne  image admin http://openstack-vip.qh.com:9292 

[root@OpenStack1 ~]# openstack endpoint create --region RegionOne  image public http://openstack-vip.qh.com:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 03005b03299a48608158e44365b079ba |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 7e3ab1a609bf4944a8ffcf01c3ae3d7a |
| service_name | glance                           |
| service_type | image                            |
| url          | http://openstack-vip.net:9292    |
+--------------+----------------------------------+
[root@OpenStack1 ~]# openstack endpoint create --region RegionOne  image internal http://openstack-vip.qh.com:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 05cca4d6ae5a4110aa6345284d7f7b7a |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 7e3ab1a609bf4944a8ffcf01c3ae3d7a |
| service_name | glance                           |
| service_type | image                            |
| url          | http://openstack-vip.net:9292    |
+--------------+----------------------------------+
[root@OpenStack1 ~]# openstack endpoint create --region RegionOne  image admin http://openstack-vip.qh.com:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 2c2de07fdbe04f3aa3f7e6113cf28b3f |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 7e3ab1a609bf4944a8ffcf01c3ae3d7a |
| service_name | glance                           |
| service_type | image                            |
| url          | http://openstack-vip.net:9292    |
+--------------+----------------------------------+
```

**查看注册结果**

```
# openstack endpoint list
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------------+
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                               |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------------+
| 03005b03299a48608158e44365b079ba | RegionOne | glance       | image        | True    | public    | http://openstack-vip.net:9292     |
| 05cca4d6ae5a4110aa6345284d7f7b7a | RegionOne | glance       | image        | True    | internal  | http://openstack-vip.net:9292     |
| 2c2de07fdbe04f3aa3f7e6113cf28b3f | RegionOne | glance       | image        | True    | admin     | http://openstack-vip.net:9292     |
| 3a5cc4f279f5482990e77261ba57e80f | RegionOne | keystone     | identity     | True    | internal  | http://openstack-vip.net:5000/v3  |
| c5649de3650e4fdeb262ab3acb71819c | RegionOne | keystone     | identity     | True    | public    | http://openstack-vip.net:5000/v3  |
| f3ac230871b2478f8e4ec1bfd471f90b | RegionOne | keystone     | identity     | True    | admin     | http://openstack-vip.net:35357/v3 |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------------+

=======================================================================
```

### **安装并配置组件**

```
1、安装软件包
[root@OpenStack1 ~]# yum install openstack-glance -y

2、编辑配置文件api
[root@OpenStack1 ~]# vim /etc/glance/glance-api.conf
[database]                    #配置数据库访问
connection = mysql+pymysql://glance:glance123@openstack-vip.qh.com/glance

[keystone_authtoken]          #配置认证服务访问
auth_uri = http:// openstack-vip.qh.com :5000
auth_url = http:// openstack-vip.qh.com :35357
memcached_servers =  openstack-vip.qh.com :11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance

[paste_deploy]
flavor = keystone


[glance_store]                 #配置本地文件系统存储和镜像文件位置，可使用NFS
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/


3、编辑配置文件registry
[root@OpenStack1 ~]# vim /etc/glance/glance-registry.conf
[database]                    #配置数据库访问
connection = mysql+pymysql://glance:glance123@openstack-vip.qh.com/glance

[keystone_authtoken]          #配置认证服务访问
auth_uri = http:// openstack-vip.qh.com :5000
auth_url = http:// openstack-vip.qh.com :35357
memcached_servers =  openstack-vip.qh.com :11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance

[paste_deploy]
flavor = keystone
```

### **4、初始化数据库，写入数据**

```
[root@OpenStack1 ~]# su -s /bin/sh -c "glance-manage db_sync" glance
```

![img](镜像服务部署glance/glance2.png)

![img](镜像服务部署glance/glance3.png)

```
5、配置haproxy
root@LB1:~# vim /etc/haproxy/haproxy.cfg 
listen openstack_glance_port_9292
  bind 10.20.0.40:9292
  mode tcp
  log global
  server 10.20.0.60 10.20.0.60:9292 check inter 3000 fall 2 rise 5
  server 10.20.0.70 10.20.0.70:9292 check inter 3000 fall 2 rise 5 backup
root@LB1:~# systemctl restart haproxy
 ss -nlt确认监听9292端口

6、启动镜像服务
[root@OpenStack1 ~]# systemctl start openstack-glance-api.service openstack-glance-registry.service
[root@OpenStack1 ~]# systemctl enable openstack-glance-api.service openstack-glance-registry.service

查看images目录是否正常生成
[root@OpenStack1 ~]# ll /var/lib/glance
total 0
drwxr-x--- 2 glance glance 6 Jun 21 09:38 images

7、在另一个服务器安装NFS服务，实现共享
[root@mysql2 ~]# yum -y install nfs-utils
[root@mysql2 ~]# mkdir /openstack/glance -p       #注意权限
[root@mysql2 ~]# vim /etc/exports
/openstack/glance *(rw,no_root_squash)
[root@mysql2 ~]# systemctl restart nfs
[root@mysql2 ~]# systemctl enable nfs
[root@mysql2 ~]# exportfs -r        #重新加载，无需重启服务

8、挂载
[root@OpenStack1 ~]# showmount -e 10.20.0.110
Export list for 10.20.0.110:
/openstack/glance *

[root@OpenStack1 ~]# mount -t nfs 10.20.0.110:/openstack/glance /var/lib/glance/images
[root@OpenStack1 ~]# vim /etc/fstab
10.20.0.110:/openstack/glance /var/lib/glance/images nfs defaults,_netdev 0 0

9、下载测试镜像
[root@OpenStack1 ~]# wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
[root@OpenStack1 ~]# systemctl restart openstack-glance-api.service openstack-glance-registry.service

使用QCOW2磁盘格式，bare容器格式上传镜像到镜像服务并设置公共可见，这样所有的项目都可以访问它

[root@OpenStack1 ~]# openstack image create "cirros-0.3.4" \
--file cirros-0.3.4-x86_64-disk.img \
--disk-format qcow2 --container-format bare \
--public
```

![img](镜像服务部署glance/glance4.png)

```
查看镜像
[root@OpenStack1 ~]# openstack image list
+--------------------------------------+--------------+--------+
| ID                                   | Name         | Status |
+--------------------------------------+--------------+--------+
| f5d00b1e-aa71-4c46-86c5-bf12d0bcd21f | cirros-0.3.5 | active |
+--------------------------------------+--------------+--------+

[root@OpenStack1 ~]# ll /var/lib/glance/images/
total 12960
-rw-r----- 1 glance glance 13267968 Jun 21 10:14 f5d00b1e-aa71-4c46-86c5-bf12d0bcd21f

同步数据
rsync -avlopg /openstack/glance/* 10.20.0.110:/openstack/glance/
```