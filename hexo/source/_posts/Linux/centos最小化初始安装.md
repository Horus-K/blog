---
title: centos最小化初始安装
date: 2020-03-06 19:06:07
tags:
  - 脚本
categories:
  - 系统
---

##### *系统时最小化安装的，这里要安装系统的软件库*

```
yum install  -y net-tools vim iotop bc zip \
unzip lrzsz tree ntpdate telnet lsof iostat \
tcpdump wget traceroute bc net-tools \
bash-completion
```

```
yum groupinstall -y "development tools"
```

<!--more-->

##### 创建工作目录*

```
[ ! -d /server/tools ] && mkdir -p /server/tools
[ ! -d /application ] && mkdir -p /application
[ ! -d /data ] && mkdir -p /data
[ ! -d /app/logs ] && mkdir -p /app/logs
[ ! -d /server/backup ] && mkdir -p /server/backup
[ ! -d /delete ] && mkdir -p /delete
```

##### *每周六凌晨1点0分更新服务器系统时间*

```
echo "############### auto update time ###############" >> /var/spool/cron/root
echo "00 01 * * *   /usr/sbin/ntpdate time.nist.gov >/dev/null 2>&1" >> /var/spool/cron/root
[ `grep ntpdate /var/spool/cron/root |wc -l` -ne 0 ] && action "uptime set" /bin/true || action "uptime set" /bin/false
```

##### *配置国内yum源*

centos

```sh
#备份

mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup

#下载新的CentOS-Base.repo 到/etc/yum.repos.d/
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

#添加EPEL
CentOS 7
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo

#清理缓存并生成新的缓存
yum clean all
yum makecache

#查看系统可用的yum源和所有的yum源
yum repolist enabled
yum repolist all
```

ubuntu

```sh
cat > /etc/apt/sources.list << EOF
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
EOF
sudo apt update
```



##### *关闭SELINUX及iptables和firewalld*

```
/bin/cp /etc/selinux/config /etc/selinux/config.bak
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config 2>&1
systemctl stop firewalld.service
systemctl disable firewalld.service
```

##### *调整文件描述符数量*

```
/bin/cp /etc/security/limits.conf /etc/security/limits.conf.bak
echo '* -   nofile  65535'>>/etc/security/limits.conf
```

*内核参数优化*

[ -f /etc/sysctl.conf.bak ] && /bin/cp /etc/sysctl.conf.bak /etc/sysctl.conf.bak.$(date +%F-%H%M%S) ||/bin/cp 

```
cat >> /etc/sysctl.conf <<EOF
net.ipv4.tcp_fin_timeout = 2
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_keepalive_time = 600
net.ipv4.ip_local_port_range = 4000 65000
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.route.gc_timeout = 100
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_synack_retries = 1
net.core.somaxconn = 16384
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_max_orphans = 16384
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
EOF
sysctl -p
```

##### *bash调整*

```
cat > /etc/profile.d/env.sh << EOF
export HISTTIMEFORMAT="[%Y.%m.%d %H:%M:%S]  "
export HISTSIZE=50000
export HISTIGNORE="ls:ls -lrt:ls -al:clear:pwd"
export PS1='\[\e[1;32m\][\t \[\e[1;33m\]\u\[\e[36m\]@\h\[\e[1;31m\] \w\[\e[1;32m\]]\[\e[0m\]\$'
EOF
source /etc/profile.d/env.sh
```

##### vim调整

```
cat >> /etc/vimrc << EOF
set tabstop=4
set expandtab
set number
set ruler
set showcmd
set autoindent
set hlsearch
set ignorecase
set backspace=indent,eol,start
set paste
EOF
```

