---
title: 记近期Rancher中的采坑
date: 2019-10-19 20:30:33
tags:
- Rancher
- Kubernetes
---

近期由于业务需求，需要在Windows系统上部署Rancher，由于Windows中的Docker比较多坑。在解决了问题后顺便记录一下填充一下博客😔

> 系统：Windows Server 2018
> Docker: 18.09
> Kubernetes: 1.10
> Rancher: 2.2.8
> 规模：单机

## Rancher访问

由于是单机部署k8s和Rancher，为了防止Rancher不阻碍Ingress的端口访问，将443端口映射到其他端口上。

```shell
docker run -d --restart=unless-stopped \
-p <主机端口>:80 -p <主机端口>:443 \
-v <主机路径>:/var/lib/rancher/ \
-v /root/var/log/auditlog:/var/log/auditlog \
-e AUDIT_LEVEL=3 \
rancher/rancher:stable
```

在我们部署完Rancher后，再用Ingress给Rancher做七层负载均衡。

> 注意：
>1.由于Rancher是独立容器部署，端口直接暴露到物理主机上，所以我们需要将后端设置成`物理主机的IP + HTTP端口`
> 2.由于做了七层代理后SSL证书也是使用了Cert-Manager来管理，CA根证书也就不需要设置了。
> 3.Rancher提供的集群Kubeconfig文件删掉CA证书的字段即可正常访问😁

## Rancher中 Cert-Manager 应用

Cert-Manager将会自动为我们的集群中使用的域名进行签证

应用安装后会默认设置一个集群签证者，这个已经够我们使用了。

我们需要在相关的负载均衡进行设置

1. 添加标签 `kubernetes.io/tls-acme: "true"` 来触发Cert-Manager进行自动签证

2. Ingress的设置

```yaml
spec:
  tls:
  - hosts:
    - host.example.com
    secretName: host-example-crt
```

如果我们有多个集群，需要添加标签
`certmanager.k8s.io/cluster-issuer: 集群名(Rancher中)`

## 关于Rancher中的PVC

在使用`hostpath`的存储类时，由于使用的是Windows系统，文件将会直接存储在系统中。在使用时请注意权限问题，如果是对于权限有严格限制的程序将会出现报错。

> 权限问题是特大坑！🤮不过听说Windows
