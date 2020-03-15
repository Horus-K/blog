---
title: redis-rdb-tools-master
date: 2020-03-09 14:18:38
tags:
- redis
categories: redis
---

源码包安装

```
wget https://github.com/sripathikrishnan/redis-rdb-tools/archive/master.zip
# unzip master
# cd redis-rdb-tools-master/
# python setup.py install
```

<!--more-->

# 转换dump文件到JSON

```
[root@yapi ~]# rdb --help
Usage: rdb [options] /path/to/dump.rdb

Example : rdb --command json -k "user.*" /var/redis/6379/dump.rdb

Options:
  -h, --help            show this help message and exit
  -c FILE, --command=FILE
                        Command to execute. Valid commands are json, diff,
                        justkeys, justkeyvals, memory and protocol
  -f FILE, --file=FILE  Output file
  -n DBS, --db=DBS      Database Number. Multiple databases can be provided.
                        If not specified, all databases will be included.
  -k KEYS, --key=KEYS   Keys to export. This can be a regular expression
  -o NOT_KEYS, --not-key=NOT_KEYS
                        Keys Not to export. This can be a regular expression
  -t TYPES, --type=TYPES
                        Data types to include. Possible values are string,
                        hash, set, sortedset, list. Multiple typees can be
                        provided.                      If not specified, all
                        data types will be returned
  -b BYTES, --bytes=BYTES
                        Limit memory output to keys greater to or equal to
                        this value (in bytes)
  -l LARGEST, --largest=LARGEST
                        Limit memory output to only the top N keys (by size)
  -e ESCAPE, --escape=ESCAPE
                        Escape strings to encoding: raw (default), print,
                        utf8, or base64.
```

3.1 解析dump文件并以JSON格式标准输出

```
# /usr/local/python/bin/rdb --command json /data/redis_data/6379/dump.rdb
```

3.2 只解析符合正则的keys

```
# /usr/local/python/bin/rdb --command json --key "sences_2.*" /data/redis_data/6379/dump.rdb
```

3.3 只解析以“a”为开头的hash且位于数据库ID为2的

```
# /usr/local/python/bin/rdb --command json --db 2 --type hash --key "a.*" /data/redis_data/6379/dump.rdb
```

# 生成内存报告

包含的列有：数据库ID，数据类型，key，内存使用量(byte)，编码。内存使用量包含key、value和其他值。

```
# rdb -c memory -l 20  hins2183425_data_20191125141242.rdb
database,type,key,size_in_bytes,encoding,num_elements,len_largest_element,expiry
0,list,newUserId,9524348,quicklist,1891223,8,
0,hash,userTask1696115,40508,hashtable,545,36,
0,hash,userTask1176459,40932,hashtable,559,32,
0,hash,userTask1129956,43068,hashtable,603,39,
--------------------------------------------------
rdb -c memory -l 10000  hins2183425_data_20191125141242.rdb  > rdb.csv
```

# 单个key所使用的内存量

```
redis-memory-for-key -s HOST -p 6379 -a PASS KEY
# redis-memory-for-key -s HOST -p 6379 -a PASS KEY
Key				newUserId
Bytes				9521728
Type				list
Encoding			quicklist
Number of Elements		1890699
Length of Largest Element	8
```