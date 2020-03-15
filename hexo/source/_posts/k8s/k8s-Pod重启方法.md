---
title: k8s-Pod重启方法
date: 2020-03-08 15:52:30
tags:
  - k8s
categories: k8s
---

# 方法 1

有最新的 yaml 文件。

```
kubectl replace --force -f xxxx.yaml
```

<!--more-->

# 方法 2

没有 yaml 文件，但是使用的是 Deployment 对象。

```
kubectl scale deployment esb-admin --replicas=0 -n {namespace}
```

由于 Deployment 对象并不是直接操控的 Pod 对象，而是操控的 ReplicaSet 对象，而 ReplicaSet 对象就是由副本的数目的定义和Pod 模板组成的。所以这条命令分别是将ReplicaSet 的数量 scale 到 0，然后又 scale 到 1，那么 Pod 也就重启了。

# 方法 3

同样没有 yaml 文件，但是使用的是 Deployment 对象。

使用命令

```
kubectl delete pod {podname} -n {namespace}
```

# 方法 4

没有 yaml 文件，直接使用的 Pod 对象。

使用命令

```
kubectl get pod {podname} -n {namespace} -o yaml | kubectl replace --force -f -
```

