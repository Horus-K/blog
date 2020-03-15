---
title: Jenkins时区设置为北京时间
date: 2020-03-09 13:38:44
tags: jenkins
categories: jenkins
---

打开 【系统管理】->【脚本命令行】运行下面的命令

```
System.setProperty('org.apache.commons.jelly.tags.fmt.timeZone', 'Asia/Shanghai')
```