---
title: k8s-RBAC配置工具
date: 2020-03-08 15:49:45
tags:
  - RBAC
categories: k8s
---

## permission-manager 简介

`permission-manager` 是一个用于 Kubernetes RBAC 和 用户管理工具。

<!--more-->

- 项目地址

  `https://github.com/sighupio/permission-manager`

- 部署依赖

  ```
  $ kubectl apply -f k8s/k8s-seeds/namespace.yml
  $ kubectl apply -f k8s/k8s-seeds
  ```

- 修改 `Deploy` 必填 `Env` 参数

  | Env 名称              | 描述                                  |
  | --------------------- | ------------------------------------- |
  | PORT                  | 服务器暴露的端口                      |
  | CLUSTER_NAME          | 在生成kubeconfig文件中使用的集群名称  |
  | CONTROL_PLANE_ADDRESS | 在生成kubeconfig文件中的k8s api 地址  |
  | BASIC_AUTH_PASSWORD   | WEB UI 登陆密码（默认用户名为 admin） |

- 部署

  ```
  $ kubectl apply -f k8s/deploy.yaml
  ```

- 访问 WEB UI

  ```
  $ kubectl port-forward svc/permission-manager-service 4000 --namespace permission-manager
  ```

## 如何添加新权限模板

默认只有 `developer` 和 `operation` 模板，模板都是以 `template-namespaced-resources___` 为开头。添加新的权限模板，可以参考 `k8s/k8s-seeds/seed.yml` 文件。