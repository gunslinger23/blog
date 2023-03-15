---
title: Kubernetes 滚动更新正确玩法
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

## 正确玩法？

想要正确的使用滚动更新，我们需要注意以下几点

1. 新副本健康状态必须正确，这样才能保证新副本可以正常接收请求，否则出现新副本请求失败问题，造成更新过程中服务不可用
2. 旧副本被删除前必须先终止所有新的请求，并等待现有请求处理完再进行关闭
3. 副本健康状态转健康后，Service 需要时间去更新状态，新的 Pod IP 仍未加入到 Endpoint 中，旧的 Pod IP 被终结仍在 Endpoint 中，这都会造成请求失败

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

### 3.1. Service 更新慢

> 此情况仅在使用 Kubernetes Service 进行服务发现才会遇到

当服务状态变成健康，Service Endpoint 加入新 IP 仍需要有一段的时间。

而终结旧服务从 Endpoint 删除也需要一定的时间。

#### 如果我们需要加快这个速度可以通过配置实现

```
--node-monitor-grace-period duration
# Default: 40s | Amount of time which we allow running Node to be unresponsive before marking it unhealthy. Must be N times more than kubelet's nodeStatusUpdateFrequency, where N means number of retries allowed for kubelet to post node status.
```

> 相关文档: https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/

#### 等待旧服务 Endpoint 删除再关闭服务请求

在服务捕获终止信号后可以多等待几秒，再进请求服务等关闭，防止仍有新请求不可用

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

### 3.2. 我还是请求到旧的怎么办？

#### GRPC 客户端设置重试机制

既然 GRPC 有这个中间件，那服务请求失败我们可以多来几次，代价就是慢一点点 🤏🤏

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
if err != nil {
    panic(err)
}
```
