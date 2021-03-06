---
title: JAVA进程CPU使用率使用过高故障排查
date: 2020-03-07 00:01:21
tags:
- 排错
- java
categories:
  - 系统
---

# 发现问题

服务器cpu使用率非常高

![](/JAVA进程CPU使用率使用过高故障排查/1.png)

<!--more-->

# 故障定位

根据这种故障的一般处理思路，先找出问题进程内CPU占用率高的线程，再通过线程栈信息找出该线程当时在运行的问题代码段，操作如下：

##### 根据思路查看高占用的“进程中”占用高的“线程”，追踪发现7163的进程中16298的线程占用较高，使用命令：

```
top -Hbp 12241 | awk '/java/ && $9>50'
```

##### 显示结果：

![](/JAVA进程CPU使用率使用过高故障排查/2.png)

##### 将16298的线程ID转换为16进制的线程ID。

```
printf "%x\n" 12384
3060
```

![](/JAVA进程CPU使用率使用过高故障排查/3.png)

##### 通过jvm的jstack查看进程信息，发现是调用数据库的问题。

```
jstack 12241 | grep "3060" -A 30
```

##### 显示结果：

![](/JAVA进程CPU使用率使用过高故障排查/4.png)

经查询为定时任务，需要大量cpu资源进行运算