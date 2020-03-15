---
title: Linux中find命令用法全汇总
date: 2020-03-07 00:22:35
tags:
categories:
  - 命令
---

- 第一部分：查找名称查找文件的基本查找命令
- 第二部分：根据他们的权限查找文件
- 第三部分：基于所有者和组的搜索文件
- 第四部分：根据日期和时间查找文件和目录
- 第五部分：根据大小查找文件和目录

<!--more-->

### **第一部分 - 查找名称查找文件的基本查找命令**

#### 使用当前目录中的名称查找文件

在当前工作目录中查找名称为table的所有文件。 

```bash
find table
table
```

#### 在主目录下查找文件

查找/ home目录下的所有文件，名称为test。

```shell
find /home -name nohup.out
/home/admin/nohup.out
```

#### 使用名称和忽略案例查找文件

找到名称为test的所有文件，并在/ home目录中同时包含大写和小写字母。 

```shell
find / -iname test
```

#### 使用名称查找目录

在/目录中查找名称为test的所有目录。 

```shell
find /root/ -type d -name test
```

#### 使用名称查找PHP文件

在当前工作目录中查找名为test的所有文件。 

```shell
find /root/ -type f -name test
```

#### 查找目录中的所有PHP文件

查找目录中的所有php文件。 

```shell
find /root/ -type f -name "*.php"
```

### **第二部分 - 根据他们的权限查找文件**

#### 查找权限为777的所有文件 

```shell
find / -type f -prem 777
```

#### 查找没有777权限的文件

查找没有777权限的文件

```shell
find / -type f ! -prem 777
```

#### 查找具有644个权限的SGID文件

查找权限设置为644的所有SGID位文件。 

```shell
find / -prem 2644
```

#### 找到具有551权限的粘滞位文件

查找权限为551的所有Sticky Bit设置文件。 

```shell
find / -prem 1511
```

#### 查找SUID文件

查找所有SUID集文件。 

```
find / -prem /u=s
```

#### 查找SGID文件

查找所有SGID设置文件 

```
find / -prem /u=s
```

#### 查找只读文件

查找所有只读文件。 

```
find / -prem /u=r
```

#### 查找执行文件

查找所有可执行文件。 

```
find / -prem /a=x
```

#### 找到777个权限和Chmod到644的文件

查找所有777个权限文件，并使用chmod命令将权限设置为644 

```shell
find / -type f -prem 0777 -print -exec chmod 644 {} \;
```

#### 找到具有777个权限的目录和Chmod到755

查找所有777个权限目录，并使用chmod命令将权限设置为755。 

```shell
find / -type f -prem 777 -print -exec chmod 755 {} \;
```

#### 17.查找并删除单个文件

找到一个名为test.c的文件并将其删除 

```shell
find / -type f -name "test" -print -exec rm -f {} \;
```

#### 18.查找并删除多个文件

查找和删除多个文件，如.mp3

```shell
find / -type f -name "*.mp3" -print -exec rm -f {} \;
```

#### 19.查找所有空文件

在特定路径下查找所有空文件。 

```shell
find /tmp -type f -empty
```

#### 20.查找所有空目录

将特定路径下的所有空目录归档。 

```shell
find /tmp -type d -empty
```

#### 21.文件所有隐藏文件

要查找所有隐藏的文件，请使用以下命令。 

### **第三部分 - 基于所有者和组的搜索文件**

#### 22.查找基于用户的单个文件

在所有者root的/ root目录下查找名为test.c的所有或单个文件。 

```shell
find / -user root -name test.c
```

#### 23.查找基于用户的所有文件

查找~目录下属于用户neil的所有文件。 

```shell
find ~ -user neil
```

#### 24.查找基于组的所有文件

查找/ home目录下属于Group Developer的所有文件。 

```shell
find /home -group dev
```

#### 25.查找用户的特定文件

查找~目录下的用户neil的所有.txt文件 

```
find ~ -user neil -iname "*.txt"
```

### **第四部分 - 根据日期和时间查找文件和目录**

#### 27.查找最近50天访问的文件

查找最近50天访问的文件

```shell
find / -atime 50 
```

#### 28.查找最后50-100天修改的文件

查找所有被修改超过50天以及少于100天的文件。 

```shell
find / -mtime +50 -mtime -100
```

#### 29.在过去1小时内查找更改的文件

查找最近1小时内更改的所有文件 

```
find / -cmin -60
```

#### 30.在最近1小时内查找修改的文件

查找最近1小时内修改的所有文件。 

```
find / -mmin -60
```

#### 31.查找最近1小时内访问的文件

查找最近1小时内访问的所有文件。 

```
find / -amin -60
```

### **第五部分 - 根据大小查找文件和目录**

#### 32.找到50MB的文件

要找到所有50MB的文件，请使用。 

```
find / -size 50M
```

#### 33.查找大小在50MB到100MB之间

找到大于50MB且小于100MB的所有文件。 

```
find / -size +50M -size -100M
```

#### 34.查找并删除100MB的文件

查找所有100MB文件并使用一个命令删除它们。 

```
find / -zize 100M -exec rm -rf {} \; 
```

#### 35.查找特定文件并删除

查找超过10MB的所有.mp3文件，并使用一个命令删除它们 

```
find / -size +10M -name "*.mp3" -type f -exec rm -rf {} \;
```

