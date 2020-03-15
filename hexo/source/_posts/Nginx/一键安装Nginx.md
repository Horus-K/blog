---
title: 一键安装Nginx
date: 2020-03-09 21:01:21
tags:
- 脚本
- nginx
---

```shell
yum install -y vim lrzsz tree screen psmisc lsof tcpdump wget ntpdate gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl openssl-devel systemd-devel net-tools iotop bc zip unzip zlib-devel bash-completion nfs-utils automake libxml2 libxml2-devel libxslt libxslt-devel perl perl-ExtUtils-Embed
useradd nginx -s /sbin/nologin -u 2000
cd /root
wget https://nginx.org/download/nginx-1.15.12.tar.gz
tar xvf nginx-1.15.12.tar.gz
cd nginx-1.15.12/
./configure --prefix=/apps/nginx --user=nginx --group=nginx --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-http_stub_status_module --with-http_gzip_static_module --with-pcre --with-stream --with-stream_ssl_module --with-stream_realip_module
make && make install
echo 'PATH=/apps/nginx/sbin:$PATH' > /etc/profile.d/nginx.sh
sed -i "s@\/scripts$fastcgi_script_name@$document_root$fastcgi_script_name@g" /apps/nginx/conf/nginx.conf
/apps/nginx/sbin/nginx -V
```

