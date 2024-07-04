我们有两套 AKS （Azure Kubernetes Service），一套用于测试环境，另一套则是投产使用。这两套环境一年多的使用下来，大大小小踩了一些坑，这里总结一下。

## 不支持 IPVS

截至目前，AKS 的 kube-proxy 仍旧不能支持 IPVS。相关的 Issuses 开了有一年多之久，尚未见到官方有任何说明。

https://github.com/Azure/AKS/issues/1846

这就是说，较大规模的 AKS 集群会因为 iptables 而产生性能问题，如果遇到这个问题的朋友可以尝试使用 cilium 这个基于 eBPF 实现的 kube-proxy。

https://docs.cilium.io/en/v1.9/gettingstarted/k8s-install-aks/

<aside>
💡 状态更新: 2023-03-09 该功能在海外版 AKS 实现支持，目前尚未 GA

</aside>

!https://blog.motofans.club/static/img/202f808b01689d78b292994b0e97ea09.image.png

:::

## DNS 解析失败

在使用 kubenet CNI 部署的 AKS 上，CoreDNS 中 timeout 的事件常有发生，这是 Kubernetes 的通病，它主要是由 Iptables 中 Conntrack竞争导致的 DNS 超时。

具体而言，在 AKS 上面，解决这个问题有三种办法。

1. 使用 Node-Local DNSCache，
2. 使用 cilium 替换 kube-proxy
3. Azure CNI Transparent Mode

但是，第一种办法 AKS 并没有提供官方支持，只是在这个 Issuse 下面提供了 workaround。

https://github.com/Azure/AKS/issues/1492

然而，测试下来并没有根本的解决问题，再查阅了更多的资料后发现很多使用 kubenet CNI 的人都有同样的问题。于是马上我们就遇到了下面的这个坑。

## CNI 不可以更换

AKS 不支持在已有的集群上调整网络策略的开关，或是切换 CNI 插件。

https://docs.microsoft.com/en-us/azure/aks/use-network-policies

这导致如果我们想要从 kubenet CNI 切换到 Azure CNI 以能够使用 Azure CNI Transparent Mode 的话，就必须另外新建集群了。

## 更新服务主体密钥

AKS 默认签发的服务主体密钥为 1 年，到期后将无法创建新的 PV，LB 等等 Azure 的资源。

Ceate Service-Principal

```bash
az ad sp create-for-rbac --skip-assignment --year 99
```

The SP_ID is your appId, and the SP_SECRET is your password:

```bash
# 这些是示例，使用 <https://www.uuidgenerator.net> 生成
SP_ID=74ab7ddf-5273-49d4-9284-34d5b5013f1e
SP_SECRET=0ec740d5-aca8-48c9-baf5-0fc2818e46b0
```

Excute upgrade, the nodepool will be restart.

```bash
az aks update-credentials \\
  --resource-group <> \\
  --name <> \\
  --reset-service-principal \\
  --service-principal $SP_ID \\
  --client-secret $SP_SECRET
```