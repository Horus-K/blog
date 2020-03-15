---
title: k8s-service模板
date: 2020-03-08 12:55:45
tags:
  - yaml
  - k8s-service
categories: k8s
---

service模板

<!--more-->

```
apiVersion: v1
kind: Service
metadata:
  annotations:
  name: my-nginx
  namespace: mynginx
  labels:
    run: my-nginx
spec:
  clusterIP: 10.254.163.198
  type: {NodePort|NodePort|LoadBalancer|ExternalName}
  ports:
  - name: http
    NodePort:30001
    port: 8080
    protocol: TCP
    targetPort: 80
  selector:
    run: my-nginx
```



