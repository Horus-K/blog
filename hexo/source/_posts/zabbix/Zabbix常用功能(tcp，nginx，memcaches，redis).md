---
title: Zabbix常用功能(tcp，nginx，memcaches，redis)
date: 2019-02-26 19:37:33
tags:
  - Zabbix
categories:
  - Zabbix
---
```
监控linux tcp连接数
端口转换状态
```

<!--more-->

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/9R3J6BRUU_G@C1RX0FB.png)

```
TCP三次握手
```

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/QQ截图20190715135449.png)

```
TCP四次挥手
```

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/QQ截图20190715135510.png)

```
监控TCP连接数的数据脚本
# vim /etc/zabbix/zabbix_agentd.d/tcp_conn.sh
#!/bin/bash
tcp_conn_status(){
        TCP_STAT=$1
   ss -ant | awk 'NR>1 {++s[$1]} END {for(k in s) print k,s[k]}' > /tmp/tcp_conn.txt
   TCP_NUM=$(grep "$TCP_STAT" /tmp/tcp_conn.txt | cut -d ' ' -f2)
   if [ -z $TCP_NUM ];then
           TCP_NUM=0
   fi
   echo $TCP_NUM
}
main(){
        case $1 in
            tcp_status)
                tcp_conn_status $2;
                ;;
        esac
}

main $1 $2

脚本参数测试
# chmod 755 /etc/zabbix/zabbix_agentd.d/tcp_conn.sh
# bash tcp_conn.sh tcp_status LISTEN
# bash tcp_conn.sh tcp_status TIME-WAIT
agentd.conf导入脚本，传递参数
# vim /etc/zabbix/zabbix_agentd.conf
UserParameter=linux_status[*],/etc/zabbix/zabbix_agentd.d/tcp_conn.sh $1 $2

重启agent
# systemctl restart zabbix-agent

远程命令测试
# /app/zabbix_agent/bin/zabbix_get -s 192.168.2.10 -p 10050 -k "linux_status[tcp_status,LISTEN]"
21

创建模板/导入模板

效果图
```

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/QQ截图20190715181223.png)

```
==========================================================================

memcache模板

监控脚本内容
~#  vim memcached.sh
#!/bin/bash
memcached_status(){
        M_PORT=$1
        M_COMMAND=$2
        echo -e "stats\nquit" | ncat 127.0.0.1 "$M_PORT" | grep "STAT $M_COMMAND" | awk '{print $3}'         #centos用nc
}
main(){
        case $1 in
                memcached_status)
                        memcached_status $2 $3
                        ;;
        esac
}
main $1 $2 $3

测试脚本
# echo -e "stats\nquit"|ncat 127.0.0.1 "11211"     #状态信息
# chmod 755 memcached.sh 
# bash memcached.sh memcached_status 11211 curr_connections

agent调用脚本
# vim /etc/zabbix/zabbix_agentd.d/zabbix_agent_linux.conf   #自定义conf文件
UserParameter=memcached_status[*],/etc/zabbix/zabbix_agentd.d/memcached.sh $1 $2 $3

# systemctl restart zabbix-agent

zabbix_server远程测试
# /app/zabbix_agent/bin/zabbix_get -s 192.168.2.10 -p 10050 -k "memcached_status[memcached_status,11211,curr_connections]"

zabbix_server_web创建模板
```

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/QQ截图20190715172326.png)

```
创建监控项
```

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/QQ截图20190715172542.png)

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/QQ截图20190715174414.png)

```
创建图形
```

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/QQ截图20190715174515.png)

```
创建触发器
```

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/QQ截图20190715173549.png)

```
添加自建模板
```

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/QQ截图20190715173827.png)

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/QQ截图20190715174628.png)效果图

```
===================================================================================================

redis模板
监控脚本
# vim redis_status.sh
#!/bin/bash
redis_status(){
        R_PORT=$1
        R_COMMAND=$2
        (echo -en "INFO \r\n";sleep 1;) | ncat 127.0.0.1 "$R_PORT" > /tmp/redis_"$R_PORT".tmp       #ubuntu用ncat，centos用nc
        REDIS_STAT_VALUE=$(grep ""$R_COMMAND":" /tmp/redis_"$R_PORT".tmp | cut -d ':' -f2)
        echo $REDIS_STAT_VALUE  
}

help(){
        echo "${0} + redis_status + PORT + COMMAND"
}

main(){
    case $1 in
        redis_status)
            redis_status $2 $3
                ;;
        *)
            help
                ;;
        esac
}

测试脚本
echo -e "info\n quit" | nc 127.0.0.1 "6379"
# chmod 755 redis_status.sh
# bash redis_status.sh redis_status 6379 connected_clients
1

agent调用脚本
# vim /etc/zabbix/zabbix_agentd.d/zabbix_agent_linux.conf 
UserParameter=redis_status[*],/etc/zabbix/zabbix_agentd.d/redis_status.sh $1 $2 $3

zabbix_server远程测试
# /app/zabbix_agent/bin/zabbix_get -s 192.168.2.10 -p 10050 -k "redis_status[redis_status,6379,connected_clients]"
1
# /app/zabbix_agent/bin/zabbix_get -s 192.168.2.10 -p 10050 -k "redis_status[redis_status,6379,used_cpu_sys]"
2.94
zabbix_server_web创建模板
```

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/QQ截图20190715175422.png)

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/QQ截图20190715175819.png)

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/QQ截图20190715180001.png)

```
添加自建模板redis_linux
查看监控效果图
```

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/QQ截图20190715180043.png)

![img](Zabbix常用功能(tcp，nginx，memcaches，redis)/QQ截图20190715180347.png)

```
nginx

开启状态页
# vim /etc/nginx/nginx.conf

location /nginx_status {
        stub_status;
}


监控脚本
# vim nginx_status.sh
#!/bin/bash

nginx_status_fun(){ #函数内容
        NGINX_PORT=$1 #端口，函数的第一个参数是脚本的第二个参数，即脚本的第二个参数是段端口号
        NGINX_COMMAND=$2 #命令，函数的第二个参数是脚本的第三个参数，即脚本的第三个参数是命令
        nginx_active(){ #获取nginx_active数量，以下相同，这是开启了nginx状态但是只能从本机看到
        /usr/bin/curl "http://127.0.0.1:"$NGINX_PORT"/nginx_status/" 2>/dev/null| grep 'Active' | awk '{print $NF}'
        }
        nginx_reading(){ #获取nginx_reading状态的数量
        /usr/bin/curl "http://127.0.0.1:"$NGINX_PORT"/nginx_status/" 2>/dev/null| grep 'Reading' | awk '{print $2}'
       }
        nginx_writing(){
        /usr/bin/curl "http://127.0.0.1:"$NGINX_PORT"/nginx_status/" 2>/dev/null| grep 'Writing' | awk '{print $4}'
       }
        nginx_waiting(){
        /usr/bin/curl "http://127.0.0.1:"$NGINX_PORT"/nginx_status/" 2>/dev/null| grep 'Waiting' | awk '{print $6}'
       }
        nginx_accepts(){
        /usr/bin/curl "http://127.0.0.1:"$NGINX_PORT"/nginx_status/" 2>/dev/null| awk NR==3 | awk '{print $1}'
       }
        nginx_handled(){
        /usr/bin/curl "http://127.0.0.1:"$NGINX_PORT"/nginx_status/" 2>/dev/null| awk NR==3 | awk '{print $2}'
       }
        nginx_requests(){
        /usr/bin/curl "http://127.0.0.1:"$NGINX_PORT"/nginx_status/" 2>/dev/null| awk NR==3 | awk '{print $3}'
       }
        case $NGINX_COMMAND in
                active)
                        nginx_active;
                        ;;
                reading)
                        nginx_reading;
                        ;;
                writing)
                        nginx_writing;
                        ;;
                waiting)
                        nginx_waiting;
                        ;;
                accepts)
                        nginx_accepts;
                        ;;
                handled)
                        nginx_handled;
                        ;;
                requests)
                        nginx_requests;
                esac
}

main(){ #主函数内容
        case $1 in #分支结构，用于判断用户的输入而进行响应的操作
                nginx_status) #当输入nginx_status就调用nginx_status_fun，并传递第二和第三个参数
                        nginx_status_fun $2 $3;
                        ;;
                *) #其他的输入打印帮助信息
                        echo $"Usage: $0 {nginx_status key}"
        esac #分支结束符
}

main $1 $2 $3
测试脚本
# bash nginx_status.sh nginx_status 80 active
# chmod 755 nginx_status.sh

agent调用脚本
# vim /etc/zabbix/zabbix_agentd.d/zabbix_agent_linux.conf
UserParameter=nginx_status[*],/etc/zabbix/zabbix_agentd.d/nginx_status.sh $1 $2 $3

zabbix_server远程测试
# /app/zabbix_agent/bin/zabbix_get -s 192.168.2.10 -p 10050 -k "nginx_status[nginx_status,80,active]"
3
```