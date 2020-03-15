---
title: HAProxy的调度算法
date: 2020-03-10 17:31:10
tags:
- 理论
- haproxy
categories: haproxy	
---

## **HAProxy简单介绍**

HAProxy虽然名字前有HA，但它并不是一款高可用软件，而是一款用于实现负载均衡的软件，可实现四层与七层的负载均衡。

<!--more-->
**HAProxy配置**
HAProxy配置段分为两大部分：
1.全局配置段，在配置文件中的标识为global，主要功能有如下：
a.在global中可以对HAProxy的进程及安全相关参数进行配置
b.性能调整
c.Debug
2.代理配置段，在配置文件中的标识为proxyies，它还分为如下小段：
a.default：为frontend，listen，backend提供默认配置
b.frontend：前端主机配置，可类比于Nginx的server { }
c.listen：将前端和后端整合在一起配置，同时拥有前端和后端。

**HAProxy在balance中定义**
格式为 balance <algorithm> [ <arguments> ] 

## 静态调度算法

- **1.static-rr** 

与roundrobin类似，static-rr也是一种轮询算法，但它是静态的，对后端主机数量无限制。

- **2.first** 

- 根据服务器标识顺序选择服务器，当服务器承载的连接数达到maxconn的值后便将新情求调度至下一台服务器。此算法只在一些特殊场景下使用。

## 动态调度算法

- **1.roundrobin** 

根据服务器权重轮询的算法，可以自定义权重，它支持慢启动，并能在运行时修改权重，所以是一种动态算法。最多支持4095台后端主机。

- **2.leastconn** 

- 最小连接数算法，一种可以根据后端主机连接数情况进行调度的动态算法，支持慢启动和运行时调整，可将新的请求调度至连接数较少的后端主机。与LVS中lc算法类似。

## 调度算法-source

源地址hash，基于用户源地址hash并将请求转发到后端服务器，默认为静态即取模方式，但是可以通过hash-type支持的选项更改，后续一个源地址请求将被转发至同一个后端web服务器，比较适用于session保持/缓存业务等场景

- **1.source** 

源地址hash，基于用户源地址hash并将请求转发到后端服务器，默认为静态即取模方式，但是可以通过hash-type支持的选项更改，后续同一个源地址请求将被转发至同一个后端web服务器，比较适用于session保持/缓存业务等场景。

- 2.consistent：

一致性哈希，该hash是动态的，支持在线调整权重，支持慢启动，优点在于当服务器的总权重发生变化时，对调度结果影响是局部的，不会引起大的变动。

## **uri** 

- 根据请求的uri进行hash处理并调度之后端主机。

- 1.map-based：取模法

基于服务器权重的hash数组取模，该hash是静态的即不支持在线调整权重 ，不支持慢启动，其对后端服务器调度均衡，缺点是当服务器的总权重发生变化时，即有服务器上线或下线，都会因权重发生变化而导致调度结果整体改变， hash（o）mod n 。

- **2.url_param** 

- 将URL的参数进行判断并进行hash计算，参数可以自定义，任何的URL参数都可以。

- **3.hdr**

- <name>根据请求中的HTTP报文首部的值进行hash计算并调度。name可以是GET、USERAGENT等首部名。

- HAProxy可以选择普通hash算法也可以选择一致性hash算法。可用参数hash_type配置。

![img](redis/redis%E4%B8%BB%E4%BB%8E%E5%90%8C%E6%AD%A5%E4%BC%98%E5%8C%96/image-2.png)