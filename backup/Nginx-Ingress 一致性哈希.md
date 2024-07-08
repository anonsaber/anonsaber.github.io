有没有办法让同一个客户端的请求落在同一个 Pod 上面？

在回答这个问题之前我们先来看看 Nginx-Ingress 是如何工作的。

## Nginx-Ingress

Nginx-Ingress 是使用 Nginx 作为反向代理和负载均衡器的 Kubernetes Ingress 控制器， 作为 Pod 执行在 kube-system 命名空间。

当我们在配置一个 Nginx-Ingress 资源时，backend 往往会是一个 serviceName，这会另很多人产生错觉，会认为当客户端的流量打到 Nginx-Ingress 时，Nginx-Ingress 会把流量打到 对对应的 Services 上面，再由 Services 进行负载。

而实际上，***Nginx-Ingress 并不使用 Services 路由流量到 Pods***

关于这一点，可以看看这篇官方发布的: Why endpoints and not services.

https://kubernetes.github.io/ingress-nginx/user-guide/miscellaneous/

观察 Nginx-Ingress Controller 中生成的 nginx.conf 可以发现，Nginx-Ingress 是通过Lua 动态获取的 Endpoints 数据。

## 一致性哈希

由于会访问到哪个 Pod 这件事儿是 Nginx-Ingrss 所决定的，那么一致性哈希的实现也一定是在 Nginx-Ingress 的配置上了。

```yaml
nginx.ingress.kubernetes.io/upstream-hash-by: "$host$request_uri"
```

<!-- ##{"timestamp":1640448000}## -->
