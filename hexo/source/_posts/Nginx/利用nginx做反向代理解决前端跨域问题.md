---
title: 利用nginx做反向代理解决前端跨域问题
date: 2020-03-09 14:17:03
tags: 
- 跨域
- nginx
categories: nginx
---

云开发机连公司本地机器调接口提示跨域

<!--more-->

1.在安装了nginx的服务器中找到nginx.conf文件里的server{},如果没有找到的话就到该文件同级的conf.d文件夹里面的default.conf文件.

2.在里面添加如下代码

```
server
{
    listen 80;
    server_name www.aaa.top;
    location / {
        proxy_pass http://www.bbb.com;
        add_header 'Access-Control-Allow-Origin' '*'; 
        add_header 'Access-Control-Allow-Credentials' 'true'; 
    }
    ##### other directive
}
```

　其中www.aaa.com代表自己的域名,www.bbb.com代表的别人的域名,就是需要跨域的域名,然后添加上允许跨域的请求头,然后重启nginx就可以了.

这样的话请求www.aaa.com的接口就相当于请求www.bbb.com的接口了.

以上就是利用nginx做反向代理解决跨域的方法.

## 解决js跨域使用nginx配置问题

```
#解决跨域问题
add_header Access-Control-Allow-Origin *;
add_header Access-Control-Allow-Credentials true;
add_header Access-Control-Allow-Headers Origin,X-Requested-With,Content-Type,Accept,x-language;
add_header Access-Control-Allow-Methods POST,GET,OPTIONSZ,PUT,DELETE;
```