---
title: k8s-ClusterRole
date: 2020-03-08 16:38:50
tags:
- k8s
- yaml
categoris: k8s
---

k8s ClusterRole yaml 模板

<!--more-->

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: template-namespaced-resources___quhui
rules:
  - apiGroups:
      - "*"
    resources:
      # - "bindings"
      - "configmaps"
      - "endpoints"
      # - "limitranges"
      - "persistentvolumeclaims"
      - "pods"
      - "pods/log"
      - "pods/portforward"
      - "podtemplates"
      - "replicationcontrollers"
      - "resourcequotas"
      - "secrets"
      # - "serviceaccounts"
      - "services"
      # - "controllerrevisions"
      # - "statefulsets"
      # - "localsubjectaccessreviews"
      # - "horizontalpodautoscalers"
      # - "cronjobs"
      # - "jobs"
      # - "leases"
      - "events"
      - "daemonsets"
      - "deployments"
      - "replicasets"
      - "ingresses"
      - "networkpolicies"
      - "poddisruptionbudgets"
      # - "rolebindings"
      # - "roles"
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

```

