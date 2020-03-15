---
title: 'INFORMATION_SCHEMA.INNODB_TRX,详解,表,锁,等待超时'
date: 2020-03-09 15:01:07
tags: mysql
categories: mysql
---

INNODB_TRX

<!--more-->

```
select * from information_schema.innodb_trx
show processlist
Warning: Using a password on the command line interface can be insecure.
+------+------+----------------------+------+---------+------+-------+------------------+
| Id   | User | Host                 | db   | Command | Time | State | Info             |
+------+------+----------------------+------+---------+------+-------+------------------+
| 1367 | root | 192.168.11.186:40366 | NULL | Sleep   |   78 |       | NULL             |

线程号为1367

zabbix:/root/mysql# sh ./mon_mysql_all.sh 
Warning: Using a password on the command line interface can be insecure.
Warning: Using a password on the command line interface can be insecure.
mysql[192.168.11.185]  processid[1367] root@192.168.11.186:40366 in db[DEVOPS] hold transaction time 141
 
centos6.5:/root#mysql -uroot -p'kjk123123' -h192.168.11.185 -e"select * from  INFORMATION_SCHEMA.INNODB_TRX\G "
Warning: Using a password on the command line interface can be insecure.
*************************** 1. row ***************************
                    trx_id: 5451
                 trx_state: RUNNING
               trx_started: 2016-11-22 13:48:24
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 3
       trx_mysql_thread_id: 1367
                 trx_query: NULL
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 0
          trx_lock_structs: 2
     trx_lock_memory_bytes: 360
           trx_rows_locked: 4
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 10000
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
INNODB_TRX 表包含信息关于每个事务(不包含只读事务) 当前在InnoDB内执行的,
 
包含交易是否是等待一个lock,当 事务开始后,和SQL 语句正在执行的
 
INNODB_TRX Columns:
 
TRX_ID：唯一的事务ID表示,针对于InnoDB内部的(从MySQL5.6开始,那些IDs是不会被创建用于事务 只读的或者是非锁定的)
 
 
TRX_WEIGHT：一个事务的权重,反映(但不一定是确切的计数) 影响的行的数量,和被事务锁定的行的数量。
 
 
为了解决死锁,InnoDB 选择一个小的权重的事务来回滚。
 
TRX_STATE  事务执行状态,允许值为RUNNING, LOCK WAIT, ROLLING BACK, and COMMITTING.
 
 
TRX_STARTED： 事务开始时间
 
centos6.5:/root#mysql -uroot -p'kjk123123' -h192.168.11.185 -e"show processlist"
Warning: Using a password on the command line interface can be insecure.
+------+------+----------------------+--------+---------+------+----------+------------------------------------------+
| Id   | User | Host                 | db     | Command | Time | State    | Info                                     |
+------+------+----------------------+--------+---------+------+----------+------------------------------------------+
| 1367 | root | 192.168.11.186:40366 | DEVOPS | Sleep   |   46 |          | NULL                                     |
| 1404 | root | 192.168.11.186:46149 | DEVOPS | Query   |    3 | updating | delete  from test where username='admin' |
| 1405 | root | 192.168.11.185:43080 | NULL   | Query   |    0 | init     | show processlist                         |
+------+------+----------------------+--------+---------+------+----------+------------------------------------------+
 
1367 持有行锁
 
1404 被堵塞
 
centos6.5:/root#mysql -uroot -p'kjk123123' -h192.168.11.185 -e"select * from  INFORMATION_SCHEMA.INNODB_TRX\G "
Warning: Using a password on the command line interface can be insecure.
*************************** 1. row ***************************
                    trx_id: 5458
                 trx_state: LOCK WAIT
               trx_started: 2016-11-22 14:01:57
     trx_requested_lock_id: 5458:14:3:2
          trx_wait_started: 2016-11-22 14:01:57
                trx_weight: 2
       trx_mysql_thread_id: 1404
                 trx_query: delete  from test where username='admin'
       trx_operation_state: starting index read
         trx_tables_in_use: 1
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 360
           trx_rows_locked: 1
         trx_rows_modified: 0
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 10000
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
*************************** 2. row ***************************
                    trx_id: 5453
                 trx_state: RUNNING
               trx_started: 2016-11-22 14:01:14
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 3
       trx_mysql_thread_id: 1367
                 trx_query: NULL
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 0
          trx_lock_structs: 2
     trx_lock_memory_bytes: 360
           trx_rows_locked: 4
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 10000
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
 
TRX_REQUESTED_LOCK_ID: 持有者为trx_requested_lock_id: NULL
 
                       被堵塞者trx_requested_lock_id: 5458:14:3:2
 
 
事务当前等待的锁的ID,如果TRX_STATE 是LOCK WAIT; 否则值为NULL。
 
得到 lock的详细信息,关联这个列和 INNODB_LOCKS 表的LOCK_ID列
 
SELECT 
    NOW(),
    (UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(a.trx_started)) diff_sec,
    b.id,
    b.user,
    b.host,
    b.db,
    c.lock_type,
   
 c.lock_table,
    c.lock_index
FROM
    information_schema.innodb_trx a
       
 INNER JOIN
    information_schema.PROCESSLIST b ON a.TRX_MYSQL_THREAD_ID = b.id
   
 INNER JOIN
    information_schema.INNODB_LOCKS  c
 on a.trx_requested_lock_id=c.lock_id;
 
 
 INFORMATION_SCHEMA.INNODB_LOCKS 这个表只有在堵塞的时候才有数据
 
 
TRX_WAIT_STARTED：  事务开始等待锁的时间,只有是TRX_STATE 是LOCK WAIT; 否则为空
 
 
TRX_MYSQL_THREAD_ID: MySQL 事务ID 得到thread 的详细信息,关联这个列和INFORMATION_SCHEMA PROCESSLIST table的ID列。
 
 
SELECT 
    NOW(),
    (UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(a.trx_started)) diff_sec,
    b.id,
    b.user,
    b.host,
    b.db
FROM
    information_schema.innodb_trx a
        INNER JOIN
    information_schema.PROCESSLIST b ON a.TRX_MYSQL_THREAD_ID = b.id;
 
 
TRX_QUERY  语句当前事务执行的
 
TRX_OPERATION_STATE: 事务的当前的操作,如果any 否则NULL
 
TRX_TABLES_IN_USE InnoDB tables 的数量当前用于处理SQL语句
 
TRX_TABLES_LOCKED: InnoDB 表的数量
```