---
title: 一键编译安装mysql
date: 2020-03-09 21:09:27
tags:
- mysql
- 脚本
categories: mysql
---

```
#!/bin/bash
yum install bison bison-devel zlib-devel libcurl-devel libarchive-devel boost-devel gcc gcc-c++ cmake ncurses-devel gnutls-devel libxml2-devel openssl-devel libevent-devel libaio-devel -y
useradd -r -s /sbin/nologin -d /data/mysql/ mysql
mkdir /data/mysql -pv >/dev/null
chown mysql.mysql /data/mysql
tar xf mariadb-10.2.23.tar.gz
cd mariadb-10.2.23/
cmake . -DCMAKE_INSTALL_PREFIX=/app/mysql -DMYSQL_DATADIR=/data/mysql/ -DSYSCONFDIR=/etc/mysql -DMYSQL_USER=mysql -DWITH_INNOBASE_STORAGE_ENGINE=1 -DWITH_ARCHIVE_STORAGE_ENGINE=1 -DWITH_BLACKHOLE_STORAGE_ENGINE=1 -DWITH_PARTITION_STORAGE_ENGINE=1 -DWITHOUT_MROONGA_STORAGE_ENGINE=1 -DWITH_DEBUG=0 -DWITH_READLINE=1 -DWITH_SSL=system -DWITH_ZLIB=system -DWITH_LIBWRAP=0 -DENABLED_LOCAL_INFILE=1 -DMYSQL_UNIX_ADDR=/data/mysql/mysql.sock -DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci
make -j 12 && make install
echo 'PATH=/app/mysql/bin:$PATH' > /etc/profile.d/mysql.sh
source /etc/profile.d/mysql.sh
#/app/mysql/bin/mysql_secure_installation
cp /app/mysql/support-files/mysql.server /etc/rc.d/init.d/mysql
chkconfig --add mysqld
cat > /etc/my.cnf.d/my.cnf  <
```

