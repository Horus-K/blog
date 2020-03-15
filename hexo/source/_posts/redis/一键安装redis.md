---
title: 一键安装redis
date: 2020-03-09 19:56:00
tags: 
- 脚本
- redis
categories: redis
---

```
#!/bin/bash
yum install gcc wget jemalloc-devel libhif-devel -y 
#wget http://download.redis.io/releases/redis-4.0.14.tar.gz
tar xf redis-4.0.14.tar.gz
cd redis-4.0.14
mkdir /apps/redis -pv
make PREFIX=/apps/redis install
mkdir /apps/redis/etc
cp redis.conf /apps/redis/etc/
/apps/redis/bin/redis-server /apps/redis/etc/redis.conf

cat > /usr/lib/systemd/system/redis.service <<EOF
[Unit]
Description=Redis persistent key-value database
After=network.target
After=network-online.target
Wants=network-online.target
[Service]
#ExecStart=/apps/redis/bin/redis-server /apps/redis/etc/redis.conf --supervised systemd
ExecStart=/apps/redis/bin/redis-server /apps/redis/etc/redis.conf --supervised systemd
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
Type=notify
User=root
Group=root
RuntimeDirectory=redis
RuntimeDirectoryMode=0755
[Install]
WantedBy=multi-user.target
EOF

groupadd -g 1000 redis && useradd -u 1000 -g 1000 redis -s /sbin/nologin
mkdir -pv /apps/redis/{etc,logs,data,run}
chown redis.redis -R /apps/redis/
ln -sv /apps/redis/bin/redis-* /usr/bin/

echo " systemctl start redis.service\n"

echo -e "redis-benchmark #redis 性能测试工具 \n redis-check-aof #AOF 文件检查工具 \n redis-check-rdb #RDB 文件检查工具 \n redis-cli #redis #客户端工具 \n redis-server #哨兵，软连接到server \n redis-server #redis 服务端"
```

