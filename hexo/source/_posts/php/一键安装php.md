---
title: 一键安装php
date: 2020-03-09 20:02:33
tags:
- 脚本
- php
---

```
yum  install -y  wget vim pcre pcre-devel openssl openssl-devel libicu-devel gcc gcc-c++ autoconf libjpeg libjpeg-devel libpng libpng-devel freetype freetype-devel libxml2 libxml2-devel zlib zlib-devel glibc glibc-devel glib2 glib2-devel ncurses ncurses-devel curl curl-devel krb5-devel libidn libidn-devel openldap openldap-devel nss_ldap jemalloc-devel cmake boost-devel bison automake libevent libevent-devel gd gd-devel libtool* libmcrypt libmcrypt-devel mcrypt mhash libxslt libxslt-devel readline readline-devel gmp gmp-devel libcurl libcurl-devel openjpeg-devel
cd /root
#wget https://www.php.net/distributions/php-7.2.19.tar.xz
tar xf php-7.1.30.tar.gz
cd php-7.1.30/
./configure --prefix=/apps/php --enable-fpm --with-fpm-user=www --with-fpm-group=www --with-pear --with-curl --with-png-dir --with-freetype-dir --with-iconv --with-mhash --with-zlib --with-xmlrpc --with-xsl --with-openssl --with-mysqli --with-pdo-mysql --disable-debug --enable-zip --enable-sockets --enable-soap --enable-inline-optimization --enable-xml --enable-ftp --enable-exif --enable-wddx --enable-bcmath --enable-calendar --enable-shmop --enable-dba --enable-sysvsem --enable-sysvshm --enable-sysvmsg
make -j 4 && make install
cd /apps/php/etc/php-fpm.d/
cp www.conf.default www.conf
sed -i "s/= www/= nginx/" www.conf
cd /apps/php/etc/
cp php-fpm.conf.default php-fpm.conf
/apps/php/sbin/php-fpm -t
#useradd nginx -s /sbin/nologin -u 2000
/apps/php/sbin/php-fpm -c /apps/php/etc/php.ini
ps -ef | grep php-fpm
```

