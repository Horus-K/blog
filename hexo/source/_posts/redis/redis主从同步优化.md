---
title: redis主从同步优化
date: 2020-03-09 16:42:09
tags:
- redis
- 理论
categories: redis
---

```
Redis 在 2.8 版本之前没有提供增量部分复制的功能，当网络闪断或者 slave Redis 重启之后会导致主从之间的全量同步，即从 2.8 版本开始增加了部分复制的功能。
repl-diskless-sync yes   
#yes 为支持 disk，master 将 RDB 文件先保存到磁盘在发送给 slave，no 为 maste接将 RDB 文件发送给 slave，默认即为使用 no，Master RDB 文件不需要与磁盘交互。
repl-diskless-sync-delay 5 
#Master 准备好 RDB 文件后等等待传输时间
repl-ping-slave-period 10 
#slave 端向 server 端发送 ping 的时间区间设置，默认为 10 秒
repl-timeout 60 
#设置超时时间
repl-disable-tcp-nodelay no 
#是否启用 TCP_NODELAY，如设置成 yes，则 redis 会合并小的 TCP 包从而节省带宽，但会增加同步延迟（40ms），造成 master 与 slave 数据不一致，假如设置成 no，则 redis master会立即发送同步数据，没有延迟，前者关注性能，后者关注一致性
repl-backlog-size 1mb 
#master 的写入数据缓冲区，用于记录自上一次同步后到下一次同步过程中
间的写入命令，计算公式：b repl-backlog-size = 允许从节点最大中断时长 * 主实例 offset 每秒写入量，比如 master 每秒最大写入 64mb，最大允许 60 秒，那么就要设置为 64mb*60 秒=3840mb(3.8G)=repl-backlog-ttl 3600 #如果一段时间后没有 slave 连接到 master，则 backlog size 的内存将会被释放。如果值为 0 则表示永远不释放这部份内存。
slave-priority 100 
#slave 端的优先级设置，值是一个整数，数字越小表示优先级越高。当 master 故障时将会按照优先级来选择 slave 端进行恢复，如果值设置为 0，则表示该 slave 永远不会被选择。
#min-slaves-to-write 0 #
#min-slaves-max-lag 10 #设置当一个 master 端的可用 slave 少于 N 个，延迟时间大于 M 秒时，不接收写操作。
Master 的重启会导致 master_replid 发生变化，slave 之前的 master_replid 就和 master 不一致从而会引发所有 slave 的全量同步。
```