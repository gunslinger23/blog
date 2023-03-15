---
title: 应用如何在 Kubernetes 正确滚动更新
date: 2023-03-15 10:00:00
tags:
  - 微服务
  - Kubernetes
categories:
  - 技术
---

## 无责声明（不是

其实本文只是基于 NewPage 微服务实践中的一些经验总结出来的，并不代表所有情况都适用，仅提供解决问题的思路。实际不同业务架构，部署方式都有不同，需要根据实际情况进行调整。

希望借此机会和大家一起探讨怎么优雅完成滚动更新，如还有其他更好的解决方案，欢迎留言交流。

## 滚动更新原理

> 没错，抄的 Kubernetes 官方文档的 👉 [滚动更新](https://kubernetes.io/zh-cn/docs/tutorials/kubernetes-basics/update/update-intro/)，觉得废话可以直接跳过

![](https://d33wubrfki0l68.cloudfront.net/30f75140a581110443397192d70a4cdb37df7bfc/fa906/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates1.svg)

服务A 在3个节点上运行，一共有4个副本，这时候我们需要更新服务A的副本

![](https://d33wubrfki0l68.cloudfront.net/678bcc3281bfcc588e87c73ffdc73c7a8380aca9/703a2/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates2.svg)

这时新增加了一个副本，拥有了新的 IP `10.0.0.6`，等待新副本健康状态变为 **健康** 后，旧副本会被删除

![](https://d33wubrfki0l68.cloudfront.net/9b57c000ea41aca21842da9e1d596cf22f1b9561/91786/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates3.svg)

同样其他节点也进行类似的操作

![](https://d33wubrfki0l68.cloudfront.net/6d8bc1ebb4dc67051242bc828d3ae849dbeedb93/fbfa8/docs/tutorials/kubernetes-basics/public/images/module_06_rollingupdates4.svg)

最终全部容器都更新完毕

## 应用正确玩法？

### 前提

先确认一点，Kubernetes 仅仅是一个容器编排平台，它不会关心你的应用是什么，它只会关心你的应用是否健康，是否可以接收请求，是否可以正常工作。

以下是应用所需要负责的事情

1. 告诉 Kubernetes 我的应用是否健康
2. 正确处理 Kubernetes 发送的终止信号，尽快结束服务生命周期
3. 正确处理请求流量，保证平滑无缝的滚动更新

### 1. 健康检查

想要让 Kubernetes 正确识别到服务状态健康，必须要自己进行 readiness 配置

例如我们服务主动提供一个健康检查接口

```golang
// Health Check
func (s *Server) HealthCheck(health func() bool) {
    s.healthFunc = health
    mux := http.NewServeMux()
    // Request path
    mux.HandleFunc("/healthz", HttpHealthHandler(s.healthFunc))
    s.healthCheck = &http.Server{
        Addr:    "0.0.0.0:888", // Listen IP Port
        Handler: mux,
    }
    // Http server
    go func() {
        err := s.healthCheck.ListenAndServe()
        if err != nil && err != http.ErrServerClosed {
            panic(err)
        }
    }()
}

// Real Service
func run() {
    server.HealthCheck(func () bool {
        // Check service health
        return true
    })
}
```

然后在 Kubernetes 中配置 readiness probe

```yaml
# deployment.yaml
# somethings...
containers:
  - name: Hello
    readinessProbe:
    httpGet:
        path: /healthz
        port: 888
    initialDelaySeconds: 5
    periodSeconds: 1
    successThreshold: 2
    failureThreshold: 3
```

如果我们其他服务发现，我们则需要 Kubernetes 容器健康状态与服务发现状态保持一致

例如 Consul 我们仅需要将配置 `connectInject.enabled` 改成 `true` 即可

> 注意: Consul 需要使用 k8s 部署方式才支持设置
> 相关文档: https://developer.hashicorp.com/consul/docs/k8s/connect

### 2. 优雅关闭

服务使用什么协议就应该按照他们的优雅关闭方式去做

如果使用 Grpc 服务，我们 Golang 可以使用 `grpcServer.GracefulStop()` 进行优雅关闭

如果使用 WebSocket ，我们需要客户端和服务端进行配合，又服务端发送新节点转移消息，等待客户端正确处理转移关闭连接。当然其中还有很多细节需要处理才能保证服务可用性

如果使用 Kubernetes Service 进行服务发现，Endpoint 上的更新是有延迟，仍会有流量进入到被终结的服务上

这个时候我们可以采取一下措施

1. 等待 Endpoint 更新再关闭服务请求
2. 客户端请求添加重试，服务 IP 不可用将继续访问下一个 IP

#### 我知道你很急，你等一等

在服务捕获终止信号后可以多等待几秒，再关闭服务请求

```golang
c := make(chan os.Signal, 1)
signal.Notify(c, syscall.SIGHUP, syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT)
for {
    s := <-c
    fmt.Println("Get a signal", s.String())
    switch s {
    case syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT:
        fmt.Println("Waiting for terminating")
        // Waiting k8s deal with terminating
        time.Sleep(time.Second * 10)
        // Close all service
        // Close somethings...
        fmt.Println(name, "exit")
        return
    case syscall.SIGHUP:
    default:
        return
    }
}
```

#### 要不你再试一试？

这个是 `正确处理更新时请求流量` 要讲的，请看下面

### 3. 正确处理更新时请求流量

服务开始更新，如何让流量快速正确切换到新服务上是关键，也是提供丝滑的重要因素

一些架构会在入口新增 **网关** 用于流量入口控制

这时在网关和应用的请求客户端都需要我们添加正确的配置，减少请求失败率

#### GRPC 客户端设置重试机制

既然 GRPC 有这个中间件，那服务请求失败我们可以多来几次，代价就是慢一点点 🤏🤏

既然是多试几次，如果每次都试同一个IP那就毫无意义，这就必须搭配 LB 配置

其中的 LB 设置请看之前的文章介绍

```golang
retry := []grpc_retry.CallOption{
    grpc_retry.WithMax(3),
    grpc_retry.WithCodes([]codes.Code{
        codes.Canceled
        codes.DataLoss,
        codes.Unavailable,
    }),
}
opts := []grpc.DialOption{
    grpc.WithUnaryInterceptor(
        grpc_middleware.ChainUnaryClient(
            grpc_retry.UnaryClientInterceptor(retry...),
        )),
    grpc.WithStreamInterceptor(
        grpc_middleware.ChainStreamClient(
            grpc_retry.StreamClientInterceptor(retry...),
        ),
    ),
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
```
