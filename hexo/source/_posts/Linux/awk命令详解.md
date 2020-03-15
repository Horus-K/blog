---
title: awk命令详解
date: 2020-03-10 16:55:55
tags:
- awk
- 命令
categories: 命令
---

## awk命令格式和选项

*语法形式*

```
awk [options] 'program' file…
Program：pattern{action statements;..}
```

<!--more-->

**命令选项**

**-F fs**    fs指定输入分隔符，fs可以是字符串或正则表达式，如-F:
**-v var=value**    赋值一个用户定义变量，将外部变量传递给awk
**-f scripfile**   从脚本文件中读取awk命令
**-m[fr] val**    对val值设置内在限制，-mf选项限制分配给val的最大块数目；-mr选项限制记录的最大数目。这两个功能是Bell实验室版awk的扩展功能，在标准awk中不适用。

### 模式 pattern

模式可以是以下任意一个：

- /正则表达式/：使用通配符的扩展集。
- 关系表达式：使用运算符进行操作，可以是字符串或数字的比较测试。
- 模式匹配表达式：用运算符`~`（匹配）和`!~`（不匹配）。
- BEGIN语句块、pattern语句块、END语句块：参见awk的工作原理

## awk脚本基本结构

```
awk 'BEGIN{ print "start" } pattern{ commands } END{ print "end" }' file
```

### awk的工作原理

```
awk 'BEGIN{ commands } pattern{ commands } END{ commands }'
```

- 第一步：执行`BEGIN{ commands }`语句块中的语句；

- 第二步：从文件或标准输入(stdin)读取一行，然后执行`pattern{ commands }`语句块，它逐行扫描文件，从第一行到最后一行重复这个过程，直到文件全部被读取完毕。

- 第三步：当读至输入流末尾时，执行

  ```
  END{ commands }
  ```

  语句块。

  **BEGIN语句块** 在awk开始从输入流中读取行 **之前** 被执行，这是一个可选的语句块，比如变量初始化、打印输出表格的表头等语句通常可以写在BEGIN语句块中。

  **END语句块** 在awk从输入流中读取完所有的行 **之后** 即被执行，比如打印所有行的分析结果这类信息汇总都是在END语句块中完成，它也是一个可选语句块。

  **pattern语句块** 中的通用命令是最重要的部分，它也是可选的。如果没有提供pattern语句块，则默认执行`{ print }`，即打印每一个读取到的行，awk读取的每一行都会执行该语句块。

  **示例**

```
echo -e "A line 1nA line 2" | awk 'BEGIN{ print "Start" } { print } END{ print "End" }'
Start
A line 1
A line 2
End
```

当使用不带参数的`print`时，它就打印当前行，当`print`的参数是以逗号进行分隔时，打印时则以空格作为定界符。在awk的print语句块中双引号是被当作拼接符使用，例如：

```
echo | awk '{ var1="v1"; var2="v2"; var3="v3"; print var1,var2,var3; }' 
v1 v2 v3
```

双引号拼接使用：

```
echo | awk '{ var1="v1"; var2="v2"; var3="v3"; print var1"="var2"="var3; }'
v1=v2=v3
```

{ }类似一个循环体，会对文件中的每一行进行迭代，通常变量初始化语句（如：i=0）以及打印文件头部的语句放入BEGIN语句块中，将打印的结果等语句放在END语句块中。

## awk内置变量（预定义变量）

FS:字段分隔符等同-F

```
awk -v FS=':' '{print $1,FS,$3}’ /etc/passwd 
awk –F: '{print $1,$3,$7}’ /etc/passwd
```

OFS：输出字段分隔符，默认为空白字符

```
awk -v FS=‘:’ -v OFS=‘:’ '{print $1,$3,$7}’ /etc/passwd
```

RS：输入记录分隔符，指定输入时的换行符

```
awk -v RS=' ' ‘{print }’ /etc/passwd 
```

ORS：输出记录分隔符，输出时用指定符号代替换行符

```
awk -v RS=' ' -v ORS='###'‘{print }’ /etc/passwd
```

NF：字段数量

```
awk -F：‘{print NF}’ /etc/fstab 引用变量时，变量前不需加$ 
awk -F：‘{print $(NF-1)}' /etc/passwd 
```

NR：行数记录号

```
awk ‘{print NR}’ /etc/fstab 
awk END‘{print NR}’ /etc/fstab
```

FNR：各文件分别计数,记录号

```
awk '{print FNR}' /etc/fstab /etc/inittab 
```

FILENAME：当前文件名

```
awk '{print FILENAME}’ /etc/fstab 
```

ARGC：命令行参数的个数

```
awk '{print ARGC}’ /etc/fstab /etc/inittab 
awk ‘BEGIN {print ARGC}’ /etc/fstab /etc/inittab 
```

ARGV：数组，保存的是命令行所给定的各参数

```
awk ‘BEGIN {print ARGV[0]}’ /etc/fstab /etc/inittab 
awk ‘BEGIN {print ARGV[1]}’ /etc/fstab /etc/inittab
```

FILENAME当前⽂件名FILENAME变量的使⽤

```
awk '{print FILENAME,FNR,$1}' /etc/fstab /root/awktest.txt
```

自定义变量量(区分字符大小写)
(1) -v var=value
(2) 在program中直接定义

## 将外部变量值传递给awk

```
awk -v test='hello gawk' '{print test}' /etc/fstab 
awk -v test='hello gawk' 'BEGIN{print test}' 
awk 'BEGIN{test="hello,gawk";print test}' 
awk -F:‘{sex=“male”;print $1,sex,age;age=18}’ /etc/passwd 
awk -F: -f awkscript script=“awk” /etc/passwd
```

## 查找进程pid

```
netstat -antup | grep 7770 | awk '{ print $NF NR}' | awk '{ print $1}'
```

## 操作符

比较操作符：
==, !=, >, >=, <=
模式匹配符：
~：左边是否和右边匹配，包含
!~：是否不匹配 示例：

```
awk -F: '$0 ~ /root/{print $1}‘ /etc/passwd 
awk '$0~“^root"' /etc/passwd 
awk '$0 !~ /root/‘ /etc/passwd 
awk -F: ‘$3==0’ /etc/passwd
```

逻辑操作符：
与&&，或||，非!
示例：

```
• awk -F: '$3>=0 && $3<=1000 {print $1}' /etc/passwd 
• awk -F: '$3==0 || $3>=1000 {print $1}' /etc/passwd 
• awk -F: ‘!($3==0) {print $1}' /etc/passwd 
• awk -F: ‘!($3>=500) {print $3}’ /etc/passwd 
```

### 运算级优先级表

!级别越高越优先
级别越高越优先

## awk高级输入输出

### 读取下一条记录

awk中`next`语句使用：在循环逐行匹配，如果遇到next，就会跳过当前行，直接忽略下面语句。而进行下一行匹配。next语句一般用于多行合并：

```
cat text.txt
a
b
c
d
e

awk 'NR%2==1{next}{print NR,$0;}' text.txt
2 b
4 d
```

当记录行号除以2余1，就跳过当前行。下面的`print NR,$0`也不会执行。下一行开始，程序有开始判断`NR%2`值。这个时候记录行号是`：2` ，就会执行下面语句块：`'print NR,$0'`

分析发现需要将包含有“web”行进行跳过，然后需要将内容与下面行合并为一行：

```
cat text.txt
web01[192.168.2.100]
httpd            ok
tomcat               ok
sendmail               ok
web02[192.168.2.101]
httpd            ok
postfix               ok
web03[192.168.2.102]
mysqld            ok
httpd               ok
0
awk '/^web/{T=$0;next;}{print T":t"$0;}' test.txt
web01[192.168.2.100]:   httpd            ok
web01[192.168.2.100]:   tomcat               ok
web01[192.168.2.100]:   sendmail               ok
web02[192.168.2.101]:   httpd            ok
web02[192.168.2.101]:   postfix               ok
web03[192.168.2.102]:   mysqld            ok
web03[192.168.2.102]:   httpd               ok
```

### 简单地读取一条记录

`awk getline`用法：输出重定向需用到`getline函数`。getline从标准输入、管道或者当前正在处理的文件之外的其他输入文件获得输入。它负责从输入获得下一行的内容，并给NF,NR和FNR等内建变量赋值。如果得到一条记录，getline函数返回1，如果到达文件的末尾就返回0，如果出现错误，例如打开文件失败，就返回-1。

getline语法：getline var，变量var包含了特定行的内容。

awk getline从整体上来说，用法说明：

- **当其左右无重定向符|或<时：** getline作用于当前文件，读入当前文件的第一行给其后跟的变量`var`或`$0`（无变量），应该注意到，由于awk在处理getline之前已经读入了一行，所以getline得到的返回结果是隔行的。

- 当其左右有重定向符`|`或`<`时：

   

  getline则作用于定向输入文件，由于该文件是刚打开，并没有被awk读入一行，只是getline读入，那么getline返回的是该文件的第一行，而不是隔行。

  **示例：**

执行linux的`date`命令，并通过管道输出给`getline`，然后再把输出赋值给自定义变量out，并打印它：

```
awk 'BEGIN{ "date" | getline out; print out }' test
```

执行shell的date命令，并通过管道输出给getline，然后getline从管道中读取并将输入赋值给out，split函数把变量out转化成数组mon，然后打印数组mon的第二个元素：

```
awk 'BEGIN{ "date" | getline out; split(out,mon); print mon[2] }' test
```

命令ls的输出传递给geline作为输入，循环使getline从ls的输出中读取一行，并把它打印到屏幕。这里没有输入文件，因为BEGIN块在打开输入文件前执行，所以可以忽略输入文件。

```
awk 'BEGIN{ while( "ls" | getline) print }'
```

### 关闭文件

awk中允许在程序中关闭一个输入或输出文件，方法是使用awk的close语句。

```
close("filename")
```

filename可以是getline打开的文件，也可以是stdin，包含文件名的变量或者getline使用的确切命令。或一个输出文件，可以是stdout，包含文件名的变量或使用管道的确切命令。

### 输出到一个文件

awk中允许用如下方式将结果输出到一个文件：

```
echo | awk '{printf("hello word!n") > "datafile"}'
或
echo | awk '{printf("hello word!n") >> "datafile"}'
```

## 控制语句：

```
{statements;...}：组合语句；
if(condition){statements;...}else {statements;...} 
if(condition1){statement1}else if(condition2){statement2} else{statement3} 
while(condition){statements;...} 
do {statements;...} while(condition)
for(expr1;expr2;expr3) {statements;...} 
break 
continue 
delete array[index] 
delete array 
exit
awk-F:'{$3>=500?usertype="commonuser":usertype="sysuser";printf"%-15s %-20s %10d\n",usertype,$1,$3}' /etc/passwd
```

### 数组的定义

数字做数组索引（下标）：

```
Array[1]="sun"
Array[2]="kai"
```

字符串做数组索引（下标）：

```
Array["first"]="www"
Array"[last"]="name"
Array["birth"]="1987"
```

使用中`print Array[1]`会打印出sun；使用`print Array[2]`会打印出kai；使用`print["birth"]`会得到1987。

**读取数组的值**

```
{ for(item in array) {print array[item]}; }       #输出的顺序是随机的
{ for(i=1;i<=len;i++) {print array[i]}; }         #Len是数组的长度
```

### 数组相关函数

**得到数组长度：**

```
awk 'BEGIN{info="it is a test";lens=split(info,tA," ");print length(tA),lens;}'
4 4
```

length返回字符串以及数组长度，split进行分割字符串为数组，也会返回分割得到数组长度。

```
awk 'BEGIN{info="it is a test";split(info,tA," ");print asort(tA);}'
4
```

asort对数组进行排序，返回数组长度。

**输出数组内容（无序，有序输出）：**

```
awk 'BEGIN{info="it is a test";split(info,tA," ");for(k in tA){print k,tA[k];}}'
4 test
1 it
2 is
3 a 
```

`for…in`输出，因为数组是关联数组，默认是无序的。所以通过`for…in`得到是无序的数组。如果需要得到有序数组，需要通过下标获得。

```
awk 'BEGIN{info="it is a test";tlen=split(info,tA," ");for(k=1;k<=tlen;k++){print k,tA[k];}}'
1 it
2 is
3 a
4 test
```

注意：数组下标是从1开始，与C数组不一样。

## 内置函数

awk内置函数，主要分以下3种类似：算数函数、字符串函数、其它一般函数、时间函数。

### 算术函数

| 格式            | 描述                                                         |
| :-------------- | :----------------------------------------------------------- |
| atan2( y, x )   | 返回 y/x 的反正切。                                          |
| cos( x )        | 返回 x 的余弦；x 是弧度。                                    |
| sin( x )        | 返回 x 的正弦；x 是弧度。                                    |
| exp( x )        | 返回 x 幂函数。                                              |
| log( x )        | 返回 x 的自然对数。                                          |
| sqrt( x )       | 返回 x 平方根。                                              |
| int( x )        | 返回 x 的截断至整数的值。                                    |
| rand( )         | 返回任意数字 n，其中 0 <= n < 1。                            |
| srand( [expr] ) | 将 rand 函数的种子值设置为 Expr 参数的值，或如果省略 Expr 参数则使用某天的时间。返回先前的种子值。 |

举例说明：

```
awk 'BEGIN{OFMT="%.3f";fs=sin(1);fe=exp(10);fl=log(10);fi=int(3.1415);print fs,fe,fl,fi;}'
0.841 22026.466 2.303 3
```

OFMT 设置输出数据格式是保留3位小数。

获得随机数：

```
awk 'BEGIN{srand();fr=int(100*rand());print fr;}'
78
awk 'BEGIN{srand();fr=int(100*rand());print fr;}'
31
awk 'BEGIN{srand();fr=int(100*rand());print fr;}'
41 
```

**格式化字符串输出（printf使用）**

格式化字符串格式：

其中格式化字符串包括两部分内容：一部分是正常字符，这些字符将按原样输出; 另一部分是格式化规定字符，以`"%"`开始，后跟一个或几个规定字符,用来确定输出内容格式。

| 格式 | 描述                     | 格式 | 描述                          |
| :--- | :----------------------- | :--- | :---------------------------- |
| %d   | 十进制有符号整数         | %u   | 十进制无符号整数              |
| %f   | 浮点数                   | %s   | 字符串                        |
| %c   | 单个字符                 | %p   | 指针的值                      |
| %e   | 指数形式的浮点数         | %x   | %X 无符号以十六进制表示的整数 |
| %o   | 无符号以八进制表示的整数 | %g   | 自动选择合适的表示法          |

```
awk -F: ‘{printf "%s",$1}’ /etc/passwd 
awk -F: ‘{printf "%s\n",$1}’ /etc/passwd 
awk -F: '{printf "%-20s %10d\n",$1,$3}' /etc/passwd 
awk -F:‘ {printf "Username: %s\n",$1}’ /etc/passwd 
awk -F: ‘{printf “Username: %s,UID:%d\n",$1,$3}’ /etc/passwd 
awk -F: ‘{printf "Username: %15s,UID:%d\n",$1,$3}’ /etc/passwd 
awk -F: ‘{printf "Username: %-15s,UID:%d\n",$1,$3}’ /etc/passwd
```

## 练习实例

输出当前所有用户

```
awk -F: '{printf "user:%d ^C\n",NR,$1}' /etc/passwd
```

ssh连接数

```
ss -nt|awk -F "[[:space:]]+|:" 'NR>1{print $(NF-2)}'
ss -nt|awk -F "[[:space:]]+|:" 'NR!=1{print $(NF-2)}'
```

uid大于1000用户

```
awk -F: '$3>=1000{print $1,$3}' /etc/passwd
```

本机ip地址

```
7
ifconfig ens33 |awk 'NR==2{print $2}'
6
ifconfig eth0 |awk -F"[[:space:]]+|:" 'NR==2{print $4}'
```

分区使用率

```
df|awk -F"[[:space:]]+|%" 'NR!=1{print $1,$5}'

df|awk -F"[[:space:]]+|%" 'NR!=1 && $5>10{print $1,$5}'
df|awk -F"[[:space:]]+|%" '/^\/dev\/sd/{print $1,$5}'
fstab
cat /etc/fstab |awk '!/^#|^$/{print $0}'
```

可登录用户

```
cat /etc/passwd |awk '!/nologin$/{print $0}'
cat /etc/passwd |awk '/nologin$/{print $0}'
```

⽂件ip_list.txt如下格式，请提取“.magedu.com”前⾯的主机名部分 并写⼊到该⽂件中：

```
1 blog.magedu.com 
2 www.magedu.com 
... 
999 study.magedu.com
awk -F'[" ".]' '{print $2}' ip_list.txt >> ip_list.txt
```

统计/etc/fstab⽂件中每个⽂件系统类型出现的次数？

```
awk '/^[UUID\/]/{fs[$3]++}END{for(i in fs){print i,fs[i]}}' /etc/fstab 
```

统计/etc/fstab⽂件中每个单词出现的次数？

```
awk '{for(i=1;i<=NF;i++){word[$i]++}}END{for(j in word){print j,word[j]}}' /etc/fstab 
```

提取出字符串Yd$C@M05MB%9Bdh7dq+YVixp3vpw中的所有数 字？

```
echo "yd$C@M05MB%9&Bdh7dq+yVixp3vpw" | awk 'gsub(/[^[:digit:]]/," ",$0)' 
```

有⼀⽂件记录了1-100000之间的随机的整数共5000个，存储的格 式，100,50,35,89，。。。请取出其中最⼤和最⼩的整数？

```
]# (echo -e "$RANDOM\c";for((i=1;<100;i++));do echo -e “,$RANDOM\c”;done)i>num.txt
]#awk -F "," '{min=$1;max=$1;for(i=1;i<=NF;i++){if($i>=max){max=$i} else if(min>=$i)min=$i}print min,max}' num.txt
```

解决DOS***⽣产案例：根据web⽇志或⽹络连接数，监控当某个ip 并发连接数或者短时间内pv达到100，即调⽤防⽕墙命令封掉对应的ip，控频率每隔5分钟；防⽕墙命令为iptables -A INPUT -s IP -j REJECT？

```
vim deny_dos.sh 
while true ;do 
awk '/^[0-9]/{IP[$1]++}END{for(i in IP){if (IP[i]>=100)print i}}' 
/var/log/httpd/access_log | while read line;do iptables -A INPUT -s $line -j REJECT;done 
sleep 300
done
```

将以下⽂件内容中FQDN取出并根据其进⾏计数从⾼到低排序：

```
http://mail.magedu.com/index.html
http://www.magedu.com/test.html
http://study.magedu.com/index.html
http://blog.magedu.com/index.html
http://www.magedu.com/images/logo.jpg
http://blog.magedu.com/20080102.html
awk -F "[/]+" '{html[$2]++}END{for(i in html)print i,html[i]}' html.txt
答：awk -F “/” ‘{print $3}’|sort|uniq -c|sort -nr
```

将以下⽂本以inode为标记，对inode相同的counts进⾏累加，并且统 计出 同一inode中，beginnumber的最小值和endnumber的最大值

```
inode|beginnumber|endnumber|counts|
106|3363120000|3363129999|10000|
106|3368560000|3368579999|20000|
310|3337000000|3337000100|101|
310|3342950000|3342959999|10000|
310|3362120960|3362120961|2|
311|3313460102|3313469999|9898|
311|3313470000|3313499999|30000|
311|3362120962|3362120963|2|
输出的结果格式为：
 310|3337000000|3362120961|10103|
 311|3313460102|3362120963|39900|
 106|3363120000|3368579999|30000| 
````
 ````
awk -F'|' -v OFS='|' '/^[0-9]/{inode[$1]++; if(!bn[$1]){bn[$1]=$2}else if(bn[$1]>$2){bn[$1]=$2}; if(en[$1]
```

\#######################

```
⽂件datafile内容如下： 
Mike Harrington:[510] 548-1278:250:100:175
Christian Dobbins:[408] 538-2358:155:90:201
Suan Dalsass:[206] 654-6279:250:60:50
Archie McNichol:[206] 548-1348:250:100:175
Jody Savage:[206] 548-1278:15:188:150
Guy Quigley:[916] 343-6410:250:100:175
Dan Saveage:[406] 298-7744:450:300:275
Nancy McNeil:[206] 548-1278:250:80:75
John Goldenrod:[916] 348-4278:250:100:175
Chet Main:[510] 548-5258:50:95:135
Tom Savage:[408] 926-3456:250:168:200
Elizabeth Stachelin:[916] 440-1763:175:75:300
```

上面的数据表中包含名字、电话号码和过去三个月里的捐款,下面分别用awk解答：

```
1)显示所有电话号码？ 
答：awk -F ":" '{print $2}' datafile.txt 
2)显示Dan的电话号码？ 
答：awk -F ":" '/^Dan/{print $2}' datafile.txt 
3)显示Suan的捐款和电话？ 
答：awk -F ":" '/^Suan/{print $3,$4,$5,$2}' datafile.txt 
4)显示所有以D开头的姓？ 
答：awk -F ":" '{print $1}' datafile.txt | awk '{print $2}'|awk '/^D/' 
5)显示所有以一个C或E开头的名？ 
答：awk -F ":" '{print $1}' datafile.txt | awk '{print $1}'|awk '/^[CE]/'
6)显示所有只有四个字符的名？ 
答：awk -F ":" '{print $1}' datafile.txt | awk '{if(length($1)==4)print $1}' 
7)显示所有区号为916的人名？ 
答：awk -F ":" '/916/{print $1}' datafile.txt
8)显示Mike的捐款，显示每个值时都有$开头，如$250$100$175？ 
答：awk -F ":" '/^Mike/{print "$"$3"$"$4"$"$5}' datafile.txt 
9)显示姓，其后跟一个逗号和名，如Jody，Savage？ 
答：awk -F ":" '{print $1}' datafile.txt | awk '{print $2,",",$1}' 
10)写一个awk的脚本，作用： 
（1）显示Savage的全名和电话号码 
（2）显示Cher的捐款 
（3）显示所有头一个月捐款$250的人名 
答：vim awk #!/bin/awk -f 
BEGIN{FS=":"} 
{if($1 ~/Savage/) print $1":"$2}
{if($1~/^Chet/) print "$"$3":""$"$4":""$"$5}
{if($3==250) print $1} 
awk -f awk datafile.txt
```

⽂件名AccQryFree2016.log，存在⽬录/root/boss/log/下，内容：

```
12:00:00 service start query_value,exited with value 0;
12:00:01 select * from crm_user where sts=1;exited with value 0;
12:10:01 use db:db_ngboss[srv_zw1] 12:20:00 service start query_value,exited with value 0;
12:22:01 select * from crm_user where sts=1;exited with value 0;
12:23:01 use db:db_ngboss[srv_zw1]
1)统计出文件中字符串exited with value 0出现的次数？ 
答：grep -o "exited with value 0" AccQryFree2016.log|uniq -c 
2)使用vi编辑器，将文件中exited with value 0替换为EXITED WITH VALUE 0？ 
答：vim /root/boss/log/AccQryFree2016.log :%s/exited with value 0/EXITED WITH VALUE 0/g 
3)写出带有exited with value 0的时间列全部输出？ 
答：grep "exited with value 0" AccQryFree2016.log|awk -F " " ' {print $2}' 
4)把该文件压缩？ 
答：tar jcvf AccQryFree2016.log.tar.bz2 /root/boss/log/AccQryFree2016.log
5)配置一个定时任务，将该文件在每周五下午6点进行删除？ 
答：crontab -e 0 18 * * 5 rm -rf /root/boss/log/AccQryFree2016.log
```

使⽤netstat和awk统计服务器出现tcp⽹络状态并按数量排序？

```
netstat -tan |grep "^tcp\b" |awk '{print $5}' |sort | uniq -c| sort -nr
```

实现查询⽂件file1⾥⾯空格开始的所在的⾏号？

```
答：grep -n -E '^[[:sapce:]] ' file1|awk -F: '{print $1}'
```

使⽤awk命令，计算⼀个⽬录下⽂件⼤⼩的总和？

```
答：ll |awk 'BEGIN{sum=0}{sum=sum+$5}END{print sum}'
```

使⽤awk统计当前主机的并发访问量？

```
答：netstat -tan | awk '/^tcp/ {++state[$NF]} END {for(key in state) print key,"\t",state[key]}' 显示结果： 
LAST_ACK 1 SYN_RECV 14 
ESTABLISHED 79 
FIN_WAIT1 28 
FIN_WAIT2 3 
CLOSING 5 
TIME_WAIT 1669 
解释： 
CLOSED：无连接是活动的或正在进行 
LISTEN：服务器在等待进入呼叫 
SYN_RECV：一个连接请求已经到达，等待确认 
SYN_SENT：应用已经开始，打开一个连接 
ESTABLISHED：正常数据传输状态 
FIN_WAIT1：应用说它已经完成 
FIN_WAIT2：另一边已同意释放 
ITMED_WAIT：等待所有分组死掉 
CLOSING：两边同时尝试关闭 
TIME_WAIT：另一边已初始化一个释放 
LAST_ACK：等待所有分组死掉
统计apache访问⽇志流量排名前10个ip？ 
答： 
cat access_log | awk ’{print $1}’ | sort | uniq -c | sort -n -r | head -10
解法2： 
awk ‘{a[$1] += 1;} END {for (i in a) printf(“%d %s\n”, a[i], i);}’ 日志文 件 | sort -n | tail
```

使⽤netstat -an输出格式，请编写脚本，统计输出连接到本地主机 数最多的10个ip，并按连接数从多到少排序？
tcp 0 52 172.18.118.155:22 172.18.116.232:49916 ESTABLISHED
答：

```
#!/bin/sh 
top10ip=`ss -nt | grep 'ESTAB' | awk '{print $5}' | cut -d: -f1 | grep "^[[:digit:]]\+.*" | sort | uniq -c | sort -rnk1 | awk '{print $2,"\t",$1}' | head` 
```

echo “连接到本地主机最多的10个ip是：$top10ip”nginx的access.log⽇志如下，⽤shell实现，将状态码为200的请求 的ip访问排名前10个列出来：

```
172.18.116.232 - - [18/May/2018:00:20:29 -0400] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.117 Safari/537.36" "-"
172.18.116.232 - - [18/May/2018:00:20:29 -0400] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.117 Safari/537.36" "-"
答：awk '($9 ~ /200/)' access.log | awk '{print $9,$7}' | sort -nr |head -n10
```

统计apaceh的access.log中访问量最多的5个ip？

```
172.18.116.232 - - [21/May/2018:05:29:11 -0400] "GET /favicon.ico HTTP/1.1" 404 209 "http://172.18.118.155:8080/" "Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.117 Safari/537.36"
答： cat access_log | awk ’{print $1}’ | sort | uniq -c | sort -nr | head -5
```

awk实现打印奇数⾏和偶数⾏

```
偶数seq 10|awk '!(i=!i)'
奇数seq 10|awk 'i=!i'
等同于sed命令：
seq 10 | sed -n '1~2p'
seq 10 | sed -n '2~2p'
```

匹配以f开头的⾏开始，到r开头的⾏结束之间的所有⾏

```
awk '/^f/,/^r/' /etc/passwd
```

查找netstat -nt命令的结果中Foreign Address列的地址，统计每个地址链接的次数，如果⼤于2次， 显⽰ip

```
netstat -nt |awk '/^tcp/{print $5}'|awk -F: '{print $1}'|sort |uniq -c|awk '$1>1{print $2}' 172.18.116.232
```