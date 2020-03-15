---
title: ANSIBLE自动化工具
date: 2020-03-09 21:12:02
tags:
- ansible
categories:
- 自动化
---

ANSIBLE自动化工具

<!--more-->

```shell
rpm包安装：EPEL源
配置文件
    /etc/ansible/hosts          #管理主机的清单
    /etc/ansible/roles/         #存放角色的目录
    /etc/ansible/ansible.cfg    #主配置文件，配置ansible工作特性，一般默认就好
程序
    /usr/bin/ansible            #主程序，临时命令执行工具
    /usr/bin/ansible-doc        #查看配置文档，模块功能查看工具
    /usr/bin/ansible-galaxy     #下载/上传优秀代码或Roles模块的官网平台
    /usr/bin/ansible-playbook   #定制自动化任务，编排剧本工具/usr/bin/ansible-pull 远程执行命令的工具
    /usr/bin/ansible-vault      #文件加密工具
    /usr/bin/ansible-console    #基于Console界面与用户交互的执行工具

ansible命令

ansible-doc:显示模块帮助
    ansible-doc [options][module]
    -a      显示所有模块的文档
    -l      列出可用模块
    -s      显示指定模块的playbook片段
例：    ansible-doc ping   
        ansible-doc -l
        ansible-doc -s ping

ansible
    --version       #显示版本
    -m module       #指定模块，默认command
    -v              #显示详细过程 -vv -vvv
    --list          #显示主机列表，
    -C              #检查，并不执行
    all             #表示所有清单列表的主机 ansible all -m ping
    *               #通配符 ansible "*" -m ping      ansible 192.168.2.* -m ping
    :               #逻辑或 ansible "web1:web2" --list
    :&              #逻辑与 ansible "web1:&web2" -m ping
    :!   用单引号    #逻辑非 ansible 'web1:&web2' --list

https://galaxy.ansible.com
ansible-galaxy list                         #列出所有已安装的galaxy
ansible-galaxy install geerlingguy.redis    #下载安装galaxy
ansible-galaxy remove geerlingguy.redis     #删除galaxy
ansible-pull                                #推送至远程，提升效率
ansible-playbook

ansible-vault
功能：管理加密解密yml文件
ansible-vault [create|decrypt|edit|encrypt|rekey|view]
ansible-vault encrypt hello.yml             #加密
ansible-vault decrypt hello.yml             #解密
ansible-vault view hello.yml                #查看
ansible-vault edit hello.yml                #编辑加密文件
ansible-vault rekey hello.yml               #修改口令
ansible-vault create new.yml                #创建新文件

ansible常用模块
command:在远程主机执行简单命令（默认是command，可以不用m选项）
[root@localhost ~]# ansible web1 -a 'cat /etc/issue'
[root@localhost ~]# ansible web1 -a 'ls -l /etc/selinux'

shell：调用bash执行复杂命令（万能模块）
[root@localhost ~]# ansible web1 -m shell -a 'sed -i "s/SELINUX=enforcing/SELINUX=disabled/"  /etc/selinux/config
[root@localhost ~]# ansible web1 -a 'echo $HOSTNAME'
[root@localhost ~]# ansible web1 -m shell -a 'tar -Jcvf /root/boot.tar.xz /boot/'

script：在远程主机上运行ansible服务器上的脚本
[root@localhost ~]# ansible web1 -m script -a '/data/hello.sh'
copy：从主控端复制文件到远程主机
[root@localhost ~]# ansible-doc -s copy
[root@localhost ~]# ansible web1 -m copy -a ' src=/etc/selinux/config dest=/etc/selinux/config.bak mode=600 owner=huahua group=bin'
[root@localhost ~]# ansible web1 -m copy -a ' src=/etc/selinux/config dest=/etc/selinux/ backup=yes' #默认覆盖，加入backup=yes备份。
[root@localhost ~]# ansible web1 -m copy -a 'content="111\n222\n333" dest=/tmp/text.txt' #content指定内容，直接生成目标文件。
[root@localhost ~]# ansible web1 -m copy -a 'content="[base]\nname=base\nbaseurl=file:///mnt/cdrom\ngpgcheck=0" dest=/etc/yum.repos.d/base.repo'   #批量创建yum源 

fetch：从远程主机提取文件至主控端，copy相反，目录的话需要tar打包
[root@localhost ~]# ansible-doc -s fetch
[root@localhost ~]# ansible web1 -m fetch -a 'src=/etc/yum.repos.d/base.repo dest=/tmp/'            #将远程base.repo文件抓取放到本机tmp目录下
file：设置文件属性
[root@localhost ~]# ansible-doc -s file
[root@localhost ~]# ansible web1 -m file -a 'path=/tmp/yum.log owner=huahua mode=000'        
[root@localhost ~]# ansible web1 -m file -a 'src=/tmp/yum.log name=/tmp/yum.log.link state=link' 
         #创建软连接  
[root@localhost ~]# ansible web1 -m file -a 'src=/tmp/yum.log name=/tmp/yum.log.hard state=hard' 
         #创建硬链接
[root@localhost ~]# ansible web1 -m file -a 'path=/tmp/dir1 state=directory'     #创建文件夹
[root@localhost ~]# ansible web1 -m file -a 'path=/tmp/f1.log state=touch'       #创建空文件
[root@localhost ~]# ansible web1 -m file -a 'path=/tmp/f1.log state=absent'      #删除文件（目录）
[root@localhost ~]# ansible web1 -m shell -a 'rm -rf /tmp/*'

hostname：管理主机名
[root@localhost ~]# ansible-doc -s hostname
[root@localhost ~]# ansible 192.168.2.20 -m hostname -a 'name=centos7.6'    

cron：计划任务
[root@localhost ~]# ansible-doc -s cron
[root@localhost ~]# ansible web2 -m cron -a 'name=synctime minute=*/5 \
job="/usr/sbin/ntpdate 192.168.2.10 &> /dev/null"'
[root@localhost ~]# ansible web2 -a 'crontab -l'

yum：管理包
[root@localhost ~]# ansible-doc -s yum
[root@localhost ~]# ansible web1 -m yum -a 'name=httpd state=present'
[root@localhost ~]# ansible web1 -m yum -a 'name=httpd state=absent'

service：管理服务
[root@localhost ~]# ansible-doc -s service
[root@localhost ~]# ansible web1 -m service -a 'name=named state=started enabled=true'
[root@localhost ~]# ansible web1 -m service -a 'name=named state=stopped'

user：管理用户
[root@localhost ~]# ansible-doc -s user
[root@localhost ~]# ansible web1 -a 'getent passwd'
[root@localhost ~]# ansible web1 -m user -a 'name=mysql system=yes shell=/sbin/nologin'
[root@localhost ~]# ansible web1 -m user -a 'name=mysql state=absent'
[root@localhost ~]# ansible web1 -m user -a 'name=mysql state=absent remove=yes'  

YAML语言
1、第一行写“---”       最后一行“...”     (建议不要省略)
2、第二行建议写明功能用#注释
3、缩进必须是统一的，不能空格和tab混用
4、缩进的级别也必须是一致的，同样的缩进代表同样的级别，程序判断配置的级别是通过缩进结合换行来实现的
5、YAML文件内容是区分大小写的,k/v的值均大小写敏感
6、一个完整的代码块功能需要最少元素需包括name和task
7、一个name只能包括一个task
8、YAML文件扩展名通常为yml和yaml

List：列表，所有元素均使用“-”打头

Dictionary：字典，由多个key和value组成
            ksy:value

playbook的核心元素
hosts:playbook配置文件作用的主机
tasks：任务列表
variables：变量
templates：包含模板语法的文本文件
handlers：由特定条件触发的任务
roles：用于层次性、结构化地组织playbook。roles能够根据层次结构自动装载变量文件、tasks以及handlers

运行playbook的方式
ansible-playbook  ... [options]
常见选项
--check -C          #只检测可能会发生的改变，但不真正执行操作
--list-hosts        #列出运行任务的主机
--list-tags         #列出tag
--list-tasks        #列出task
--limit             #主机列表 只针对主机列表中的主机执行
-v -vv -vvv         #显示过程

[root@localhost ansible]# vim http.yml
---
#install httpd
- hosts: web1
  remote_user: root

  tasks:
    - name: install package
      yum: name=httpd
    - name: cofig file
      copy: src=/tmp/httpd.conf dest=/etc/httpd/conf/ backup=yes
    - name: service
      service: name=httpd state=started enabled=yes

[root@localhost ansible]# ansible-playbook -C http.yml
[root@localhost ansible]# ansible-playbook http.yml

触发handlers（handlers有notify触发）
---
#install httpd
- hosts: web1
  remote_user: root

  tasks:
    - name: install package
      yum: name=httpd
    - name: cofig file
      copy: src=/tmp/httpd.conf dest=/etc/httpd/conf/ backup=yes
      notify: restart service
    - name: service
      service: name=httpd state=started enabled=yes

  handlers:
    - name: restart service
      service: name=httpd state=restarted

[root@localhost ansible]# ansible-playbook http.yml

tags标签（根据tags来实现部分功能）
---
#install httpd
- hosts: web1
  remote_user: root

  tasks:
    - name: install package
      yum: name=httpd
    - name: cofig file
      copy: src=/tmp/httpd.conf dest=/etc/httpd/conf/ backup=yes
      notify: restart service
      tags: config
    - name: service
      service: name=httpd state=started enabled=yes
      tags: service

  handlers:
    - name: restart service
      service: name=httpd state=restarted

[root@localhost ansible]# ansible-playbook -t config http.yml
[root@localhost ansible]# ansible-playbook -t config,service http.yml       #选择多个标签

ansible初步准备
[root@localhost ~]# yum -y install ansible
[root@localhost ~]# vim /etc/ansible/hosts      #加入清单列表
[web1]
192.168.2.20
192.168.2.30

[web2]
192.168.2.30
192.168.2.40

[root@localhost ~]# vim /etc/ansible/ansible.cfg 
log_path = /var/log/ansible.log             #开启日志
module_name = shell                         #修改默认模块
host_key_checking = False                   #取消对应服务器host_key的检查

基于key验证，实现无秘钥登录管理
[root@localhost ~]# ssh-keygen
[root@localhost ~]# ssh-copy-id 192.168.2.20
[root@localhost ~]# ssh-copy-id 192.168.2.30
[root@localhost ~]# ssh-copy-id 192.168.2.40

测试连通
[root@localhost ~]# ansible web1 -m ping
192.168.2.20 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
192.168.2.30 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}

[root@localhost ~]# ansible web2 -m ping
192.168.2.30 | SUCCESS => {    "changed": false, 
    "ping": "pong"
}
192.168.2.40 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}

playbook变量使用
[root@localhost ansible]# ansible all -m setup  #查看所有变量

[root@localhost ansible]# ansible-playbook -e port=6869 file.yml        #命令行指定变量，优先级最高
ansible_hostname
ansible_memtotal_mb

#调用ansible_hostname变量
---
# file var
 - hosts: web1
   remote_user: root

   tasks:
     - name: file
       file: name=/tmp/{{ansible_hostname}}.log state=touch     

在清单里定义变量port和mark
[root@localhost ansible]# vim /etc/ansible/hosts
[web2]
192.168.2.30 port=80
192.168.2.40 port=8080
[web2:vars]                 
mark="_"

#调用变量
---
# file var
 - hosts: web1
   remote_user: root

   tasks:
     - name: file
       file: name=/tmp/{{ ansible_hostname }}{{ mark }}{{ port }}.log state=touch

在playbook定义变量
---
# file var
 - hosts: web1
   remote_user: root
   vars_files:
     - vars.yml                 #调用vars.yuml变量文件
---
# file var
 - hosts: web1
   remote_user: root
   vars:
     - port: 1989               #文件内定义

模板template
文本文件，嵌套有脚本（使用模板编程语言编写）
Jinja2语言，使用字面量，有下面形式
    字符串：使用单引号或双引号
    数字：整数，浮点数
    列表：[item1, item2, ...]
    元组：(item1, item2, ...)
    字典：{key1:value1, key2:value2, ...}
    布尔型：true/false
算术运算：+, -, *, /, //, %, **
比较操作：==, !=, >, >=, <, <=
逻辑运算：and，or，not
流表达式：For，If，When

template功能：根据模块文件动态生成对应的配置文件
    template文件必须存放于templates目录下，且命名为 .j2 结尾
    yaml/yml 文件需和templates目录平级，目录结构如下：
 ./
 ├── temnginx.yml
 └── templates
 └── nginx.conf.j2

---
#nginx 
- hosts: web2
  remote_user: root

  tasks:
    - name: package
      yum: name=nginx
    - name: config
      template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
      notify: restart
    - name: service
     service: name=nginx state=started enabled=yes

   handlers:
     - name: restart
       service: name=nginx state=restarted

[root@localhost ansible]# tree
.
├── nginx.yml
└── templates
    └── nginx.conf.j2  
when条件判断
---
#install httpd
- hosts: web1
  remote_user: root

  tasks:
    - name: install package
      yum: name=httpd
    - name: config file
      template: src=templates/httpd.conf6.j2 dest=/etc/httpd/conf/httpd.conf
      when: ansible_distribution_major_version == "6"
      notify: restart service
    - name: config file
      template: src=templates/httpd.conf7.j2 dest=/etc/httpd/conf/httpd.conf
      when: ansible_distribution_major_version == "7"
    - name: service
      service: name=httpd state=started enabled=yes
      tags: service

  handlers:
    - name: restart service
      service: name=httpd state=restarted

迭代：with_items
---
# file var
 - hosts: web1
   remote_user: root

   tasks:
     - name: file
       file: name=/tmp/{{item}}.log state=touch
       with_items:
         - abc
         - qwe
         - 798

---
- hosts: web1
  remote_user: root

  tasks:
    - name: create user
      user: name={{item}}
      with_items:
        - huahua
        - lili
        - yangyang

---
- hosts: web1
  remote_user: root

  tasks:
    - name: create group
      group: name={{item}}
      with_items:
        - agroup
        - bgroup
        - cgroup

    - name: create user
      user: name={{item.name}} group={{item.group}}
      with_items:
        - {name: "huahua",group: "agroup"}
        - {name: "lili",group: "bgroup"}
        - {name: "yangyang",group: "cgroup"}

template for if
1
[root@localhost templates]# pwd
/tmp/ansible/templates
[root@localhost templates]# vim test.j2         #模板文件
{%for i in ports  %}
server{
    listen  {{i}}
    server_name www.a.com
    root        /app/log/
}
{%endfor%}

[root@localhost ansible]# pwd
/tmp/ansible
[root@localhost ansible]# vim test.yml      #YAML文件调用
---
- hosts: web1
  remote_user: root
  vars:
    ports:
      - 81
      - 82
      - 83

  tasks:
    - name: template
      template: src=test.j2 dest=/tmp/test.conf

[root@localhost ansible]# ansible-playbook -C test.yml
[root@localhost ansible]# ansible-playbook test.yml 
[root@localhost ansible]# ansible web1 -a 'cat /tmp/test.conf'
192.168.2.20 | CHANGED | rc=0 >>
server{
    listen  81
    server_name www.a.com
    root        /app/log/
}
server{
    listen  82
    server_name www.a.com
    root        /app/log/
}
server{
    listen  83
    server_name www.a.com
    root        /app/log/
}
...

2
[root@localhost templates]# pwd
/tmp/ansible/templates
[root@localhost templates]# vim test2.j2 
{%for i in vhosts  %}
server{
    listen  {{i.port}}
    server_name {{i.name}}
    root     {{i.dir}}
}
{%endfor%}

[root@localhost ansible]# pwd
/tmp/ansible
[root@localhost ansible]# vim test2.yml
---
- hosts: web1
  remote_user: root
  vars:
    vhosts:
      - web1:
        port: 81
        name: www.a.com
        dir: /tmp/webs
      - web1:
        port: 82
        name: www.b.com
        dir: /tmp/webs
      - web1:
        port: 83
        name: www.c.com
        dir: /tmp/webs

  tasks:
    - name: template
      template: src=test2.j2 dest=/tmp/test2.conf

[root@localhost ansible]# ansible-playbook -C test2.yml
[root@localhost ansible]# ansible-playbook test2.yml
[root@localhost ansible]# ansible web1 -a "cat /tmp/test2.conf"
192.168.2.30 | CHANGED | rc=0 >>
server{
    listen  81
    server_name www.a.com
    root     /tmp/webs
}
server{
    listen  82
    server_name www.b.com
    root     /tmp/webs
}
server{
    listen  83
    server_name www.c.com
    root     /tmp/webs
}
...

3
[root@localhost templates]# pwd
/tmp/ansible/templates
[root@localhost templates]# vim test3.j2
{%for i in vhosts  %}
server{
    listen  {{i.port}}
{% if i.name is defined %}
    server_name  {{i.name}}
{% endif %}
    root  {{i.dir}}
}

{%endfor%}

[root@localhost ansible]# pwd
/tmp/ansible
[root@localhost ansible]# vim test3.yml
---
- hosts: web1
  remote_user: root
  vars:
    vhosts:
      - web1:
        port: 81
        # name: www.a.com
        dir: /tmp/webs
      - web1:
        port: 82
        name: www.b.com
        dir: /tmp/webs
      - web1:
        port: 83
        #name: www.c.com
        dir: /tmp/webs

  tasks:
    - name: template
      template: src=test3.j2 dest=/tmp/test3.conf

[root@localhost ansible]# ansible-playbook -C test3.yml
[root@localhost ansible]# ansible-playbook test3.yml 
[root@localhost ansible]# ansible web1 -a 'cat /tmp/test3.conf'
192.168.2.30 | CHANGED | rc=0 >>
server{
    listen  81
    root  /tmp/webs
}

server{
    listen  82
    server_name  www.b.com
    root  /tmp/webs
}

server{
    listen  83
    root  /tmp/webs
}

Roles角色
/roles/project/ :项目名称,有以下子目录

  files/ ：存放由copy或script模块等调用的文件

  templates/：template模块查找所需要模板文件的目录

  tasks/：定义task,role的基本元素，至少应该包含一个名为main.yml的文件；
其它的文件需要在此文件中通过include进行包含

  handlers/：至少应该包含一个名为main.yml的文件；其它的文件需要在此
文件中通过include进行包含

  vars/：定义变量，至少应该包含一个名为main.yml的文件；其它的文件需要
在此文件中通过include进行包含

  meta/：定义当前角色的特殊设定及其依赖关系,至少应该包含一个名为
main.yml的文件，其它文件需在此文件中通过include进行包含

  default/：设定默认变量时使用此目录中的main.yml文件

创建role的步骤
  (1) 创建以roles命名的目录
  (2) 在roles目录中分别创建以各角色名称命名的目录，如webservers等
  (3) 在每个角色命名的目录中分别创建files、handlers、meta、tasks、
templates和vars目录；用不到的目录可以创建为空目录，也可以不创建
  (4) 在playbook文件中，调用各角色

安装httpd
目录结构
[root@localhost ansible]# tree
.
├── role-httpd.yml
└── roles
    └── httpd
        ├── files
        │   ├── httpd.conf
        │   └── index.html
        └── tasks
            ├── conf.yml
            ├── data.yml
            ├── install.yml
            ├── main.yml
            └── service.yml

[root@localhost tasks]# cat conf.yml
- name: config
  copy: src=httpd.conf dest=/etc/httpd/conf/httpd.conf

[root@localhost tasks]# cat data.yml 
- name: copy data file
  copy: src=index.html dest=/var/www/html/index.html

[root@localhost tasks]# cat install.yml 
- name: install package
  yum: name=httpd

[root@localhost tasks]# cat service.yml 
- name: service
  service: name=httpd state=started enabled=yes

[root@localhost tasks]# cat main.yml 
- include: install.yml
- include: conf.yml
- include: data.yml
- include: service.yml

[root@localhost ansible]# cat role-httpd.yml
---
#test httpd role
- hosts: web1

  roles:
    - role: httpd

[root@localhost ansible]# ansible-playbook role-httpd.yml

nginx安装
目录结构
[root@localhost ansible]# tree
.
├── role-httpd.yml
├── role-nginx.yml
└── roles
    ├── httpd
    │   ├── files
    │   │   ├── httpd.conf
    │   │   └── index.html
    │   └── tasks
    │       ├── conf.yml
    │       ├── data.yml
    │       ├── install.yml
    │       ├── main.yml
    │       └── service.yml
    └── nginx
        ├── files
        │   └── index.html
        ├── handlers
        │   └── main.yml
        ├── tasks
        │   ├── config.yml
        │   ├── data.yml
        │   ├── group.yml
        │   ├── install.yml
        │   ├── main.yml
        │   ├── service.yml
        │   └── user.yml
        ├── templates
        │   └── nginx.conf.j2
        └── vars
            └── main.yml

[root@localhost handlers]# cat main.yml
- name: restart service
  service: name=nginx state=restarted

[root@localhost tasks]# cat config.yml 
- name: config
  template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
  notify: restart service

[root@localhost tasks]# cat data.yml 
- name: copy data file
  copy: src=index.html dest=/usr/share/nginx/html/index.html

[root@localhost tasks]# cat group.yml 
- name: group
  group: name=nginx system=yes gid=77 

[root@localhost tasks]# cat user.yml 
- name: user
  user: name=nginx system=yes uid=77 group=nginx

[root@localhost tasks]# cat install.yml 
- name: install
  yum: name=nginx 

[root@localhost tasks]# cat service.yml 
- name: service
  service: name=nginx state=started enabled=yes

[root@localhost nginx]# cat tasks/main.yml
- include: group.yml
- include: user.yml
- include: install.yml
- include: config.yml
- include: data.yml
- include: service.yml

[root@localhost ansible]# cat role-nginx.yml 
---
#test nginx role
- hosts: web2

  roles:
    - role: nginx

tags标签和when判断
---
#test httpd role
- hosts: web1:web3

  roles:
    - role: httpd
      tags: web
      when: ansible_distribution_major_version == "6"
    - role: nginx
      tags: web2
      when: ansible_distribution_major_version == "7"

[root@localhost ansible]# ansible-playbook -t web1 role-httpd-nginx.yml
```