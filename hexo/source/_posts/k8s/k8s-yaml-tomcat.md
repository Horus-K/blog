---
title: k8s-yaml-tomcat
date: 2020-03-09 21:07:15
tags:
- yaml
- tomcat
categories: k8s
---

k8s-yaml-tomcat

<!--more-->

```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  labels:
    app: linux-tomcat-app1-deployment-label
  name: linux-tomcat-app1-deployment
  namespace: linux
spec:
  replicas: 1
  selector:
    matchLabels:
      app: linux-tomcat-app1-selector
  template:
    metadata:
      labels:
        app: linux-tomcat-app1-selector
    spec:
      containers:
      - name: linux-tomcat-app1-container
        image: harbor.qh.net/linux36/tomcat-app1:2019-05-03_11-29-30 
        #command: ["/apps/tomcat/bin/run_tomcat.sh"]
        #imagePullPolicy: IfNotPresent
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          protocol: TCP
          name: http
        env:
        - name: "password"
          value: "123456"
        - name: "age"
          value: "18"
        resources:
          limits:
            cpu: 2
            memory: "2048Mi"
          requests:
            cpu: 500m
            memory: "1024Mi"
        volumeMounts:
        - name: linux-images
          mountPath: /data/tomcat/webapps/myapp/images
          readOnly: false
        - name: linux-static
          mountPath: /data/tomcat/webapps/myapp/static
          readOnly: false
      volumes:
      - name: linux-images
        nfs:
          server: 192.168.7.108
          path: /data/k8sdata/linux/images
      - name: linux36-static
        nfs:
          server: 192.168.7.108
          path: /data/k8sdata/linux/static
      nodeSelector: #位置在当前containers参数结束后的部分
        project: linux36 #指定的label标签     


---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: linux-tomcat-app1-service-label
  name: linux-tomcat-app1-service
  namespace: linux
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
    nodePort: 30003
  selector:
    app: linux-tomcat-app1-selector
```

