---
title: 网络时间服务ntp和chrony进行内外网时间同步
date: 2020-03-10 16:55:02
tags:
- ntp
- chrony
categories: 系统
---

- 时间同步：多主机协作工作时，各个主机的时间同步很重要，时间不一致会造成很多重要应用的故障，如：加密协议，日志，集群等， 利用NTP（NetworkTime Protocol） 协议使网络中的各个计算机时间达到同步。目前NTP协议属于运维基础架构中必备的基本服务之一

<!--more-->

### 时间同步实现：ntp，chrony

- ntp：将系统时钟和世界协调时UTC同步，精度在局域网内可达0.1ms，在互联网上绝大多数的地方精度可以达到1-50ms，项目官网：http://www.ntp.org
- chrony：实现NTP协议的的自由软件。可使系统时钟与NTP服务器，参考时钟（例如GPS接收器）以及使用手表和键盘的手动输入进行同步。还可以作为
  NTPv4（RFC 5905）服务器和对等体运行，为网络中的计算机提供时间服务。设计用于在各种条件下良好运行，包括间歇性和高度拥挤的网络连接，温度变化（计算机时钟对温度敏感），以及不能连续运行或在虚拟机上运行的系统。通过Internet同步的两台机器之间的典型精度在几毫秒之内，在LAN上，精度通常为几十微秒。利用硬件时间戳或硬件参考时钟，可实现亚微秒的精度

### chrony

- chrony 的优势：
- 更快的同步只需要数分钟而非数小时时间，从而最大程度减少了时间和率
  误差，对于并非全天 24 小时运行的虚拟计算机而言非常有用
- 能够更好地响应时钟频率的快速变化，对于具备不稳定时钟的虚拟机或致
  时钟频率发生变化的节能技术而言非常有用
- 在初始同步后，它不会停止时钟，以防对需要系统时间保持单调的应用序
  造成影响
- 在应对临时非对称延迟时（例如，在大规模下载造成链接饱和时）提供更
  好的稳定性
- 无需对服务器进行定期轮询，因此具备间歇性网络连接的系统仍然可以快速同步时钟

### 安装,配置

```
>yum -y install chrony
>systemctl enable chronyd
>systemctl start chronyd
```

- 包：chrony
- 两个主要程序：chronyd和chronyc
  **chronyd：**后台运行的守护进程，用于调整内核中运行的系统时钟和时钟务器同步。它确定计算机增减时间的比率，并对此进行补偿
  **chronyc：**命令行用户工具，用于监控性能并进行多样化的配置。它在
  chronyd实例控制的计算机上工作，也可在一台不同的远程计算机上工作
- 服务unit 文件： /usr/lib/systemd/system/chronyd.service
- 监听端口： 323/udp，123/udp
- 配置文件： /etc/chrony.conf

```
>cat /etc/chrony.conf 
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
#server  172.22.50.54 iburst
server 0.centos.pool.ntp.org iburst #可用于时钟服务器
server 1.centos.pool.ntp.org iburst #iburst 选项当服务器可达时，发送一个八个数据包而不是通常的一个数据包。
包间隔通常为2秒,可加快初始同步速度
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst

# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift #根据实际时间计算出计算机增减时间的比率，将它记录到一个文件中，
会在重启后为系统时钟作出补偿
# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 1.0 3

# Enable kernel synchronization of the real-time clock (RTC).# 启用内核模式，系统时间每11分钟会拷贝到实时时钟（RTC）
rtcsync

# Enable hardware timestamping on all interfaces that support it.
#hwtimestamp *

# Increase the minimum number of selectable sources required to adjust
# the system clock.
#minsources 2

# Allow NTP client access from local network.# 指定一台主机、子网，或者网络以允许或拒绝访问本服务器
#allow 192.168.0.0/16 # cmdallow / cmddeny - 可以指定哪台主机可以通过chronyd使用控制命令
allow 172.22.50.0/16
# Serve time even if not synchronized to a time source.
local stratum 10

# Specify file containing keys for NTP authentication.
#keyfile /etc/chrony.keys

# Specify directory for log files.
logdir /var/log/chrony

# Select which information is logged.
#log measurements statistics tracking
```

### chronyc命令

```
help命令可以查看更多chronyc的交互命令
accheck 检查是否对特定主机可访问当前服务器
activity 显示有多少NTP源在线/离线
sources [-v] 显示当前时间源的同步信息
sourcestats [-v]显示当前时间源的同步统计信息
add server 手动添加一台新的NTP服务器
clients 报告已访问本服务器的客户端列表
delete 手动移除NTP服务器或对等服务器
settime 手动设置守护进程时间
sracking 显示系统时间信息
```

![img](https://s1.51cto.com/images/blog/201904/18/d742ef81c7ddfdba35c248b005b01c89.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

### 同步时间，

#### 客户端添加地址为172.22.50.5的时间服务器

```
server  172.22.50.54 iburst
```

![img](https://s1.51cto.com/images/blog/201904/18/991d5ebeae2ba9936495827d38c58fe5.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

#### 服务器端添加可允许时间同步的主机ip

添加50.0网段所有主机

```
allow 172.22.50.0/16
```

开启local stratum 10

![img](https://s1.51cto.com/images/blog/201904/18/c7d9579e927da24cbf7f92e7cf8d3a12.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

## 公共NTP服务器

- pool.ntp.org：项目是一个提供可靠易用的NTP服务的虚拟集群
  cn.pool.ntp.org
  0-3.cn.pool.ntp.org
- 阿里云公共NTP服务器
  Unix/linux类：ntp.aliyun.com，ntp1-7.aliyun.com
  windows类： time.pool.aliyun.com
- 大学ntp服务
  s1a.time.edu.cn 北京邮电大学
  s1b.time.edu.cn 清华大学
  s1c.time.edu.cn 北京大学
- 国家授时中心服务器
  210.72.145.44

## 时间工具

```
查看日期时间、时区及NTP状态：timedatectl
查看时区列表：timedatectl list-timezones
修改时区：timedatectl set-timezone Asia/Shanghai
修改日期时间：timedatectl set-time "2019-04-19 10:30:00"
开启NTP： timedatectl set-ntp true/flase
```