---
title: ansible-role二进制批量部署mariadb
date: 2020-03-10 14:26:01
tags:
- ansible
- mysql
categories: ansible
---

## 总结构

![img](ansible-role二进制批量部署mariadb/c1f0bf54f5e0cf4587331e22812f6d20.png)

<!--more-->

### 1

![img](ansible-role二进制批量部署mariadb/f2d69972ef2ba98f253737dadd951422.png)

```
- include: group.yml
- include: user.yml
- include: mkdir.yml
- include: data.yml
- include: link.yml
- include: linkstate.yml
- include: conf.yml
- include: server.yml
```

### 2

![img](ansible-role二进制批量部署mariadb/82595d709bf9d6ccba3546fdf364b00a.png)

```
- name: create group
  group: name=mysql system=yes gid=306
```

### 3

![img](ansible-role二进制批量部署mariadb/b23f26c6ce68d880fb6fdba3d7d82e8c.png)

```
- name: create user
  user: name=mysql system=yes uid=306 group=306 home=/data/mysql
```

### 4

![img](ansible-role二进制批量部署mariadb/c628193bbe252a6daeaf23347099f1c5.png)

```
- name: mkdir
  file: name=/data/mysql owner=root group=mysql state=directory
```

### 5

![img](ansible-role二进制批量部署mariadb/df696a950abc9f6f72466cf926456b09.png)

```
- name: copy data
  unarchive: src=mariadb-10.2.23-linux-x86_64.tar.gz  dest=/usr/local
```

### 6

![img](ansible-role二进制批量部署mariadb/842e1dec7fb2b08120ecbbdcde9c2477.png)

```
- name: link
  file: src=/usr/local/mariadb-10.2.23-linux-x86_64 dest=/usr/local/mysql state=link
```

### 7

![img](ansible-role二进制批量部署mariadb/0b547aad2e8b61bd5181477e52ab5a47.png)

```
- name: link state
  file: name=/usr/local/mysql owner=root group=mysql
```

### 8

![img](ansible-role二进制批量部署mariadb/b8a7bd8b680b58e033c2f32280a1e279.png)

```
- name: mkdir conf 
  file: path=/etc/mysql state=directory
- name: copy conf
  template: src=my-large.cnf.j2 dest=/etc/mysql/my.cnf
```

### 9

![img](ansible-role二进制批量部署mariadb/ffd24e2bbcacaf1e2d7cbfe3491164d5.png)

```
- name: install
  shell: /usr/local/mysql/scripts/mysql_install_db --datadir=/data/mysql --user=mysql
- name: copy service conf
  template: src=mysql.server.j2 dest=/etc/rc.d/init.d/mysqld
- name: add service
  shell: chkconfig --add mysqld
- name: chmod
  shell: chmod +x /etc/init.d/mysqld
- name: start server
  shell: service mysqld start
- name: PATH
  shell: echo PATH=/usr/local/mysql/bin:$PATH > /etc/profile.d/mysql
- name: source
  shell: source /etc/profile.d/mysql
```

### 10 安全初始化

```
#!/usr/bin/expect
set timeout 60
#set password [lindex $argv 0]
spawn mysql_secure_installation
expect {
        "enter for none" { send "\r"; exp_continue}
        "Change the root password" { send "\r"; exp_continue}
        "New password" { send "new passwd\r"; exp_continue}
        "Re-enter new password" { send "newpasswd\r"; exp_continue}
        "Remove anonymous users" { send "\r"; exp_continue}
        "Disallow root login remotely" { send "n\r"; exp_continue}
        "Remove test database and access to it" { send "\r"; exp_continue}
        "Reload privilege tables now" { send "\r"; exp_continue}
        "Cleaning up" { send "\r"}
}
interact ' > mysql_secure_installation.exp
```