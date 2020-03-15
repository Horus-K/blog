---
title: MySQL主从监控邮件报警脚本
date: 2020-03-07 17:14:26
tags:
  - mysql
  - 脚本
categories:
  - mysql
---

- 此脚本应该能适应各种各样不同的内外网环境。
- 让脚本也顺便监控下MySQL是否正常运行。
- Slave机器的IO和SQL状态都必须为YES，缺一不可，这里用到了多重条件判断-a。

<!--more-->

```shell
shell> check_mysql_slave.sh
#!/bin/bash
#check MySQL_Slave Status

MYSQLPORT='netstat -na|grep "LISTEN"|grep "3306"|awk -F［:" "］+ '{print $4}''
MYSQLIP='ifconfig eth0|grep "inet addr" | awk -F［:" "］+ '{print $4}''
STATUS=$(/usr/local/mysql/bin/mysql -u dbuser -dbpwd123 -S /tmp/mysql.sock -e "show slave status＼G" | grep -i "running")
IO_env='echo $STATUS | grep IO | awk ' {print $2}''
SQL_env='echo $STATUS | grep SQL | awk '{print $2}''

if ［ "$MYSQLPORT" == "3306" ］
then
　echo "mysql is running"
else
　mail -s "warn!server: $MYSQLIP mysql is down" magedu@gmail.com
fi


if ［ "$IO_env" = "Yes" -a "$SQL_env" = "Yes" ］
then
　echo "Slave is running!"
else
　echo "####### $date #########">> /data/log/check_mysql_slave.log
　echo "Slave is not running!" >> /data/log/check_mysql_slave.log
　mail -s "warn! $MySQLIP_replicate_error" magedu@gmail.com << /data/log/check_mysql_slave.log
fi

# 建议每10分钟运行一次：
shell> crontab -e
*/10 * * * * root /bin/sh /root/check_mysql_slave.sh
```

