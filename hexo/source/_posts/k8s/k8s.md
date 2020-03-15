---
title: k8s-15.1手动搭建
date: 2020-03-02 11:57:12
tags:
  - k8s
categories:
  - k8s
---

## 环境说明

### 部署节点说明

| 主机名 |       IP        |  用途  |                       部署软件                       |
| :----: | :-------------: | :----: | :--------------------------------------------------: |
| k8s-1  | 192.168.123.110 | master | apiserver,scheduler,controller-manager etcd,flanneld |
| k8s-2  | 192.168.123.120 |  node  |       kubelet,kube-proxy etcd,flanneld,docker        |
| k8s-3  | 192.168.123.130 |  node  |       kubelet,kube-proxy etcd,flanneld,docker        |

<!--more-->

### 操作系统环境

```
# 三台机器相同
[root@k8s-1 /root] master
# uname  -r
3.10.0-957.el7.x86_64

[root@k8s-1 /root] master
# cat  /etc/redhat-release
CentOS Linux release 7.6.1810 (Core)

[root@k8s-1 /root] master
# sestatus
SELinux status:                 disabled
```

### 软件包版本

|                软件包                |                           下载地址                           |
| :----------------------------------: | :----------------------------------------------------------: |
|  kubernetes-node-linux-amd64.tar.gz  | <https://dl.k8s.io/v1.15.1/kubernetes-node-linux-amd64.tar.gz> |
| kubernetes-server-linux-amd64.tar.gz | <https://dl.k8s.io/v1.15.1/kubernetes-server-linux-amd64.tar.gz> |
|  flannel-v0.11.0-linux-amd64.tar.gz  | <https://github.com/coreos/flannel/releases/download/v0.11.0/flannel-v0.11.0-linux-amd64.tar.gz> |
|   etcd-v3.3.10-linux-amd64.tar.gz    | <https://github.com/etcd-io/etcd/releases/download/v3.3.10/etcd-v3.3.10-linux-amd64.tar.gz> |



## 初始化环境

### 设置关闭防火墙及SELINUX

```shell
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
vi /etc/selinux/config
SELINUX=disabled
```



### 关闭swap

```shell
swapoff -a && sysctl -w vm.swappiness=0
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 设置Docker所需参数

```shell
cat > /etc/sysctl.d/kubernetes.conf <<EOF
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
EOF
sysctl -p  /etc/sysctl.d/kubernetes.conf 
```



### 安装 Docker(node节点)

```shell
# 配置yum源
cd  /etc/yum.repos.d/
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum clean all && yum  repolist -y

# 安装docker ，版本 18.06.2
yum  install  docker-ce-18.06.2.ce-3.el7 -y

# 启动
systemctl start docker && systemctl enable docker
```



### SSH 互信配置

```shell
添加hoosts解析
cat >> /etc/hosts << EOF
192.168.220.110 k8s-m1
192.168.220.120 k8s-n1
192.168.220.130 k8s-n2
EOF
ssh-keygen
ssh-copy-id  k8s-m1
ssh-copy-id  k8s-n1
ssh-copy-id  k8s-n2
```

### 工作目录

```
mkdir -p /usr/k8s/bin/
mkdir -p /etc/kubernetes/ssl
mkdir -p /etc/etcd/ssl
mkdir -p /etc/flanneld/ssl
mkdir -p /var/lib/etcd
```

### 集群环境变量

```shell
echo 'PATH=/usr/k8s/bin:$PATH' >> /etc/profile

cat >> /etc/profile.d/k8s.sh << EOF
# TLS Bootstrapping 使用的Token，可以使用命令 head -c 16 /dev/urandom | od -An -t x | tr -d ' ' 生成
BOOTSTRAP_TOKEN="f3be89d17cf7d80847f3bc8b8ea58250"

# 建议使用未用的网段来定义服务网段和Pod 网段
# 服务网段(Service CIDR)，部署前路由不可达，部署后集群内部使用IP:Port可达
SERVICE_CIDR="10.254.0.0/16"

# Pod 网段(Cluster CIDR)，部署前路由不可达，部署后路由可达(flanneld 保证)
CLUSTER_CIDR="172.30.0.0/16"

# 服务端口范围(NodePort Range)
NODE_PORT_RANGE="30000-32766"

# etcd集群服务地址列表
ETCD_ENDPOINTS="https://192.168.220.110:2379,https://192.168.220.120:2379,https://192.168.220.130:2379"

# flanneld 网络配置前缀
FLANNEL_ETCD_PREFIX="/kubernetes/network"

# kubernetes 服务IP(预先分配，一般为SERVICE_CIDR中的第一个IP)
CLUSTER_KUBERNETES_SVC_IP="10.254.0.1"

# 集群 DNS 服务IP(从SERVICE_CIDR 中预先分配)
CLUSTER_DNS_SVC_IP="10.254.0.2"

# 集群 DNS 域名
CLUSTER_DNS_DOMAIN="cluster.local."

# MASTER API Server 地址
MASTER_URL="k8s-api.virtual.local"

#变量KUBE_APISERVER 指定kubelet 访问的kube-apiserver 的地址，后续被写入~/.kube/config配置文件
KUBE_APISERVER="https://${MASTER_URL}:6443"
EOF
source /etc/profile
source /etc/profile.d/k8s.sh
```

## 创建CA 证书和密钥

`kubernetes` 系统各个组件需要使用`TLS`证书对通信进行加密，这里我们使用`CloudFlare`的PKI 工具集[cfssl](https://github.com/cloudflare/cfssl) 来生成Certificate Authority(CA) 证书和密钥文件， CA 是自签名的证书，用来签名后续创建的其他TLS 证书。

### 安装 CFSSL(master)

```shell
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
chmod +x cfssl_linux-amd64
sudo mv cfssl_linux-amd64 /usr/k8s/bin/cfssl

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssljson_linux-amd64
sudo mv cfssljson_linux-amd64 /usr/k8s/bin/cfssljson

wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl-certinfo_linux-amd64
sudo mv cfssl-certinfo_linux-amd64 /usr/k8s/bin/cfssl-certinfo
chmod +x /usr/k8s/bin/*
```

```shll
mkdir ssl && cd ssl
cfssl print-defaults config > ca-config.json
cfssl print-defaults csr > ca-csr.json
```

### 创建CA

修改上面创建的`ca-config.json`：

```json
cat > ca-config.json << EOF
{
    "signing": {
        "default": {
            "expiry": "876000h"
        },
        "profiles": {
            "kubernetes": {
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF
```

- `config.json`：可以定义多个profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个profile；
- `signing`: 表示该证书可用于签名其它证书；生成的ca.pem 证书中`CA=TRUE`；
- `server auth`: 表示client 可以用该CA 对server 提供的证书进行校验；
- `client auth`: 表示server 可以用该CA 对client 提供的证书进行验证。

修改CA 证书签名ca-csr.json：

```json
cat > ca-csr.json << EOF
{
    "CN": "kubernetes",
    
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

- `CN`: `Common Name`，kube-apiserver 从证书中提取该字段作为请求的用户名(User Name)；浏览器使用该字段验证网站是否合法；
- `O`: `Organization`，kube-apiserver 从证书中提取该字段作为请求用户所属的组(Group)；

生成CA 证书和私钥：

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
ls ca*
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```

### 分发证书

将生成的CA 证书、密钥文件、配置文件拷贝到**所有机器**的`/etc/kubernetes/ssl`目录下面：

```bash
sudo cp ca* /etc/kubernetes/ssl
```

## 部署高可用etcd 集群

kubernetes 系统使用`etcd`存储所有的数据，我们这里部署3个节点的etcd 集群，这3个节点直接复用kubernetes master的3个节点，分别命名为`etcd01`、`etcd02`、`etcd03`:

- etcd01：192.168.220.110
- etcd02：192.168.220.120
- etcd03：192.168.220.130

### 定义环境变量

使用到的变量如下：

节点1

```bash
cat >> /usr/k8s/bin/env.sh <<EOF

# 当前部署的机器名称(随便定义，只要能区分不同机器即可)
NODE_NAME=etcd01

#当前部署的机器IP
NODE_IP=192.168.220.110

# etcd 集群所有机器 IP
NODE_IPS="192.168.220.110 192.168.220.120 192.168.220.130" 

# etcd 集群间通信的IP和端口
ETCD_NODES=etcd01=https://192.168.220.110:2380,etcd02=https://192.168.220.120:2380,etcd03=https://192.168.220.130:2380
# 导入用到的其它全局变量：ETCD_ENDPOINTS、FLANNEL_ETCD_PREFIX、CLUSTER_CIDR
EOF
source /usr/k8s/bin/env.sh
```

节点2

```bash
cat >> /usr/k8s/bin/env.sh <<EOF

# 当前部署的机器名称(随便定义，只要能区分不同机器即可)
NODE_NAME=etcd02

#当前部署的机器IP
NODE_IP=192.168.220.120

# etcd 集群所有机器 IP
NODE_IPS="192.168.220.110 192.168.220.120 192.168.220.130" 

# etcd 集群间通信的IP和端口
ETCD_NODES=etcd01=https://192.168.220.110:2380,etcd02=https://192.168.220.120:2380,etcd03=https://192.168.220.130:2380
# 导入用到的其它全局变量：ETCD_ENDPOINTS、FLANNEL_ETCD_PREFIX、CLUSTER_CIDR
EOF
source /usr/k8s/bin/env.sh
```

节点3

```bash
cat >> /usr/k8s/bin/env.sh <<EOF

# 当前部署的机器名称(随便定义，只要能区分不同机器即可)
NODE_NAME=etcd03

#当前部署的机器IP
NODE_IP=192.168.220.130

# etcd 集群所有机器 IP
NODE_IPS="192.168.220.110 192.168.220.120 192.168.220.130" 

# etcd 集群间通信的IP和端口
ETCD_NODES=etcd01=https://192.168.220.110:2380,etcd02=https://192.168.220.120:2380,etcd03=https://192.168.220.130:2380
# 导入用到的其它全局变量：ETCD_ENDPOINTS、FLANNEL_ETCD_PREFIX、CLUSTER_CIDR
EOF
source /usr/k8s/bin/env.sh
```

### 下载etcd 二进制文件

到<https://github.com/coreos/etcd/releases>页面下载最新版本的二进制文件：

```
wget https://github.com/etcd-io/etcd/releases/download/v3.3.17/etcd-v3.3.17-linux-amd64.tar.gz
tar -xvf etcd-v3.3.17-linux-amd64.tar.gz
sudo mv etcd-v3.3.17-linux-amd64/etcd* /usr/k8s/bin/
chmod +x /usr/k8s/bin/*
```

### 创建TLS 密钥和证书

为了保证通信安全，客户端(如etcdctl)与etcd 集群、etcd 集群之间的通信需要使用TLS 加密。

创建etcd 证书签名请求：

```
root@k8s-m1 ~/ssl
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "${NODE_IP}"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

- `hosts` 字段指定授权使用该证书的`etcd`节点IP

生成`etcd`证书和私钥：

```
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
  
$ ls etcd*
etcd.csr  etcd-csr.json  etcd-key.pem  etcd.pem

sudo mv etcd*.pem /etc/etcd/ssl/
```

### 创建etcd 的systemd unit 文件

```
cat > etcd.service <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/k8s/bin/etcd \\
  --name=${NODE_NAME} \\
  --cert-file=/etc/etcd/ssl/etcd.pem \\
  --key-file=/etc/etcd/ssl/etcd-key.pem \\
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \\
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \\
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --initial-advertise-peer-urls=https://${NODE_IP}:2380 \\
  --listen-peer-urls=https://${NODE_IP}:2380 \\
  --listen-client-urls=https://${NODE_IP}:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://${NODE_IP}:2379 \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=${ETCD_NODES} \\
  --initial-cluster-state=new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

- 指定`etcd`的工作目录和数据目录为`/var/lib/etcd`，需要在启动服务前创建这个目录；
- 为了保证通信安全，需要指定etcd 的公私钥(cert-file和key-file)、Peers通信的公私钥和CA 证书(peer-cert-file、peer-key-file、peer-trusted-ca-file)、客户端的CA 证书(trusted-ca-file)；
- `--initial-cluster-state`值为`new`时，`--name`的参数值必须位于`--initial-cluster`列表中；

### 启动其他节点etcd 服务

#### 分发ca证书

```
scp -r /etc/kubernetes/ssl/ca* k8s-n1:/etc/kubernetes/ssl/
scp -r /etc/kubernetes/ssl/ca* k8s-n2:/etc/kubernetes/ssl/
```

#### 创建node1，node2 etcd 的systemd unit 文件

#### 生成`etcd`证书和私钥： (node1,node2)

```
先在node上创建etcd 证书签名请求：
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
  
$ ls etcd*
etcd.csr  etcd-csr.json  etcd-key.pem  etcd.pem

sudo mv etcd*.pem /etc/etcd/ssl/
```

### 启动etcd 服务

```
sudo mv etcd.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
sudo systemctl status etcd
```

最先启动的etcd 进程会卡住一段时间，等待其他节点启动加入集群，在所有的etcd 节点重复上面的步骤，直到所有的机器etcd 服务都已经启动。



### 验证服务

部署完etcd 集群后，在任一etcd 节点上执行下面命令：

```
for ip in ${NODE_IPS}; do
  ETCDCTL_API=3 /usr/k8s/bin/etcdctl \
  --endpoints=https://${ip}:2379  \
  --cacert=/etc/kubernetes/ssl/ca.pem \
  --cert=/etc/etcd/ssl/etcd.pem \
  --key=/etc/etcd/ssl/etcd-key.pem \
  endpoint health; done
```

输出如下结果：

```
https://192.168.220.110:2379 is healthy: successfully committed proposal: took = 5.709368ms
https://192.168.220.120:2379 is healthy: successfully committed proposal: took = 6.465407ms
https://192.168.220.130:2379 is healthy: successfully committed proposal: took = 5.402329ms
```

可以看到上面的信息3个节点上的etcd 均为**healthy**，则表示集群服务正常。

## 部署Flannel网络

kubernetes 要求集群内各节点能通过Pod 网段互联互通，下面我们来使用Flannel 在所有节点上创建互联互通的Pod 网段的步骤。

> 需要在所有的Node节点安装。 #所有节点（包括master）

### 创建TLS 密钥和证书

etcd 集群启用了双向TLS 认证，所以需要为flanneld 指定与etcd 集群通信的CA 和密钥。

创建flanneld 证书签名请求：

```
cat > flanneld-csr.json <<EOF
{
  "CN": "flanneld",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

生成flanneld 证书和私钥：

```
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
$ ls flanneld*
flanneld.csr  flanneld-csr.json  flanneld-key.pem flanneld.pem
sudo mv flanneld*.pem /etc/flanneld/ssl
scp -r /etc/flanneld/ssl/flanneld* k8s-n1:/etc/flanneld/ssl/
scp -r /etc/flanneld/ssl/flanneld* k8s-n2:/etc/flanneld/ssl/
```

### 向etcd 写入集群Pod 网段信息

> 该步骤只需在第一次部署Flannel 网络时执行，后续在其他节点上部署Flanneld 时无需再写入该信息

```
etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/flanneld/ssl/flanneld.pem \
  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
  set ${FLANNEL_ETCD_PREFIX}/config '{"Network":"'${CLUSTER_CIDR}'", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}'
# 得到如下反馈信息
{"Network":"172.30.0.0/16", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}
```

- 写入的 Pod 网段(${CLUSTER_CIDR}，172.30.0.0/16) 必须与`kube-controller-manager` 的 `--cluster-cidr` 选项值一致；

### 安装和配置flanneld

前往[flanneld release](https://github.com/coreos/flannel/releases)页面下载最新版的flanneld 二进制文件：

```
mkdir flannel
wget https://github.com/coreos/flannel/releases/download/v0.11.0/flannel-v0.11.0-linux-amd64.tar.gz
tar -xzvf flannel-v0.11.0-linux-amd64.tar.gz -C flannel
sudo cp flannel/{flanneld,mk-docker-opts.sh} /usr/k8s/bin
```

创建flanneld的systemd unit 文件

```
cat > flanneld.service << EOF
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
ExecStart=/usr/k8s/bin/flanneld \\
  -etcd-cafile=/etc/kubernetes/ssl/ca.pem \\
  -etcd-certfile=/etc/flanneld/ssl/flanneld.pem \\
  -etcd-keyfile=/etc/flanneld/ssl/flanneld-key.pem \\
  -etcd-endpoints=${ETCD_ENDPOINTS} \\
  -etcd-prefix=${FLANNEL_ETCD_PREFIX}
ExecStartPost=/usr/k8s/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
EOF
```

- `mk-docker-opts.sh`脚本将分配给flanneld 的Pod 子网网段信息写入到`/run/flannel/docker` 文件中，后续docker 启动时使用这个文件中的参数值为 docker0 网桥
- flanneld 使用系统缺省路由所在的接口和其他节点通信，对于有多个网络接口的机器(内网和公网)，可以用 `--iface` 选项值指定通信接口(上面的 systemd unit 文件没指定这个选项)

### 启动flanneld

```
sudo cp flanneld.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable flanneld
sudo systemctl restart flanneld
systemctl status flanneld
```

### 检查flanneld 服务

```
ifconfig flannel.1
```

### 检查分配给各flanneld 的Pod 网段信息

```
# 查看集群 Pod 网段(/16)
etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/flanneld/ssl/flanneld.pem \
  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
  get ${FLANNEL_ETCD_PREFIX}/config
{"Network":"172.30.0.0/16", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}

# 查看已分配的 Pod 子网段列表(/24)
etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/flanneld/ssl/flanneld.pem \
  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
  ls ${FLANNEL_ETCD_PREFIX}/subnets
/kubernetes/network/subnets/172.30.1.0-24
/kubernetes/network/subnets/172.30.70.0-24
/kubernetes/network/subnets/172.30.29.0-24
/kubernetes/network/subnets/172.30.61.0-24
/kubernetes/network/subnets/172.30.47.0-24

查看某一 Pod 网段对应的 flanneld 进程监听的 IP 和网络参数
etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/flanneld/ssl/flanneld.pem \
  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
  get ${FLANNEL_ETCD_PREFIX}/subnets/172.30.1.0-24
{"PublicIP":"192.168.220.102","BackendType":"vxlan","BackendData":{"VtepMAC":"66:51:aa:cc:3f:e7"}}
```

### 确保各节点间Pod 网段能互联互通

在各个节点部署完Flanneld 后，查看已分配的Pod 子网段列表：

```
$ etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/flanneld/ssl/flanneld.pem \
  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
  ls ${FLANNEL_ETCD_PREFIX}/subnets
  
/kubernetes/network/subnets/172.30.47.0-24
/kubernetes/network/subnets/172.30.1.0-24
/kubernetes/network/subnets/172.30.70.0-24
/kubernetes/network/subnets/172.30.29.0-24
/kubernetes/network/subnets/172.30.61.0-24
```

检查是否正确写入iptables

![](/k8s/1.png)

## 配置kubectl 命令行工具(master)

`kubectl`默认从`~/.kube/config`配置文件中获取访问kube-apiserver 地址、证书、用户名等信息，需要正确配置该文件才能正常使用`kubectl`命令。

需要将下载的kubectl 二进制文件和生产的`~/.kube/config`配置文件拷贝到需要使用kubectl 命令的机器上。

> 很多童鞋说这个地方不知道在哪个节点上执行，`kubectl`只是一个和`kube-apiserver`进行交互的一个命令行工具，所以你想安装到那个节点都行，master或者node任意节点都可以，比如你先在master节点上安装，这样你就可以在master节点使用`kubectl`命令行工具了，如果你想在node节点上使用(当然安装的过程肯定会用到的)，你就把master上面的`kubectl`二进制文件和`~/.kube/config`文件拷贝到对应的node节点上就行了。

### 环境变量

```
[16:10:31 root@k8s-m1 ~]$echo $MASTER_URL
k8s-api.virtual.local
# 变量KUBE_APISERVER 指定kubelet 访问的kube-apiserver 的地址，后续被写入~/.kube/config配置文件
[17:17:49 root@k8s-m1 ~]$echo $KUBE_APISERVER
https://k8s-api.virtual.local:6443
```



> 注意这里的`KUBE_APISERVER`地址，因为我们还没有安装`haproxy`，所以暂时需要手动指定使用`apiserver`的6443端口，等`haproxy`安装完成后就可以用使用443端口转发到6443端口去了。

- 变量KUBE_APISERVER 指定kubelet 访问的kube-apiserver 的地址，后续被写入`~/.kube/config`配置文件

### 下载kubectl

```shell
wget https://dl.k8s.io/v1.8.2/kubernetes-client-linux-amd64.tar.gz # 如果服务器上下载不下来，可以想办法下载到本地，然后scp上去即可
tar -xzvf kubernetes-client-linux-amd64.tar.gz
cp kubernetes/client/bin/kube* /usr/k8s/bin/
sudo chmod a+x /usr/k8s/bin/kube*
```

### 创建admin 证书

kubectl 与kube-apiserver 的安全端口通信，需要为安全通信提供TLS 证书和密钥。创建admin 证书签名请求：

```shell
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF
```

- 后续`kube-apiserver`使用RBAC 对客户端(如kubelet、kube-proxy、Pod)请求进行授权
- `kube-apiserver` 预定义了一些RBAC 使用的RoleBindings，如cluster-admin 将Group `system:masters`与Role `cluster-admin`绑定，该Role 授予了调用`kube-apiserver`所有API 的权限
- O 指定了该证书的Group 为`system:masters`，kubectl使用该证书访问`kube-apiserver`时，由于证书被CA 签名，所以认证通过，同时由于证书用户组为经过预授权的`system:masters`，所以被授予访问所有API 的劝降
- hosts 属性值为空列表

生成admin 证书和私钥： #分发证书到各个服务器

```shell
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes admin-csr.json | cfssljson -bare admin
ls admin*
admin.csr  admin-csr.json  admin-key.pem  admin.pem
sudo mv admin*.pem /etc/kubernetes/ssl/
```

### 创建kubectl kubeconfig 文件

```shell
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER}
# 设置客户端认证参数
kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --embed-certs=true \
  --client-key=/etc/kubernetes/ssl/admin-key.pem \
  --token=${BOOTSTRAP_TOKEN}
# 设置上下文参数
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin
# 设置默认上下文
kubectl config use-context kubernetes
```

- `admin.pem`证书O 字段值为`system:masters`，`kube-apiserver` 预定义的 RoleBinding `cluster-admin` 将 Group `system:masters` 与 Role `cluster-admin` 绑定，该 Role 授予了调用`kube-apiserver` 相关 API 的权限
- 生成的kubeconfig 被保存到 `~/.kube/config` 文件

### 分发kubeconfig 文件

将`~/.kube/config`文件拷贝到运行`kubectl`命令的机器的`~/.kube/`目录下去。

```
mkdir ~/.kube
```

## 部署master 节点

kubernetes master 节点包含的组件有：

- kube-apiserver
- kube-scheduler
- kube-controller-manager

目前这3个组件需要部署到同一台机器上：（后面再部署高可用的master）

- `kube-scheduler`、`kube-controller-manager` 和 `kube-apiserver` 三者的功能紧密相关；
- 同时只能有一个 `kube-scheduler`、`kube-controller-manager` 进程处于工作状态，如果运行多个，则需要通过选举产生一个 leader；

master 节点与node 节点上的Pods 通过Pod 网络通信，所以需要在master 节点上部署Flannel 网络。

### 环境变量

```shell
[16:14:18 root@k8s-m1 ~]$echo $NODE_IP
192.168.220.110
```

### 下载最新版本的二进制文件

在[kubernetes changelog](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.8.md#server-binaries) 页面下载最新版本的文件：

```shell
wget https://dl.k8s.io/v1.8.9/kubernetes-server-linux-amd64.tar.gz
tar -xzvf kubernetes-server-linux-amd64.tar.gz
```

将二进制文件拷贝到`/usr/k8s/bin`目录

```shell
sudo cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler} /usr/k8s/bin/
```

### 创建kubernetes 证书 #以下master机器都要做

创建kubernetes 证书签名请求：

```shell
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "${NODE_IP}",
    "${MASTER_URL}",
    "${CLUSTER_KUBERNETES_SVC_IP}",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

- 如果 hosts 字段不为空则需要指定授权使用该证书的 **IP 或域名列表**，所以上面分别指定了当前部署的 master 节点主机 IP 以及apiserver 负载的内部域名
- 还需要添加 kube-apiserver 注册的名为 `kubernetes` 的服务 IP (Service Cluster IP)，一般是 kube-apiserver `--service-cluster-ip-range` 选项值指定的网段的**第一个IP**，如 “10.254.0.1”

生成kubernetes 证书和私钥：

```shell
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
  
$ ls kubernetes*
kubernetes.csr  kubernetes-csr.json  kubernetes-key.pem  kubernetes.pem

sudo mv kubernetes*.pem /etc/kubernetes/ssl/
```

### 6.1 配置和启动kube-apiserver

#### 创建kube-apiserver 使用的客户端token 文件

kubelet 首次启动时向kube-apiserver 发送TLS Bootstrapping 请求，kube-apiserver 验证请求中的token 是否与它配置的token.csv 一致，如果一致则自动为kubelet 生成证书和密钥。

```shell
# 导入的 environment.sh 文件定义了 BOOTSTRAP_TOKEN 变量
cat > token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
sudo mv token.csv /etc/kubernetes/
```

#### 创建kube-apiserver 的systemd unit文件

```shell
cat > /etc/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/k8s/bin/kube-apiserver \\
--logtostderr=true \\
--v=4 \\
--etcd-servers=${ETCD_ENDPOINTS} \\
--bind-address=${NODE_IP} \\
--secure-port=6443 \\
--advertise-address=${NODE_IP} \\
--allow-privileged=true \\
--service-cluster-ip-range=${SERVICE_CIDR} \\
--insecure-bind-address=${NODE_IP} \\
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota,NodeRestriction \\
--authorization-mode=RBAC,Node \\
--enable-bootstrap-token-auth \\
--token-auth-file=/etc/kubernetes/token.csv \\
--service-node-port-range=30000-50000 \\
--tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem  \\
--tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \\
--client-ca-file=/etc/kubernetes/ssl/ca.pem \\
--service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \\
--etcd-cafile=/etc/kubernetes/ssl/ca.pem \\
--etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \\
--etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

- kube-apiserver 1.6 版本开始使用 etcd v3 API 和存储格式
- `--authorization-mode=RBAC` 指定在安全端口使用RBAC 授权模式，拒绝未通过授权的请求
- kube-scheduler、kube-controller-manager 一般和 kube-apiserver 部署在同一台机器上，它们使用**非安全端口**和 kube-apiserver通信
- kubelet、kube-proxy、kubectl 部署在其它 Node 节点上，如果通过**安全端口**访问 kube-apiserver，则必须先通过 TLS 证书认证，再通过 RBAC 授权
- kube-proxy、kubectl 通过使用证书里指定相关的 User、Group 来达到通过 RBAC 授权的目的
- 如果使用了 kubelet TLS Boostrap 机制，则不能再指定 `--kubelet-certificate-authority`、`--kubelet-client-certificate` 和 `--kubelet-client-key` 选项，否则后续 kube-apiserver 校验 kubelet 证书时出现 ”x509: certificate signed by unknown authority“ 错误
- `--admission-control` 值必须包含 `ServiceAccount`，否则部署集群插件时会失败
- `--bind-address` 不能为 `127.0.0.1`
- `--service-cluster-ip-range` 指定 Service Cluster IP 地址段，该地址段不能路由可达
- `--service-node-port-range=${NODE_PORT_RANGE}` 指定 NodePort 的端口范围
- 缺省情况下 kubernetes 对象保存在`etcd/registry` 路径下，可以通过 `--etcd-prefix` 参数进行调整
- kube-apiserver 1.8版本后需要在`--authorization-mode`参数中添加`Node`，即：`--authorization-mode=Node,RBAC`，否则Node 节点无法注册
- licy-file`，我们也可以使用日志收集工具收集相关的日志进行分析。

#### 启动kube-apiserver

```shell
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver
sudo systemctl start kube-apiserver
sudo systemctl status kube-apiserver
```

### 6.2 配置和启动kube-controller-manager

#### 创建kube-controller-manager 的systemd unit 文件

```shell
cat > /etc/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/k8s/bin/kube-controller-manager \\
--logtostderr=true \\
--v=4 \\
--master=${MASTER_URL}:8080 \\
--leader-elect=true \\
--address=127.0.0.1 \\
--service-cluster-ip-range=${SERVICE_CIDR} \\
--cluster-name=kubernetes \\
--cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  \\
--root-ca-file=//etc/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

- `--address` 值必须为 `127.0.0.1`，因为当前 kube-apiserver 期望 scheduler 和 controller-manager 在同一台机器
- `--master=http://${MASTER_URL}:8080`：使用`http`(非安全端口)与 kube-apiserver 通信，需要下面的`haproxy`安装成功后才能去掉8080端口。
- `--cluster-cidr` 指定 Cluster 中 Pod 的 CIDR 范围，该网段在各 Node 间必须路由可达(flanneld保证)
- `--service-cluster-ip-range` 参数指定 Cluster 中 Service 的CIDR范围，该网络在各 Node 间必须路由不可达，必须和 kube-apiserver 中的参数一致
- `--cluster-signing-*` 指定的证书和私钥文件用来签名为 TLS BootStrap 创建的证书和私钥
- `--root-ca-file` 用来对 kube-apiserver 证书进行校验，**指定该参数后，才会在Pod 容器的 ServiceAccount 中放置该 CA 证书文件**
- `--leader-elect=true` 部署多台机器组成的 master 集群时选举产生一处于工作状态的 `kube-controller-manager` 进程

#### 启动kube-controller-manager

```shell
sudo systemctl daemon-reload
sudo systemctl enable kube-controller-manager
sudo systemctl start kube-controller-manager
sudo systemctl status kube-controller-manager
```

### 6.3 配置和启动kube-scheduler

#### 创建kube-scheduler 的systemd unit文件

```shell
cat > /etc/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/k8s/bin/kube-scheduler \\
--logtostderr=true \\
--v=4 \\
--address=127.0.0.1 \\
--master=${MASTER_URL}:8080 \\
--leader-elect
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

- `--address` 值必须为 `127.0.0.1`，因为当前 kube-apiserver 期望 scheduler 和 controller-manager 在同一台机器
- `--master=http://${MASTER_URL}:8080`：使用`http`(非安全端口)与 kube-apiserver 通信，需要下面的`haproxy`启动成功后才能去掉8080端口
- `--leader-elect=true` 部署多台机器组成的 master 集群时选举产生一处于工作状态的 `kube-controller-manager` 进程

#### 启动kube-scheduler

k8s-api.virtual.local 加hosts解析

```shell
sudo cp kube-scheduler.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable kube-scheduler
sudo systemctl start kube-scheduler
sudo systemctl status kube-scheduler
```

### 验证master 节点

```shell
kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-1               Healthy   {"health": "true"}
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
```

## 部署Node 节点

kubernetes Node 节点包含如下组件：

- flanneld
- docker
- kubelet
- kube-proxy

### 部署 kubelet 组件

> ```
> kublet 运行在每个 worker 节点上，接收 kube-apiserver 发送的请求，管理 Pod 容器，执行交互式命令，如exec、run、logs 等;
> ```
>
> kublet 启动时自动向 kube-apiserver 注册节点信息，内置的 cadvisor 统计和监控节点的资源使用情况;
> 为确保安全，本文档只开启接收 https 请求的安全端口，对请求进行认证和授权，拒绝未授权的访问(如apiserver、heapster)。

**创建 kubelet bootstrap.kubeconfig 文件**

```
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig

# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig

# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```

**创建 kubelet.kubeconfig 文件**

```
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kubelet.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials kubelet \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=kubelet.kubeconfig

# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet \
  --kubeconfig=kubelet.kubeconfig

# 设置默认上下文
kubectl config use-context default --kubeconfig=kubelet.kubeconfig
```

#### 创建kube-proxy 证书签名请求：

```shell
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

- CN 指定该证书的 User 为 `system:kube-proxy`
- `kube-apiserver` 预定义的 RoleBinding `system:node-proxier` 将User `system:kube-proxy` 与 Role `system:node-proxier`绑定，该 Role 授予了调用 `kube-apiserver` Proxy 相关 API 的权限
- hosts 属性值为空列表

#### 生成kube-proxy 客户端证书和私钥

```shell
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
ls kube-proxy*
kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem
sudo mv kube-proxy*.pem /etc/kubernetes/ssl/
```

**创建kube-proxy kubeconfig文件**

```
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
# 设置客户端认证参数
kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
# 设置默认上下文
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

- 设置集群参数和客户端认证参数时 `--embed-certs` 都为 `true`，这会将 `certificate-authority`、`client-certificate` 和 `client-key` 指向的证书文件内容写入到生成的 `kube-proxy.kubeconfig` 文件中
- `kube-proxy.pem` 证书中 CN 为 `system:kube-proxy`，`kube-apiserver` 预定义的 RoleBinding `cluster-admin` 将User `system:kube-proxy` 与 Role `system:node-proxier` 绑定，该 Role 授予了调用 `kube-apiserver` Proxy 相关 API 的权限

将bootstrap.kubeconfig kube-proxy.kubeconfig 文件拷贝到所有 nodes节点

```
scp -rp bootstrap.kubeconfig kube-proxy.kubeconfig kubelet.kubeconfig k8s-n1:/etc/kubernetes/
scp -rp bootstrap.kubeconfig kube-proxy.kubeconfig kubelet.kubeconfig k8s-n2:/etc/kubernetes/
```

#### 创建kube-proxy systemd unit 文件

```
cat > /etc/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/k8s/bin/kube-proxy \\
--logtostderr=true \\
--v=4 \\
--hostname-override=${NODE_IP} \\
--cluster-cidr=${SERVICE_CIDR} \\
--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```



#### 启动kube-proxy服务

```
systemctl daemon-reload
systemctl enable kube-proxy
systemctl restart kube-proxy
```



#### 检查服务运行状态

```
systemctl status kube-proxy
```

#### 创建kubelet 参数配置文件拷贝到所有 nodes节点

**创建 kubelet 参数配置模板文件**

```
[root@k8s-2 /root] node
# cat /cloud/k8s/kubernetes/cfg/kubelet.config << EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 192.168.220.120
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDomain: cluster.local.
failSwapOn: false
authentication:
  anonymous:
    enabled: true
```

**创建kubelet systemd unit 文件**

```
cat > /etc/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/k8s/bin/kubelet \\
--logtostderr=true \\
--v=4 \\
--address=${NODE_IP} \\
--hostname-override=${NODE_IP} \\
--cluster-dns=${CLUSTER_DNS_SVC_IP} \\
--cluster-domain=${CLUSTER_DNS_DOMAIN} \\
--kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \\
--config=/etc/kubernetes/kubelet.config \\
--cert-dir=/etc/kubernetes/ssl \\
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.1
Restart=on-failure
KillMode=process

[Install]
WantedBy=multi-user.target
EOF
```

**将kubelet-bootstrap用户绑定到系统集群角色**

```
kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```



#### 启动kubelet服务

```
systemctl daemon-reload
systemctl enable kubelet
systemctl restart kubelet
```

#### approve kubelet CSR 请求

```
kubectl get csr
kubectl certificate approve $NAME 
kubectl get csr
csr 状态变为 Approved,Issued 即可
```

> - Requesting User：请求 CSR 的用户，kube-apiserver 对它进行认证和授权；
> - Subject：请求签名的证书信息；
> - 证书的 CN 是 system:node:kube-node2， Organization 是 system:nodes，kube-apiserver 的 Node 授权模式会授予该证书的相关权限；



#### 查看集群状态

```
[root@k8s-1 /root] master
# kubectl  get  nodes,cs
NAME         STATUS   ROLES   AGE     VERSION
node/k8s-2   Ready    node    3d22h   v1.15.1
node/k8s-3   Ready    node    3d22h   v1.15.1

NAME                                 STATUS    MESSAGE             ERROR
componentstatus/scheduler            Healthy   ok
componentstatus/controller-manager   Healthy   ok
componentstatus/etcd-0               Healthy   {"health":"true"}
componentstatus/etcd-2               Healthy   {"health":"true"}
componentstatus/etcd-1               Healthy   {"health":"true"}
```

### 验证集群功能

定义yaml 文件：（将下面内容保存为：nginx-ds.yaml）

```yaml
cat > nginx-ds.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-ds
  labels:
    app: nginx-ds
spec:
  type: NodePort
  selector:
    app: nginx-ds
  ports:
  - name: http
    port: 80
    targetPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    app: static
spec:
  containers:
  - name: web
    image: nginx
    ports:
    - name: web
      containerPort: 80
EOF
```

创建 Pod 和服务：

```shell
kubectl create -f nginx-ds.yaml
service "nginx-ds" created
daemonset "nginx-ds" created
```

### 创建失败：

describe 出现Failed create pod sandbox.

journalctl -u kubelet -n 100

![](D:/Github/hexo/source/_posts/%E6%89%8B%E5%8A%A8%E6%90%AD%E5%BB%BA%E9%AB%98%E5%8F%AF%E7%94%A8%E7%9A%84k8s%E9%9B%86%E7%BE%A4/2.png)

解决方法如下，从docker.io把pause-amd64镜像取下来，然后做个标签。这样就可以解决问题。

```
docker pull googlecontainer/pause-amd64:3.1
docker tag googlecontainer/pause-amd64:3.1 gcr.io/google_containers/pause-amd64:3.1
```

- 1.15.1拉取的版本是3.1

执行下面的命令查看Pod 和SVC：

```shell
kubectl get pods -o wide
NAME             READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-ds-f29zt   1/1       Running   0          23m       172.17.0.2   192.168.1.170
kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-ds     NodePort    10.254.6.249   <none>        80:30813/TCP   24m
```

可以看到：

- 服务IP：10.254.6.249
- 服务端口：80
- NodePort端口：30813

在所有 Node 上执行：

```
$ curl 10.254.6.249
$ curl 192.168.1.170:30813
```

执行上面的命令预期都会输出nginx 欢迎页面内容，表示我们的Node 节点正常运行了。

## 9. 部署kubedns 插件

官方文件目录：[kubernetes/cluster/addons/dns](https://github.com/kubernetes/kubernetes/tree/v1.8.2/cluster/addons/dns)

使用的文件：

```shell
$ ls *.yaml *.base
kubedns-cm.yaml  kubedns-sa.yaml  kubedns-controller.yaml.base  kubedns-svc.yaml.base
```

### 系统预定义的RoleBinding

预定义的RoleBinding `system:kube-dns`将kube-system 命名空间的`kube-dns`ServiceAccount 与 `system:kube-dns` Role 绑定，该Role 具有访问kube-apiserver DNS 相关的API 权限：

```shell
$ kubectl get clusterrolebindings system:kube-dns -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: 2017-11-06T10:51:59Z
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-dns
  resourceVersion: "78"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/system%3Akube-dns
  uid: 83a25fd9-c2e0-11e7-9646-00163e0055c1
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-dns
subjects:
- kind: ServiceAccount
  name: kube-dns
  namespace: kube-system
```

- `kubedns-controller.yaml` 中定义的 Pods 时使用了 `kubedns-sa.yaml` 文件定义的 `kube-dns` ServiceAccount，所以具有访问 kube-apiserver DNS 相关 API 的权限；

### 配置kube-dns ServiceAccount

无需更改

### 配置kube-dns 服务

```shell
$ diff kubedns-svc.yaml.base kubedns-svc.yaml
30c30
<   clusterIP: __PILLAR__DNS__SERVER__
---
>   clusterIP: 10.254.0.2
```

- 需要将 spec.clusterIP 设置为集群环境变量中变量 `CLUSTER_DNS_SVC_IP` 值，这个IP 需要和 kubelet 的 `—cluster-dns` 参数值一致

### 配置kube-dns Deployment

```shell
$ diff kubedns-controller.yaml.base kubedns-controller.yaml
88c88
<         - --domain=__PILLAR__DNS__DOMAIN__.
---
>         - --domain=cluster.local
128c128
<         - --server=/__PILLAR__DNS__DOMAIN__/127.0.0.1#10053
---
>         - --server=/cluster.local/127.0.0.1#10053
160,161c160,161
<         - --probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.__PILLAR__DNS__DOMAIN__,5,A
<         - --probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.__PILLAR__DNS__DOMAIN__,5,A
---
>         - --probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.cluster.local,5,A
>         - --probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.cluster.local,5,A
```

- `--domain` 为集群环境变量`CLUSTER_DNS_DOMAIN` 的值
- 使用系统已经做了 RoleBinding 的 `kube-dns` ServiceAccount，该账户具有访问 kube-apiserver DNS 相关 API 的权限
- 修改拉取镜像的地址类似：

anjia0532/k8s-dns-dnsmasq-nanny-amd64:1.14.5

### 执行所有定义文件

```
$ pwd
/home/ych/k8s-repo/kube-dns
$ ls *.yaml
kubedns-cm.yaml  kubedns-controller.yaml  kubedns-sa.yaml  kubedns-svc.yaml
$ kubectl create -f .
```

### 检查kubedns 功能

```shell
cat > my-nginx.yaml<<EOF
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
EOF
$ kubectl create -f my-nginx.yaml
deployment "my-nginx" created
```

Expose 该Deployment，生成my-nginx 服务

```shell
$ kubectl expose deploy my-nginx
$ kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.254.0.1      <none>        443/TCP   1d
my-nginx     ClusterIP   10.254.32.162   <none>        80/TCP    56s
```

然后创建另外一个Pod，查看`/etc/resolv.conf`是否包含`kubelet`配置的`--cluster-dns` 和`--cluster-domain`，是否能够将服务`my-nginx` 解析到上面显示的CLUSTER-IP `10.254.32.162`上

```shell
$ cat > pod-centosnx.yaml<<EOF
apiVersion: v1
kind: Pod
metadata:
  name: centos
spec:
  containers:
  - name: centos
    image: centos
    ports:
    - containerPort: 80
EOF
$ kubectl create -f pod-centos.yaml
pod "nginx" created
$ kubectl exec  nginx -i -t -- /bin/bash
root@centos:/# cat /etc/resolv.conf
nameserver 10.254.0.2
search default.svc.cluster.local. svc.cluster.local. cluster.local.
options ndots:5
root@nginx:/# ping my-nginx
PING my-nginx.default.svc.cluster.local (10.254.32.162): 48 data bytes
^C--- my-nginx.default.svc.cluster.local ping statistics ---
14 packets transmitted, 0 packets received, 100% packet loss

root@nginx:/# ping kubernetes
PING kubernetes.default.svc.cluster.local (10.254.0.1): 48 data bytes
^C--- kubernetes.default.svc.cluster.local ping statistics ---
6 packets transmitted, 0 packets received, 100% packet loss

root@nginx:/# ping kube-dns.kube-system.svc.cluster.local
PING kube-dns.kube-system.svc.cluster.local (10.254.0.2): 48 data bytes
^C--- kube-dns.kube-system.svc.cluster.local ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
```

在centos内访问my-nginx服务

## 部署Dashboard 插件

官方文件目录：[kubernetes/cluster/addons/dashboard](https://github.com/kubernetes/kubernetes/tree/v1.8.2/cluster/addons/dashboard)

使用的文件如下：

```shell
$ ls *.yaml
dashboard-controller.yaml  dashboard-rbac.yaml  dashboard-service.yaml
```

- 新加了 `dashboard-rbac.yaml` 文件，定义 dashboard 使用的 RoleBinding。

由于 `kube-apiserver` 启用了 `RBAC` 授权，而官方源码目录的 `dashboard-controller.yaml` 没有定义授权的 ServiceAccount，所以后续访问 `kube-apiserver` 的 API 时会被拒绝，前端界面提示：

![403](D:\Github\hexo\source\_posts\assets\dashboard-403.png)403

解决办法是：定义一个名为dashboard 的ServiceAccount，然后将它和Cluster Role view 绑定：

```shell
$ cat > dashboard-rbac.yaml<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dashboard
subjects:
  - kind: ServiceAccount
    name: dashboard
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 配置dashboard-controller

```
	spec:
      serviceAccountName: dashboard
      containers:
```

- 使用名为 dashboard 的自定义 ServiceAccount

### 配置dashboard-service

```shell
  type: NodePort
  ports:
  - port: 80
    targetPort: 9090
    nodePort: 30036

```

- 指定端口类型为 NodePort，这样外界可以通过地址 nodeIP:nodePort 访问 dashboard

### 执行所有定义文件

```shell
$ pwd
/home/ych/k8s-repo/dashboard
docker pull anjia0532/kubernetes-dashboard-amd64:v1.6.3
$ ls *.yaml
dashboard-controller.yaml  dashboard-rbac.yaml  dashboard-service.yaml
$ kubectl create -f  .
```

### 检查执行结果

查看分配的 NodePort

```shell
$ kubectl get services kubernetes-dashboard -n kube-system
NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes-dashboard   NodePort   10.254.104.90   <none>        80:31202/TCP   1m
```

- NodePort 31202映射到dashboard pod 80端口；

检查 controller

```shell
$ kubectl get deployment kubernetes-dashboard  -n kube-system
NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-dashboard   1         1         1            1           3m

$ kubectl get pods  -n kube-system | grep dashboard
kubernetes-dashboard-6667f9b4c-4xbpz   1/1       Running   0          3m
```

