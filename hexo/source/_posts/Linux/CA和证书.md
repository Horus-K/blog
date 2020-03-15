---
title: CA和证书
date: 2020-03-10 16:52:58
tags:
- ca
- 证书
categories: 系统
---

# 企业内网搭建CA服务器生成自签名证书，CA签署，实现企业内网基于key验证访问服务器

<!--more-->

### 一些CA基础

> - PKI：Public Key Infrastructure
>   签证机构：CA（Certificate Authority）
>   注册机构：RA
>   证书吊销列表：CRL
> - X.509：定义了证书的结构以及认证协议标准
>   版本号 主体公钥
>   序列号 CRL分发点
>   签名算法 扩展信息
>   颁发者 发行者签名
>   有效期限
>   主体名称
> - 证书类型：
>   证书授权机构的证书
>   服务器
>   用户证书
>   获取证书两种方法：
>   1)使用证书授权机构
>   生成证书请求（csr）
>   2)将证书请求csr发送给CA
>   CA签名颁发证书
>   自签名的证书
>   自已签发自己的公钥

## 证书作用

* 获取证书后，例如网站流量将基于HTTPS 协议
* HTTPS 协议：就是“HTTP 协议”和“SSL/TLS 协议”的组合。HTTP over
* SSL”或“HTTP over TLS”，对http协议的文本数据进行加密处理后，成为二
进制形式传输
SSL：Secure Socket Layer，TLS: Transport Layer Security
1995：SSL 2.0 Netscape
1996：SSL 3.0
1999：TLS 1.0
2006：TLS 1.1 IETF(Internet工程任务组) RFC 4346
2008：TLS 1.2 当前使用
2015：TLS 1.3
功能：机密性，认证，完整性，重放保护
* HTTPS结构

* HTTPS工作过程

* 必要命令openssl了解

> OpenSSL：开源项目
> 三个组件：
> openssl：多用途的命令行工具，包openssl
> libcrypto：加密算法库，包openssl-libs
> libssl：加密模块应用库，实现了ssl及tls，包nss
> \* openssl命令：
> 两种运行模式：交互模式和批处理模式
> openssl version：程序版本号
> 标准命令、消息摘要命令、加密命令
> 标准命令：enc, ca, req, …

## 1搭建CA服务器

### ①在服务器端生成私钥

```
[root@localhost ~]# cd /etc/pki/CA
[root@localhost CA]# touch index.txt #生成证书索引数据库文件
[root@localhost CA]# echo 0F > serial #指定第一个颁发证书的序列号
[root@localhost CA]# (umask 066;openssl genrsa -out private/cakey.pem 4096 )   #生成私钥
Generating RSA private key, 4096 bit long modulus
.......++
.........................................++
e is 65537 (0x10001)
[root@localhost CA]# openssl req -new -x509 -key private/cakey.pem  -out cacert.pem -days 3650 #给自己颁发证书
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:beijin
Locality Name (eg, city) [Default City]:beijin
Organization Name (eg, company) [Default Company Ltd]:ailibaba
Organizational Unit Name (eg, section) []:taobao
Common Name (eg, your name or your server's hostname) []:www.taobao.com
Email Address []
[root@localhost CA]# tree
.
├── cacert.pem
├── certs
├── crl
├── index.txt
├── newcerts
├── private
│   └── cakey.pem
└── serial
4 directories, 4 files
[root@localhost CA]# openssl x509 -in cacert.pem -noout -text  # 以易读方式打开证书
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            f6:4f:6a:1f:a6:de:88:9a
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=CN, ST=beijin, L=beijin, O=ailibaba, OU=taobao, CN=www.taobao.com
        Validity
            Not Before: Apr 18 07:51:51 2019 GMT
            Not After : Apr 15 07:51:51 2029 GMT
        Subject: C=CN, ST=beijin, L=beijin, O=ailibaba, OU=taobao, CN=www.taobao.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:ed:09:66:55:c8:65:18:a7:aa:7d:0b:fe:d3:91:
                    b3:f2:a2:a2:4a:ca:02:34:70:37:5d:80:8c:21:79:
                    e9:58:78:73:98:8c:c4:e5:43:ee:44:ca:60:72:50:
                    05:43:d4:cc:4a:bc:b7:4a:33:53:13:b0:df:b0:5d:
                    ac:9d:a3:af:70:37:ca:09:4e:ce:69:77:2a:1a:ee:
                    db:40:0c:d5:49:be:c0:a0:f6:a4:8d:33:20:57:54:
                    30:ce:74:fe:cd:30:3f:8d:9f:bc:f9:0e:db:1f:7c:
                    93:ab:ad:41:78:53:b5:f9:a2:8c:d4:48:80:82:e0:
                    aa:13:45:73:22:f0:41:16:a1:1f:59:bb:c1:7e:58:
                    16:3c:24:ac:1b:53:19:0b:81:87:f7:9b:b6:86:4e:
                    82:c4:7a:29:d1:39:54:d9:36:b0:7b:95:79:fc:13:
                    29:48:d2:cc:b0:ae:34:f0:22:8f:df:b3:76:8a:84:
                    3a:ce:36:97:85:3d:10:50:a7:12:24:17:1d:9d:bf:
                    f8:e9:7c:7b:b4:67:c9:1f:41:ee:19:45:9b:39:70:
                    d7:9e:7f:97:44:1e:f5:ee:cb:70:e6:6a:f7:8f:a6:
                    44:da:00:18:c3:de:4b:66:8f:d7:45:a7:09:43:f1:
                    be:0c:68:1a:18:ae:05:61:1f:2f:01:c7:8d:74:3f:
                    7f:b5:5b:65:dd:6e:d9:47:0f:38:b3:ff:7c:92:95:
                    48:de:d5:44:17:07:da:5e:bd:00:e8:03:bd:ee:47:
                    3f:7a:14:a6:63:1c:29:d8:16:ce:26:1a:2a:ee:bd:
                    57:43:d0:4d:08:52:96:e4:68:0a:b5:19:c9:ea:4d:
                    42:53:ec:3a:45:a6:ca:68:b9:e8:2e:38:f0:4c:51:
                    4b:e9:20:5c:f4:b4:7b:20:6a:dd:21:31:49:d6:b1:
                    39:0f:dc:22:52:2c:cb:94:21:af:e6:82:09:a8:08:
                    ef:f1:21:61:da:fb:ba:ce:8f:70:4d:e0:d9:b0:d1:
                    6e:42:37:33:f0:8d:57:14:56:6a:5e:2c:60:8e:3f:
                    05:06:35:53:e0:0b:81:9a:11:38:b1:95:c6:f6:1d:
                    f6:85:61:99:b6:bc:d0:2e:ab:d9:5e:6a:53:4e:95:
                    5e:a5:a5:4d:6a:45:3b:dd:d5:c4:1b:d1:95:f0:24:
                    a0:7c:19:42:8b:2e:cd:df:a7:2d:e3:d6:a4:f7:22:
                    a4:52:bd:2c:0f:77:fc:b3:27:89:55:31:0a:8f:2a:
                    3a:ec:07:45:29:96:09:f5:e6:95:87:e2:21:c8:a1:
                    be:6b:f8:95:9a:9c:08:52:48:19:c0:0c:a4:d8:37:
                    19:42:98:21:40:45:3c:6a:ff:e7:33:8d:1f:2f:ef:
                    73:c5:5f
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier: 
                D5:5D:21:99:D3:9A:BA:90:16:F4:BF:2D:78:C7:27:DF:F5:8B:42:F7
            X509v3 Authority Key Identifier: 
                keyid:D5:5D:21:99:D3:9A:BA:90:16:F4:BF:2D:78:C7:27:DF:F5:8B:42:F7

            X509v3 Basic Constraints: 
                CA:TRUE
    Signature Algorithm: sha256WithRSAEncryption
         27:a5:73:06:6c:2f:c4:a4:c0:24:29:3e:3f:5b:e8:e2:d7:fe:
         38:93:b5:c9:05:f5:45:9d:78:5b:ae:cd:bb:26:c0:fc:b6:e1:
         82:ef:7d:f3:28:48:c4:e2:c0:1a:ab:13:39:9f:95:98:c6:47:
         d1:dd:8f:b4:3e:dd:c5:79:38:94:01:9d:14:b9:f4:87:bd:88:
         a2:5d:4a:16:ee:f9:0d:9f:fa:d0:dc:c3:4b:a2:df:28:57:33:
         4e:31:c0:45:4f:d6:6e:ee:43:e5:9b:8f:7b:d8:46:66:83:fa:
         56:68:e6:30:19:0e:b4:41:74:dd:72:ce:e7:83:f5:50:f1:5d:
         46:29:fa:09:73:c5:e7:76:99:78:2b:35:9d:7c:69:91:47:cd:
         98:1d:28:b2:df:0b:a1:51:3b:f9:09:32:64:41:f1:00:d9:29:
         74:18:f9:98:bf:2c:b1:81:95:bb:3d:d0:57:46:cc:78:9a:51:
         38:7e:6b:cb:ff:7d:84:98:81:70:c2:49:79:f3:f0:5a:7a:47:
         db:4d:4d:6a:6a:14:97:02:fa:80:91:39:b2:8c:b8:85:ec:a6:
         10:b5:aa:82:a3:7f:5a:f4:75:09:11:47:91:64:f9:6c:f0:87:
         11:9a:d8:26:71:be:45:dc:9a:aa:57:2e:5b:78:45:5f:72:9f:
         ae:d8:d4:f1:e7:65:c7:fb:69:b9:d7:04:03:3d:26:00:74:09:
         4d:97:4d:83:1f:d9:ec:52:18:e0:45:ff:f6:2d:d7:2d:6a:76:
         e7:63:28:a5:24:97:73:46:d5:2b:39:aa:25:7c:78:fb:f7:13:
         65:f7:56:18:13:74:f0:f2:a2:b2:a0:61:09:0c:a3:56:aa:46:
         4f:34:3e:ca:85:30:ea:06:7b:a3:ed:ce:a1:83:d2:c6:63:26:
         e8:02:f5:a7:78:fd:84:dd:33:5d:b1:0c:af:fe:6b:30:0b:b2:
         fe:eb:95:3c:dd:7e:37:ac:4f:cf:19:64:45:4b:b8:05:14:91:
         97:68:39:39:08:d8:e2:4d:d0:eb:64:0b:a1:38:68:ac:c6:14:
         66:b1:d3:15:d2:5c:50:eb:99:69:bf:ce:87:38:07:00:af:14:
         4a:d1:0d:f8:e2:be:6f:46:5f:5a:ad:0c:e3:42:d0:49:37:59:
         47:93:17:b7:ee:6f:0a:8f:b1:13:ef:9d:dd:7f:c1:fc:f5:80:
         73:42:cf:aa:57:62:96:99:8e:eb:4c:6c:d3:fd:4a:82:52:e3:
         03:e0:07:c9:33:44:e3:6e:60:7e:5b:b6:fb:62:e1:55:5a:4b:
         fb:61:7e:87:e7:59:0b:4c:bd:72:f1:4d:91:02:b4:39:01:ae:
         45:0b:5b:e1:f7:1e:41:c3
```

## ②在客户端生成证书申请

```
root:/data# (umask 066;openssl genrsa -out test.key 1024)  # 生成私钥
Generating RSA private key, 1024 bit long modulus
................++++++
.......................++++++
e is 65537 (0x10001)
root:/data# openssl req -new  -key test.key -out test.csr # 生成csr证书申请文件
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:beijin
Locality Name (eg, city) [Default City]:changping
Organization Name (eg, company) [Default Company Ltd]:jindong
Organizational Unit Name (eg, section) []:wuliu
Common Name (eg, your name or your server's hostname) []:www.jd.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
root:/data# scp test.csr 172.22.50.53:/etc/pki/CA/certs/test.csr # 将证书传给客户端
```

- 注意：默认要求 国家，省，公司名称三项必须和CA一致

## ③在CA服务器端给客户端颁发证书

```
    [root@localhost CA]# openssl ca -in certs/test.csr  -out certs/test.crt -days 100
    Using configuration from /etc/pki/tls/openssl.cnf
    Check that the request matches the signature
    Signature ok
    Certificate Details:
    Serial Number: 15 (0xf)
    Validity
     Not Before: Apr 18 08:15:24 2019 GMT
        Not After : Jul 27 08:15:24 2019 GMT
    Subject:
        countryName               = CN
        stateOrProvinceName       = beijin
        localityName              = changping
        organizationName          = jindong
        organizationalUnitName    = wuliu
        commonName                = www.jd.com
    X509v3 extensions:
        X509v3 Basic Constraints: 
            CA:FALSE
        Netscape Comment: 
            OpenSSL Generated Certificate
        X509v3 Subject Key Identifier: 
            FB:94:F3:F3:2B:AB:12:4A:93:B0:83:8C:B3:CA:0E:0A:82:E8:EA:B9
        X509v3 Authority Key Identifier: 
            keyid:D5:5D:21:99:D3:9A:BA:90:16:F4:BF:2D:78:C7:27:DF:F5:8B:42:F7
                            Certificate is to be certified until Jul 27 08:15:24 2019 GMT (100 days)
                            Sign the certificate? [y/n]:y
                            1 out of 1 certificate requests certified, commit? [y/n]y
                            Write out database with 1 new entries
                            Data Base Updated
```

## 实现可多次颁发证书

```
cat index.txt.attr
unique_subject = yes
改为no
```

## 吊销证书

在客户端获取要吊销的证书的

```
serial openssl x509 -in /PATH/FROM/CERT_FILE -noout -serial -subject 
```

在CA上，根据客户提交的serial与subject信息，对比检验是否与index.txt文件中的信息一致， 吊销证书：

```
openssl ca -revoke /etc/pki/CA/newcerts/SERIAL.pem 
```

指定第一个吊销证书的编号,注意：第一次更新证书吊销列表前，才需要执行

```
echo 01 > /etc/pki/CA/crlnumber 
更新证书吊销列表 openssl ca -gencrl -out /etc/pki/CA/crl.pem 
查看crl文件： openssl crl -in /etc/pki/CA/crl.pem -noout -text
```

## 修改默认配置

```
policy          = policy_anything # 可使国家，城市等信息不一样
```

![img](CA和证书/6f6054985d77c3769bdd0f9bf5eded56.png)

## 基于key验证远程登录主机

进入用户秘钥管理
![img](CA和证书/baa3c7a51804d36466401de16de35127.png)
点击生成
![img](CA和证书/cc8d801577a6d65c3112b50e4995c631.png)
点击保存为文件
![img](CA和证书/eec64de7987a4543b45307dd944e0fb7.png)
在客户端保存公钥

```
[root@localhost CA]# cd 
[root@localhost ~]# cd .ssh
-bash: cd: .ssh: No such file or directory
[root@localhost ~]# mkdir .ssh
[root@localhost ~]# cd .ssh
[root@localhost .ssh]# 
[root@localhost .ssh]# tree
.
└── known_hosts

0 directories, 1 file
[root@localhost .ssh]# rz -E
rz waiting to receive.
[root@localhost .ssh]# >authorized_keys
[root@localhost .ssh]# ls
7key.pub  authorized_keys  known_hosts
[root@localhost .ssh]# cat 7key.pub >>authorized_keys 
[root@localhost .ssh]# tree
.
├── 7key.pub
├── authorized_keys
└── known_hosts

0 directories, 3 files
```

![img](CA和证书/cfa4cc9097c6771909f8639add4be724.png)
更改后即可基于key验证登录