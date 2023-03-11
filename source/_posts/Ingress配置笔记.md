---
title: Ingress配置笔记
date: 2019-12-18 16:52:56
tags:
  - Ingress
  - Kubernetes
categories:
  - 技术
---

记录了一些自己在使用ingress所用到的ConfigMap

> 相关官方文档
[https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/)

## 413 Request Entity Too Large

- nginx-configuration

`proxy-body-size: 0`

> 根据实际情况修改大小

## API的访问控制

- nginx-configuration

```yaml
add-headers: ingress-nginx/custom-headers
```

> custom-headers为自定义ConfigMap名字，命名空间与ingress所在的相同

- custom-headers

```yaml
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Origin: $http_origin
```

## 四层负载均衡设置

- tcp-services & udp-services

```yaml
# 端口号: 命名空间/服务:端口
# e.g.
"80": namespace/service:80
```
