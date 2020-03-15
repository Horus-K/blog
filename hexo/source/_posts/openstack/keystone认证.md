---
title: keystone认证
date: 2020-03-09 16:12:41
tags: openstack
categories: openstack
---

## 先决条件

<!--more-->

```
用数据库连接客户端以 root 用户连接到数据库服务器：

$ mysql -u root -p
创建 keystone 数据库：

MariaDB [(none)]> CREATE DATABASE keystone;
对``keystone``数据库授予恰当的权限：

MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'keystone123';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'keystone123';
```

在管理节点安装mysq客户端

```
yum install mysql -y
```

在 管理节点安装包

```
运行以下命令来安装包。
# yum install openstack-keystone httpd mod_wsgi
```

编辑文件 /etc/keystone/keystone.conf 并完成如下动作：

```
在 [database] 部分，配置数据库访问：
[database]
# ...
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
+hosts
192.168.220.248 openstack-vip.qh.net
connection = mysql+pymysql://keystone:123456@openstack-vip.qh.net/keystone
```

在`[token]`部分，配置Fernet UUID令牌的提供者。

```
[token]
# ...
provider = fernet
```

![img](keystone认证/screenshot_20190619_172135.png)

管理端[DEFAULT]生成admin token

```
[root@controller1 ~]#openssl rand -hex 10
2587bdc190c8de5235ea
```

![img](keystone认证/screenshot_20190619_173625.png)

验证

```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

![img](keystone认证/screenshot_20190619_174709.png)

![img](keystone认证/screenshot_20190619_190055.png)

初始化Fernet key：

```
# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

![img](keystone认证/screenshot_20190619_200004.png)

![img](keystone认证/screenshot_20190619_190347.png)

## 配置 Apache HTTP 服务器

```
创建一个链接到``/usr/share/keystone/wsgi-keystone.conf``文件

# ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
启动 Apache HTTP 服务并配置其随系统启动：

# systemctl enable httpd.service
# systemctl start httpd.service
```

![img](keystone认证/screenshot_20190619_192229.png)有355357

开启一个新tty窗口，开始测试和创建用户

```
export OS_TOKEN=2587bdc190c8de5235ea
export OS_URL=http://192.168.220.202:35357/v3
export OS_IDENTITY_API_VERSION=3
```

![img](keystone认证/screenshot_20190619_195825.png)

创建默认域：

```
[root@controller2 ~]#openstack domain create --description "Default Domain" default
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Default Domain                   |
| enabled     | True                             |
| id          | f9a9c4f4cd2647e8a5d6fe012ec07788 |
| name        | default                          |
+-------------+----------------------------------+
[root@controller2 ~]#openstack domain list
+----------------------------------+---------+---------+----------------+
| ID                               | Name    | Enabled | Description    |
+----------------------------------+---------+---------+----------------+
| f9a9c4f4cd2647e8a5d6fe012ec07788 | default | True    | Default Domain |
+----------------------------------+---------+---------+----------------+

[root@controller2 ~]#openstack domain delete f9a9c4f4cd2647e8a5d6fe012ec07788  #删除域
```

## 创建一个 admin 的项目

```
openstack project create --domain default --description "Admin Project" admin
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Admin Project                    |
| domain_id   | f9a9c4f4cd2647e8a5d6fe012ec07788 |
| enabled     | True                             |
| id          | 1f04e20d276c405bb1fd49d1bf09946e |
| is_domain   | False                            |
| name        | admin                            |
| parent_id   | f9a9c4f4cd2647e8a5d6fe012ec07788 |
+-------------+----------------------------------+
```

创建 admin 用户并设置密码为 admin：

```
openstack user create --domain default --password-prompt admin
User Password:admin
Repeat User Password:admin
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | f9a9c4f4cd2647e8a5d6fe012ec07788 |
| enabled             | True                             |
| id                  | 8f7d37d54641497889f23eb3b0f2eed8 |
| name                | admin                            |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
[root@controller2 ~]#openstack user list
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| 8f7d37d54641497889f23eb3b0f2eed8 | admin |
+----------------------------------+-------+
```

创建 admin 角色：
一个项目里面可以有多个角色，目前角色只能创建在/etc/keystone/policy.json 文件中定义好的角色：

![img](keystone认证/screenshot_20190619_201758.png)

```
[root@controller2 ~]#openstack role create admin
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 8e3790e764544b19ad89852948c9e077 |
| name      | admin                            |
+-----------+----------------------------------+
[root@controller2 ~]#openstack role list
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| 8e3790e764544b19ad89852948c9e077 | admin |
+----------------------------------+-------+
```

给 admin 用户授权：

将 admin 用户授予 admin 项目的 admin 角色，即给 admin 项目添加一个用户叫 admin，并将其添加至 admin 角色，角色是权限的一种集合：

```
[root@controller2 ~]#openstack role add --project admin --user admin admin
```

## 创建 o demo 项目：

该项目可用于演示和测试等



```
[root@controller2 ~]#openstack project create --domain default --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | f9a9c4f4cd2647e8a5d6fe012ec07788 |
| enabled     | True                             |
| id          | 389fbbfa451742dbb4989f6f4e857aed |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | f9a9c4f4cd2647e8a5d6fe012ec07788 |
+-------------+----------------------------------+
```

创建 demo 用户并设置密码为 demo

```
[root@controller2 ~]#openstack user create --domain default --password-prompt demo
User Password: demo
Repeat User Password:demo
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | f9a9c4f4cd2647e8a5d6fe012ec07788 |
| enabled             | True                             |
| id                  | 1b4610763476402c9195650b54b9b551 |
| name                | demo                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

创建一个 user 角色：

角色目前有 user 和 admin：

```
[root@controller2 ~]#openstack role create user
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 1d5ed3e1be4f4f15acc67d1040cccc55 |
| name      | user                             |
+-----------+----------------------------------+
```

把 demo 用户添加到 demo 项目：然后赋予 user 权限：

```
[root@controller2 ~]#openstack role add --project demo --user demo user
```

## 创建一个 service 项目：

各服务之间与 keystone 进行访问和认证，service 用于给服务创建用户

创建 service 项目：

```
[root@controller2 ~]#openstack project create --domain default --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | f9a9c4f4cd2647e8a5d6fe012ec07788 |
| enabled     | True                             |
| id          | cac1d2794af34a9990ea8ede036e1250 |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | f9a9c4f4cd2647e8a5d6fe012ec07788 |
+-------------+----------------------------------+
```

创建 glance 用户并设置密码为 glance

```
[root@controller2 ~]#openstack user create --domain default --password-prompt glance
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | f9a9c4f4cd2647e8a5d6fe012ec07788 |
| enabled             | True                             |
| id                  | b44be253b32c4c01b9d6529e01391be0 |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

对 glance 用户授权：把 glance 和 neutron 用户添加到 service 项目并授予 admin 角色

```
[root@controller2 ~]#openstack role add --project service --user glance admin
```

## 按照以上 步骤操作 a nova 和 neutron 用户：

将 nova 用户添加到 service 项目并授予 admin 权限

创建 nova 用户并设置 密码为 为 nova

```
[root@controller2 ~]#openstack user create --domain default --password-prompt nova
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | f9a9c4f4cd2647e8a5d6fe012ec07788 |
| enabled             | True                             |
| id                  | 7be49e0adbd24743836b022f99d7b4f3 |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

创建 neutron 用户并 设置 密码为 neutron ：

```
[root@controller2 ~]#openstack user create --domain default --password-prompt neutron
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | f9a9c4f4cd2647e8a5d6fe012ec07788 |
| enabled             | True                             |
| id                  | 55cd53a5f8cd4027a677074aa299e85f |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

将 nova 和 neutron 用户授权为 service 项目的 admi 权限

```
[root@controller2 ~]#openstack role add --project service --user nova admin
[root@controller2 ~]#openstack role add --project service --user neutron admin
```

## 服务注册：

![img](keystone认证/screenshot_20190619_203539.png)查看当前的服务

将 keystone 服务地址注册到 openstack：

创建一个 keystone 认证服务：

```
[root@controller2 ~]#openstack service create --name keystone --description "OpenStack Identity" identity
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Identity               |
| enabled     | True                             |
| id          | 69db4257102b4f55b17166b40a386f76 |
| name        | keystone                         |
| type        | identity                         |
+-------------+----------------------------------+
[root@controller2 ~]#openstack service list
+----------------------------------+----------+----------+
| ID                               | Name     | Type     |
+----------------------------------+----------+----------+
| 69db4257102b4f55b17166b40a386f76 | keystone | identity |
+----------------------------------+----------+----------+
[root@controller2 ~]#cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.220.201 controller1.qh.net
192.168.220.202 controller2.qh.net
192.168.50.248 openstack-vip.qh.net  ###
openstack endpoint create --region RegionOne identity public http://openstack-vip.qh.net:5000/v3  #公共端点

openstack endpoint create --region RegionOne identity internal http://openstack-vip.qh.net:5000/v3 #私有端点

openstack endpoint create --region RegionOne identity admin http://openstack-vip.qh.net:35357/v3 #管理端点


####替换成vip
[root@controller2 ~]#openstack endpoint create --region RegionOne identity public http://openstack-vip.qh.net:5000/v3
+--------------+-------------------------------------+
| Field        | Value                               |
+--------------+-------------------------------------+
| enabled      | True                                |
| id           | 17c797c8b7604b84ad0bf9a887652a24    |
| interface    | public                              |
| region       | RegionOne                           |
| region_id    | RegionOne                           |
| service_id   | 69db4257102b4f55b17166b40a386f76    |
| service_name | keystone                            |
| service_type | identity                            |
| url          | http://openstack-vip.qh.net:5000/v3 |
+--------------+-------------------------------------+
[root@controller2 ~]#openstack endpoint create --region RegionOne identity internal http://openstack-vip.qh.net:5000/v3
+--------------+-------------------------------------+
| Field        | Value                               |
+--------------+-------------------------------------+
| enabled      | True                                |
| id           | 413681a81dac4769809d8fe4ab009261    |
| interface    | internal                            |
| region       | RegionOne                           |
| region_id    | RegionOne                           |
| service_id   | 69db4257102b4f55b17166b40a386f76    |
| service_name | keystone                            |
| service_type | identity                            |
| url          | http://openstack-vip.qh.net:5000/v3 |
+--------------+-------------------------------------+
[root@controller2 ~]#openstack endpoint create --region RegionOne identity admin http://openstack-vip.qh.net:35357/v3
+--------------+--------------------------------------+
| Field        | Value                                |
+--------------+--------------------------------------+
| enabled      | True                                 |
| id           | fe66690d5a224a85ae07ae3a00db303a     |
| interface    | admin                                |
| region       | RegionOne                            |
| region_id    | RegionOne                            |
| service_id   | 69db4257102b4f55b17166b40a386f76     |
| service_name | keystone                             |
| service_type | identity                             |
| url          | http://openstack-vip.qh.net:35357/v3 |
+--------------+--------------------------------------+
```

验证

```
[root@controller2 ~]#openstack endpoint list
+-------------------+-----------+--------------+--------------+---------+-----------+--------------------+
| ID                | Region    | Service Name | Service Type | Enabled | Interface | URL                |
+-------------------+-----------+--------------+--------------+---------+-----------+--------------------+
| 17c797c8b7604b84a | RegionOne | keystone     | identity     | True    | public    | http://openstack-  |
| d0bf9a887652a24   |           |              |              |         |           | vip.qh.net:5000/v3 |
| 413681a81dac47698 | RegionOne | keystone     | identity     | True    | internal  | http://openstack-  |
| 09d8fe4ab009261   |           |              |              |         |           | vip.qh.net:5000/v3 |
| fe66690d5a224a85a | RegionOne | keystone     | identity     | True    | admin     | http://openstack-v |
| e07ae3a00db303a   |           |              |              |         |           | ip.qh.net:35357/v3 |
+-------------------+-----------+--------------+--------------+---------+-----------+--------------------+
```

配置haproxy代理管理ip

```
vim /etc/haproxy/haproxy.cfg

listen openstack_keystone_5000
        bind 192.168.220.248:5000
        mode tcp
        balance roundrobin
        server web1 192.168.50.202:5000 weight 1 check port 3306 inter 3s fall 3 rise 5

listen openstack_keystone_35357
        bind 192.168.220.248:35357
        mode tcp
        balance roundrobin
        server web1 192.168.50.202:35357 weight 1 check port 3306 inter 3s fall 3 rise 5
```

![img](keystone认证/screenshot_20190619_210114.png)

测试

![img](keystone认证/screenshot_20190619_210152.png)

## 测试 keystone 是否可以做用户验证：

验证 admin 用户，密码 admin，新打开一个窗口并进行以下操作：

```
export OS_IDENTITY_API_VERSION=3
[root@controller2 ~]#keystone-manage db_sync
[root@controller2 ~]#openstack --os-auth-url http://192.168.220.202:35357/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
Password: admin
+------------+-------------------------------------------------------------------------------------------+
| Field      | Value                                                                                     |
+------------+-------------------------------------------------------------------------------------------+
| expires    | 2019-06-19T14:17:29+0000                                                                  |
| id         | gAAAAABdCjXqL1Le5-YQi9N6FYEz3ovx5CRJp5aoHRNyeVNQ1UjshmX2QXIt-                             |
|            | hzu0CRdJjSclN9xT08DStgHdSbi_RQJv3bh0ryUm_thzF1GUmS8Ta8BL-                                 |
|            | tgC4ZkBlqMjOn1v8hxYURWAs5dAkvELWFLe56NIhWNgeuLD4-_R8ao2LYqldka0u4                         |
| project_id | 1f04e20d276c405bb1fd49d1bf09946e                                                          |
| user_id    | 8f7d37d54641497889f23eb3b0f2eed8                                                          |
+------------+-------------------------------------------------------------------------------------------+
[root@controller2 ~]#openstack --os-auth-url http://192.168.220.202:35357/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name demo --os-username demo token issue
Password: 
+------------+-------------------------------------------------------------------------------------------+
| Field      | Value                                                                                     |
+------------+-------------------------------------------------------------------------------------------+
| expires    | 2019-06-19T14:20:21+0000                                                                  |
| id         | gAAAAABdCjaV3R7LbNyRV7ltTRR8c6nHrLgOdqlMQKPScfoCvTo_O6LZ7do_q8u3R5qc0rR5AAEZbEPrG6j4x52QH |
|            | RZMTQgH9oRMddjvw5-IYGMUxwuGHfWnIu2HyfLj2zq_kJEpnQFzUcJmMn5QzF5XjAAnQtFB3CJX8qiXGOX0GNN6f7 |
|            | JiLuQ                                                                                     |
| project_id | 389fbbfa451742dbb4989f6f4e857aed                                                          |
| user_id    | 1b4610763476402c9195650b54b9b551                                                          |
+------------+-------------------------------------------------------------------------------------------+
```

## 使用脚本设置环境变量：

Admin 用户脚本内容：

```
[root@linux-host1 ~]# chmod a+x admin-ocata.sh
[root@controller2 ~]#source admin-ocata.sh
[root@linux-host1 ~]# cat admin-ocata.sh
#!/bin/bash
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_AUTH_URL=http://192.168.220.202:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
Demo 用户脚本内容：
[root@linux-host1 ~]# chmod a+x demo-ocata.sh
[root@controller2 ~]#source demo-ocata.sh
[root@linux-host1 ~]# cat demo-ocata.sh
#!/bin/bash
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=demo
export OS_AUTH_URL=http://192.168.220.202:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
[root@controller2 ~]#openstack --os-auth-url http://192.168.220.202:35357/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
+------------+-------------------------------------------------------------------------------------------+
| Field      | Value                                                                                     |
+------------+-------------------------------------------------------------------------------------------+
| expires    | 2019-06-19T14:26:16+0000                                                                  |
| id         | gAAAAABdCjf4xSv7V-zQyj9eu7uIvDGw-5JwHtLHoV9vjhWLKyVfKrJT8nu0zh9fiOA-sZ8aXSccPjIZPukiwh5-4 |
|            | OE3pEtR7EpX440Klo3_cD6fNhIPtk8qxIglODhXa6XIjvDoVFCo5jsiSO9RHMe6y_H1Xw93ZRUBtqIPrB7qISMlZH |
|            | IeqAg                                                                                     |
| project_id | 1f04e20d276c405bb1fd49d1bf09946e                                                          |
| user_id    | 8f7d37d54641497889f23eb3b0f2eed8                                                          |
+------------+-------------------------------------------------------------------------------------------+
###无需输入密码验证
```