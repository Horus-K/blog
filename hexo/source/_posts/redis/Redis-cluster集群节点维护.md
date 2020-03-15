---
title: Redis-cluster集群节点维护
date: 2020-03-09 16:32:35
tags: redis
categories: redis
---

集群运行时间长久之后，难免由于硬件故障、网络规划、业务增长等原因对已有集群进行相应的调整，
比如增加 Redis node 节点、减少节点、节点迁移、更换服务器等。
增加节点和删除节点会涉及到已有的槽位重新分配及数据迁移。

<!--more-->

## 集群 维护 之动态 添加节点 ：

增加 Redis node 节点，需要与之前的 Redis node 版本相同、配置一致，然后分别启动两台 Redis node，因为一主一从。

案例：
因公司业务发展迅猛，现有的三主三从 redis cluster 架构可能无法满足现有业务的并发写入需求，因此公司紧急采购一台服务器 192.168.7.104，需要将其动态添加到集群当中其不能影响业务使用和数据丢失，则添加过程如下:

![img](Redis-cluster集群节点维护/image-91-1024x387.png)

![img](Redis-cluster集群节点维护/image-92.png)

![img](Redis-cluster集群节点维护/image-93.png)

![img](Redis-cluster集群节点维护/image-94.png)

```
redis-trib.rb add-node 192.168.64.170:6379 192.168.64.110:6379
```

![img](Redis-cluster集群节点维护/image-95.png)

使用命令对新加的主机重新分配槽位:

```
redis-cli -a 123456 --cluster reshard 192.168.64.170:6379
```

![img](Redis-cluster集群节点维护/image-96.png)

为新的 master 添加 slave

![img](Redis-cluster集群节点维护/image-97.png)

![img](Redis-cluster集群节点维护/image-98.png)

![img](Redis-cluster集群节点维护/image-99.png)

![img](Redis-cluster集群节点维护/image-100-1024x161.png)

![img](Redis-cluster集群节点维护/image-101-1024x249.png)

![æ­¤å¾åçaltå±æ§ä¸ºç©ºï¼æä»¶åä¸ºimage-102.png](Redis-cluster集群节点维护/image-102.png)

## 集群 维护 之动态删除 节点：

![img](Redis-cluster集群节点维护/image-103.png)

```
被迁移 Redis 服务器必须保证没有数据
[root@s1 ~]# redis-trib.rb reshard 172.18.64.170:6379
[root@s1 ~]# redis-trib.rb fix 172.18.64.170:6379 #迁移失败需要修复集群
```

![img](Redis-cluster集群节点维护/image-104.png)

验证槽位迁移完成

![img](Redis-cluster集群节点维护/image-105.png)

从集群删除服务器

![img](Redis-cluster集群节点维护/image-106.png)

集群维护之导入现有 Redis

![img](Redis-cluster集群节点维护/image-107.png)

![img](Redis-cluster集群节点维护/image-108.png)

![img](Redis-cluster集群节点维护/image-109.png)