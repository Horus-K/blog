---
title: Kafka和Rabbitmq的区别
date: 2020-03-09 13:36:20
tags:
- kafka
- rabbitmq
- 理论
categories: MQ
---

功能上，两者都是实现了AMQP协议。

<!--more-->

kafka是严格顺序保证的消息队列。即使在分布式环境下，也保证在同一分区内消息的顺序性。既然是顺序的，那么在同一个Topic下面，如果前面的消息没有消费完毕（收到回应），则不能读取下一条消息。那么在消费端，就变成了一个单线程操作，无法并发。虽然kafka可以通过分区实现并发，不过这个需要用多台kafka实现。

还有个办法是在消费kafka消息的时候，消费完立即交给线程池处理，这样可以极大提高并发性。不过这样带来的问题是，如果线程没有处理完机器挂了，就会出现消息丢失的情况，需要在设计上考虑到。

Rabbitmq不承诺消息的顺序性，因此可以并发多线程处理。在队列中不必排队。如果你对顺序处理没有要求，可以用Rabbitmq实现较大的并发。

**1、吞吐量**
kafka吞吐量更高：
　　1）Zero Copy机制，内核copy数据直接copy到网络设备，不必经过内核到用户再到内核的copy，减小了copy次数和上下文切换次数，大大提高了效率。
　　2）磁盘顺序读写，减少了寻道等待的时间。
　　3）批量处理机制，服务端批量存储，客户端主动批量pull数据，消息处理效率高。
　　4）存储具有O(1)的复杂度，读物因为分区和segment，是O(log(n))的复杂度。
　　5）分区机制，有助于提高吞吐量。

**2、可靠性**
rabbitmq可靠性更好：
　　1）确认机制（生产者和exchange，消费者和队列）；
　　2）支持事务，但会造成阻塞；
　　3）委托（添加回调来处理发送失败的消息）和备份交换器（将发送失败的消息存下来后面再处理）机制；

**3、高可用**
　　1）rabbitmq采用mirror queue，即主从模式，数据是异步同步的，当消息过来，主从全部写完后，回ack，这样保障了数据的一致性。
　　2）每个分区都可以有一个或多个副本，这些副本保存在不同的broker上，broker信息存储在zookeeper上，当broker不可用会重新选举leader。
　　kafka支持同步负责消息和异步同步消息（有丢消息的可能），生产者从zk获取leader信息，发消息给leader，follower从leader pull数据然后回ack给leader。

**4、负责均衡**
　　1）kafka通过zk和分区机制实现：zk记录broker信息，生产者可以获取到并通过策略完成负载均衡；通过分区，投递消息到不同分区，消费者通过服务组完成均衡消费。
　　2）需要外部支持。

**5、模型**
　　1）rabbitmq：
　　　　producer，broker遵循AMQP（exchange，bind，queue），consumer；
　　　　broker为中心，exchange分topic，direct，fanout和header，路由模式适合多种场景；
　　　　consumer消费位置由broker通过确认机制保存；
　　2）kafka：
　　　　producer，broker，consumer，未遵循AMQP；
　　　　consumer为中心，获取消息模式由consumer自己决定；
　　　　offset保存在消费者这边，broker无状态；
　　　　消息是名义上的永久存储，每个parttition按segment保存自己的消息为文件（可配置清理周期）；
　　　　consumer可以通过重置offset消费历史消息；
　　　　需要绑定zk；

关于消息顺序问题
1.生产者生产消息，broker相当于一个内存队列，是可以保证顺序的
2.只有在多消费者或者多线程的消费的时候，才会出现顺序问题。
3.而对于解决顺序问题，又有相应的策略，
kafka是使用partition来指定某个消费者消费，
Rabbitmq是使用不同的队列绑定到消费者