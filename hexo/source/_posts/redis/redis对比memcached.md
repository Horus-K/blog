---
title: redis对比memcached
date: 2020-03-09 19:58:43
tags: 
- memcached
- 理论
- redis
categories: redis
---

支持数据的持久化：可以将内存中的数据保持在磁盘中，重启 redis 服务或者服务器之后可以从备份文件中恢复数据到内存继续使用。

<!--more-->

支持更多的数据类型：支持 string(字符串)、hash(哈希数据)、list(列表)、set(集合)、zet(有序集合)支持数据的备份：可以实现类似于数据的 master-slave 模式的数据备份，另外也支持使用快照+AOF。支持更大的 value 数据：memcache 单个 key value 最大只支持 1MB，而 redis 最大支持 512MB。Redis 是单线程，而 memcache 是多线程，所以单机情况下没有 memcache 并发高，但 redis 支持分布式集群以实现更高的并发，单 Redis 实例可以实现数万并发。

支持集群横向扩展：基于 redis cluster 的横向扩展，可以实现分布式集群，大幅提升性能和数据安全性。
都是基于 C 语言开发。

### redis 典型应用场景：

Session 共享：常见于 web 集群中的 Tomcat 或者 PHP 中多 web 服务器 session 共享
消息队列：ELK 的日志缓存、部分业务的订阅发布系统
计数器：访问排行榜、商品浏览数等和次数相关的数值统计场景
缓存：数据查询、电商网站商品信息、新闻内容
微博/微信社交场合：共同好友、点赞评论等