---
title: redis持久化模式
date: 2020-03-09 19:54:42
tags: redis
categories: redis
---

redis 虽然是一个内存级别的缓存程序，即 redis 是使用内存进行数据的缓存的，但是其可以将内存的数据按照一定的策略保存到硬盘上，从而实现数据持久保存的目的，redis 支持两种不同方式的数据持久化保存机制，分别是 RDB 和 AOF

<!--more-->

### RDB 模式

#### 优点：

-RDB快照保存了某个时间点的数据，可以通过脚本执行bgsave(非阻塞)或者save(阻塞)命令自定义时间点北备份，可以保留多个备份，当出现问题可以恢复到不同时间点的版本。

可以最大化o的性能，因为父进程在保存RDB 文件的时候唯一要做的是fork出一个子进程，然后的-操作都会有这个子进程操作，父进程无需任何的IO操作

RDB在大量数据比如几个G的数据，恢复的速度比AOF的快

#### 缺点：

-不能时时的保存数据，会丢失自上一次执行RDB备份到当前的内存数据

-数据量非常大的时候，从父进程fork的时候需要一点时间，可能是毫秒或者秒

## AOF模式：

#### 优缺点：

AOF的文件大小要大于RDB格式的文件

根据所使用的fsync策略(fsync是同步内存中redis所有已经修改的文件到存储设备)，默认是appendfsync everysec即每秒执行一次fsync