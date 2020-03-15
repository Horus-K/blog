---
title: k8s-yaml-nginx
date: 2020-03-09 21:08:28
tags:
- yaml
- nginx
categories: k8s
---

nginx

<!--more-->

```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  labels:
    app: linux-nginx-deployment-label
  name: linux-nginx-deployment
  namespace: linux
spec:
  replicas: 1
  selector:
    matchLabels:
      app: linux-nginx-selector
  template:
    metadata:
      labels:
        app: linux-nginx-selector
    spec:
      containers:
      - name: linux-nginx-container
        image: harbor.qh.net/linux/nginx-web1:2019-05-03_18_26_52
        #command: ["/apps/tomcat/bin/run_tomcat.sh"]
        #imagePullPolicy: IfNotPresent
        imagePullPolicy: Always
        ports:
        - containerPort: 80
          protocol: TCP
          name: http
        - containerPort: 443
          protocol: TCP
          name: https
        env:
        - name: "password"
          value: "123456"
        - name: "age"
          value: "18"
        resources:
          limits:
            cpu: 2
            memory: 2Gi
          requests:
            cpu: 500m
            memory: 1Gi

        volumeMounts:
        - name: linux-images
          mountPath: /usr/local/nginx/html/webapp/images
          readOnly: false
        - name: linux-static
          mountPath: /usr/local/nginx/html/webapp/static
          readOnly: false
      volumes:
      - name: linux-images
        nfs:
          server: 192.168.7.108
          path: /data/k8sdata/linux36/images 
      - name: linux-static
        nfs:
          server: 192.168.7.108
          path: /data/k8sdata/linux36/static





---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: linux-nginx-service-label
  name: linux-nginx-service
  namespace: linux
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30002
  - name: https
    port: 443
    protocol: TCP
    targetPort: 443
    nodePort: 30443
  selector:
    app: linux-nginx-selector
```

