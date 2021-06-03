# SpringBoot 项目优雅关闭

目前 Spring Boot 已经发展到了 2.4.5.RELEASE，而伴随着 2.3.0版本的到来，优雅关闭机制也更加完善了。

目前版本的 Spring Boot，对 Jetty, Tomcat 等WEB容器都支持优雅关闭。

## 最佳实践

### SpringBoot优雅关闭

升级Spring Boot至2.3.0以上版本并添加相关配置：

```yaml
server:
  shutdown: graceful  #启用优雅关闭

spring:
  lifecycle:
    timeout-per-shutdown-phase: 20s #强制终止的宽限时间
```

### Kubernetes 优雅的停止 Pod

我们要优雅的停止服务，先要了解 Kubernetes 是如何优雅的停止 Pod的。下面我们先了解一下 Pod 优雅停止的过程。

当我们 kill 掉一个 Pod 的时候，会执行下面的流程：  
- Pod 会先被标记为 Terminating
- Service（假如有的话）会把这个 Pod 从 Endpoint 中摘掉，这样子负载均衡不会再给这个 Pod 流量
- 看是否有 preStop，如果定义了，就执行他
- 之后给 Pod 发 SIGTERM 信号让 Pod 中的所有容器优雅退出
 
但是实际情况肯定不是这样子的，我们可能会遇到以下情况：  

- 容器里的业务代码根本不回处理 SIGTERM 信号
- 容器里的业务代码有处理优雅推出的逻辑，但是有问题导致一直退出不了
- 容器里的业务已经卡死，导致资源使用满了，处理不了优雅退出的代码逻辑或需要很久才能处理完成

为了处理以上这几种无法退出的情况，Kubernetes 还有一个 terminationGracePeriodSeconds 的硬停止时间，默认是 30s，如果 30s 内还是无法完成上述的过程，那就就会发送 SIGKILL，强制干掉 Pod。

通过上面的描述来看，想要优雅的停止 Pod ，我们可以做以下2点：  

- 定义 preStop 操作
- 处理 SIGTERM 信号


最佳实践：结合SpringBoot的优雅关闭，为容器加上PreStop，这里我们直接sleep 10s
```yaml
...
    spec:
      containers:
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8081
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8081
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 10"]

...

```
