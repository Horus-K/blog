---
title: 搭建PXE无人值守自动化安装系统，支持多系统
date: 2020-03-10 16:38:01
tags:
- PXE
categories: 自动化
---

## 需要的服务

<!--more-->

## 1httpd

```
[root@localhost ~]# yum install -y httpd
systemctl start httpd 
systemctl enable httpd
```

## 2 DHCP

```
cp /usr/share/doc/dhcp-4.2.5/dhcpd.conf.example /etc/dhcp/dhcpd.conf

vim  /etc/dhcp/dhcpd.conf
option domain-name "QH.org";
option domain-name-servers 8.8.8.8, 114.114.114.114;
default-lease-time 86400;
max-lease-time 864000;
subnet 192.168.64.0 netmask 255.255.255.0 {
  range 192.168.64.200 192.168.64.250;
  option routers 192.168.64.254;
  next-server 192.168.64.132;
  filename "pxelinux.0"
}
[root@localhost ~]# systemctl start dhcpd
systemctl enable dhcpd
```

## 3 tftp

```
yum install tftp-server
systemctl start tftp
systemctl enable  tftp
```

## 4光盘挂载

```
[root@localhost ~]# mkdir /var/www/html/centos/{6,7}/os/x86_64/ -pv
```

> /dev/sr1 11G 11G 0 100% /var/www/html/centos/7/os/x86_64
> /dev/sr0 3.8G 3.8G 0 100% /var/www/html/centos/6/os/x86_64

## 5准备启动文件

```
[root@localhost network-scripts]# yum install syslinux
[root@localhost network-scripts]# mkdir /var/lib/tftpboot/kernel{6,7}
[root@localhost network-scripts]# cp /var/www/html/centos/6/os/x86_64/isolinux/vmlinuz /var/lib/tftpboot/kernel6/
[root@localhost network-scripts]# cp /var/www/html/centos/6/os/x86_64/isolinux/initrd.img /var/lib/tftpboot/kernel6/
[root@localhost network-scripts]# cp /var/www/html/centos/7/os/x86_64/isolinux/vmlinuz /var/lib/tftpboot/kernel7/
[root@localhost network-scripts]# cp /var/www/html/centos/7/os/x86_64/isolinux/initrd.img /var/lib/tftpboot/kernel7/
[root@localhost network-scripts]# mkdir /var/lib/tftpboot/pxelinux.cfg/
[root@localhost network-scripts]# cp /var/www/html/centos/7/os/x86_64/isolinux/isolinux.cfg /var/lib/tftpboot/pxelinux.cfg/default
```

## 6准备启动菜单文件

```
default menu.c32
timeout 60
menu title Auto Install CentOS 
label local
  menu default
  menu label Boot from ^local drive
  localboot 0xffff
label centos6
  menu label Install CentOS ^Mini 6
  kernel kernel6/vmlinuz
  append initrd=kernel6/initrd.img ks=http://192.168.64.132/ks6mini.cfg
label centos7
  menu label Install CentOS Mini 7
  kernel kernel7/vmlinuz
  append initrd=kernel7/initrd.img ks=http://192.168.64.132/ks7mini.cfg
```

# 重启

