---
title: Kubernetes 中的 GRPC 服务发现与 LB 玩法
date: 2023-03-10 09:30:00
tags:
  - 微服务
  - GRPC
  - Kubernetes
categories:
  - 技术
---

## 本文你讲看到（技术栈

- 集群界大哥 Kubernetes
- 微服务 RPC 大哥 GRPC

## 通讯原理

![来自: colobu.com](https://colobu.com/2017/03/25/grpc-naming-and-load-balance/4.png)

微服务使用 GRPC 连接，我们一般使用 DNS 服务发现让 GRPC 客户端找到服务端。

在 Kubernetes 中，我们可以使用 Headless Service 来实现 DNS 服务发现。

而我们 GRPC 客户端仅需写好 Service 的 DNS 即可让客户端找到对应集群所在的服务端 IP

## 问题所在？

如果在 GRPC 的默认设置中，我们的客户端会随机从 DNS 选择一个服务端 IP 进行连接。

而后续的请求会使用这个缓存的 IP ，请求就会一直请求到同一个服务端上。

这对后续我们进行的滚动更新也造成了一定的影响

> 当缓存的 IP 对应容器已关闭则导致请求失败

## 解决方案

我们可以通过配置 GRPC 的负载均衡策略来解决这个问题。

仅需简单的几个咒语（以 Golang 为例）

```golang
opts := []grpc.DialOption{
    // ...
    // Config LB policy
    grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`),
}

// Initialize
if conf.Dial > 0 {
    var cancel context.CancelFunc
    ctx, cancel = context.WithTimeout(ctx, conf.Dial)
    defer cancel()
}

// Use dns with balancer
dnsTarget := "dns:///" + target
conn, err := grpc.DialContext(ctx, dnsTarget, opts...)
if err != nil {
    panic(err)
}
```

## 写的什么 JB ？

1. 配置 `grpc.WithDefaultServiceConfig` 来配置负载均衡策略。
2. 使用 `dns:///` 来指定 DNS 服务发现。

这样就可以让 GRPC 客户端从 DNS 中获取所有服务端 IP 并进行轮询访问了。

## 小尾巴

GRPC 搭配 Kubernetes 还有很多细节。

后续我们来聊聊如何让程序适配 Kubernetes 滚动更新方式。