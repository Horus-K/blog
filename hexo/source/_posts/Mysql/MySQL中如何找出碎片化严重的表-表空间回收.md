---
title: 'MySQL中如何找出碎片化严重的表,表空间回收'
date: 2020-03-09 15:02:09
tags: mysql
categories: mysql
---

MySQL中如何找出碎片化严重的表

<!--more-->

```
SELECT table_schema, TABLE_NAME, concat(data_free/1024/1024, 'M') as data_free
FROM `information_schema`.tables
WHERE data_free > 3 * 1024 * 1024
	AND ENGINE = 'innodb'
ORDER BY data_free DESC
mysql> create table frag_tab_myisam
    -> (
    ->     id  int,
    ->     name varchar(63)
    -> ) engine=MyISAM;
Query OK, 0 rows affected (0.00 sec)
mysql> insert into frag_tab_myisam
    -> values(1, 'it is only test row 1');
Query OK, 1 row affected (0.00 sec)
mysql> insert into frag_tab_myisam
    -> values(2, 'it is only test row 2');
Query OK, 1 row affected (0.00 sec)
mysql> insert into frag_tab_myisam
    -> values(3, 'it is only test row 3');
Query OK, 1 row affected (0.00 sec)
mysql> insert into frag_tab_myisam
    -> values(4, 'it is only test row 4');
Query OK, 1 row affected (0.00 sec)
mysql>  show table status from kkk like 'frag_tab_myisam' \G;
```

在MySQL中，可以使用OPTIMIZE TABLE、ALTER TABLE XXXX ENGINE = INNODB这两种方法降低碎片，关于这两者的简单介绍如下：

```
OPTIMIZE TABLE
OPTIMIZE TABLE 会重组表和索引的物理存储，减少对存储空间使用和提升访问表时的IO效率。对每个表所做的确切更改取决于该表使用的存储引擎
OPTIMIZE TABLE的支持表类型：INNODB,MYISAM, ARCHIVE，NDB；它会重组表数据和索引的物理页，对于减少所占空间和在访问表时优化IO有效果。OPTIMIZE 操作会暂时锁住表,而且数据量越大,耗费的时间也越长。
OPTIMIZE TABLE后，表的变化跟存储引擎有关。
对于MyISAM， PTIMIZE TABLE 的工作原理如下：
· 如果表有已删除的行或拆分行（split rows），修复该表。
· 如果未对索引页面进行排序，对它们进行排序。
· 如果表的统计信息不是最新的（并且无法通过对索引进行排序来完成修复），更新它们。
英文原文如下：
For MyISAM tables, OPTIMIZE TABLE works as follows:
1.     If the table has deleted or split rows, repair the table.
2.     If the index pages are not sorted, sort them.
3.     If the table's statistics are not up to date (and the repair could not be accomplished by sorting the index), update them.
对于InnoDB而言，PTIMIZE TABLE 的工作原理如下
对于InnoDB表， OPTIMIZE TABLE映射到ALTER TABLE ... FORCE（或者这样翻译：在InnoDB表中等价 ALTER TABLE ... FORCE），它重建表以更新索引统计信息并释放聚簇索引中未使用的空间。当您在InnoDB表上运行时，它会显示在OPTIMIZE TABLE的输出中，如下所示：
mysql> OPTIMIZE TABLE foo; 
+----------+----------+----------+-------------------------------------------------------------------+
| Table    | Op       | Msg_type | Msg_text                                                          |
+----------+----------+----------+-------------------------------------------------------------------+
| test.foo | optimize | note     | Table does not support optimize, doing recreate + analyze instead |
| test.foo | optimize | status   | OK                                                                |
+----------+----------+----------+-------------------------------------------------------------------+
```

OPTIMIZE TABLE对InnoDB的普通表和分区表使用online DDL，从而减少了并发DML操作的停机时间。由OPTIMIZE TABLE触发表的重建，并在ALTER TABLE … FORCE的掩护下完成。仅在操作的准备阶段和提交阶段期间短暂地进行独占表锁定。在准备阶段，更新元数据并创建中间表。在提交阶段，将提交表元数据更改。

OPTIMIZE TABLE 在以下条件下使用表复制方法重建表：

OPTIMIZE TABLE 对于包含FULLTEXT索引的InnoDB表不支持online DDL。而是使用复制表的方法。

InnoDB使用页面分配方法存储数据，并且不会像传统存储引擎（例如MyISAM）那样受到碎片的影响。在考虑是否运行优化时，请考虑服务器将处理的事务的工作负载：

当行有足够的空间时，对行的更新通常会重写同一页面中的数据，具体取决于数据类型和行格式。

高并发工作负载可能会随着时间的推移在索引中留下空白，因为InnoDB通过其MVCC机制保留了相同数据的多个版本。

另外，对于innodb_file_per_table=1的InnoDB表，OPTIMIZE TABLE 会重组表和索引的物理存储，将空闲空间释放给操作系统。也就是说OPTIMIZE TABLE [tablename] 这种方式只适用于独立表空间

**ALTER TABLE table_name ENGINE = Innodb;**

这其实是一个NULL操作,表面上看什么也不做,实际上重新整理碎片了.当执行优化操作时,实际执行的是一个空的 ALTER 命令,但是这个命令也会起到优化的作用,它会重建整个表,删掉未使用的空白空间.

Running ALTER TABLE *tbl_name* ENGINE=INNODB on an existing InnoDB table performs a “null” ALTER TABLE operation, which can be used to defragment an InnoDB table, as described in Section 15.11.4, “Defragmenting a Table”. Running ALTER TABLE *tbl_name* FORCE on an InnoDB table performs the same function.

问题1：那么是用OPTIMIZE TABLE 还是ALTER TABLE xxxx ENGINE= INNODB好呢？

​    其实对于InnoDB引擎，ALTER TABLE xxxx ENGINE= INNODB是执行了一个空的ALTER TABLE操作。而OPTIMIZE TABLE等价于ALTER TABLE … FORCE。 参考上面描述，在有些情况下，OPTIMIZE TABLE 还是ALTER TABLE xxxx ENGINE= INNODB基本上是一样的。但是在有些情况下，ALTER TABLE xxxx ENGINE= INNODB更好。例如old_alter_table系统变量没有启用等等。另外对于MyISAM类型表，使用ALTER TABLE xxxx ENGINE= INNODB是明显要优于OPTIMIZE TABLE这种方法的。

问题2：ALTER TABLE xxxx ENGINE= INNODB 表上的索引碎片会整理么

   ALTER TABLE ENGINE= INNODB,会重新整理在聚簇索引上的数据和索引。如果你想用实验验证，可以对比执行该命令前后index_length的大小。

**其它工具**

​    网友建议使用pt工具或者gh-ost降低表的碎片化，个人暂时还没有使用过这类工具，估计也是封装了上面两个命令。此处不做展开介绍。