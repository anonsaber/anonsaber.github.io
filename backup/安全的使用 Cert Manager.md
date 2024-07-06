## 前言

在 Kubernetes 上使用 Cert Manager 的 ACME DNS Challenge 方式能够很轻松的为域名签发一张受信任的泛域名证书。然而由于 ACME-DNS 为了能够验证域名的所有权需要修改 DNS 记录，因此需要提供 DNS 服务商的 API Secret，如果它被泄漏了呢？

## 更安全的实现

简单来说就是实现通过 A 域名为 B 域名申请证书。

在下面的例子中，我们使用 `acme.exampleacmeverify.info` 为 `example.net` 生成一张泛域名证书。你可以将 `acme.exampleacmeverify.info` 单独托管到一个 DNS 服务商（本文中将其单独托管到阿里云），从而无需使用 `example.net` 的 API Secret.

### Cert Manager

在 Kubernetes 上部署 Cert Manager

https://cert-manager.io/docs/installation/helm/

### 添加解析

在 `example.net` 的域名下添加一个 CNAME 解析。

```bash
_acme-challenge CNAME _acme-challenge.acme.exampleacmeverify.info.
```

在 `exampleacmeverify.info` 的域名下添加一个 NS 解析。

```bash
acme NS ns1.alidns.com.
acme NS ns2.alidns.com.
```

### 使用 AliDNS-Webhook

本文以 AliDNS 为例，你也可以使用 CloudFlare.

在阿里云上，添加子域 `acme.exampleacmeverify.info`

💡 添加子域的时候，需要在主域 exampleacmeverify.info 添加 TXT 验证记录

生成能够管理子域 `acme.exampleacmeverify.info` 的 API Secret，随后在 Kubernetes 中创建一个 Secret，用于管理 AliDNS

```Bash
kubectl -n cert-manager create secret generic alidns-secret --from-literal=access-key='YOUR_ACCESS_KEY' --from-literal=secret-key='YOUR_SECRET_KEY'
```

部署 alidns-webhook

```Bash
wget https://raw.githubusercontent.com/pragkent/alidns-webhook/master/deploy/bundle.yaml
```

*获取下来的 yaml 不要直接用，要将里面的示例信息换掉*

```Bash
sed -i 's|acme.yourcompany.com|acme.exampleacmeverify.info|g' bundle.yaml
```

*有必要的话，可以把里面涉及的 Images Pull 下来，并传到自己的镜像仓库。*

```Bash
sed -i 's|pragkent/alidns-webhook|nexus.motofans.club/infra-pub/pragkent/alidns-webhook|g' bundle.yaml

```

### 创建 Clusterissuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: motofans-prod-issuer
spec:
  acme:
    email: reg@motofans.club
    server: <https://acme-v02.api.letsencrypt.org/directory>
    privateKeySecretRef:
      name: motofans-prod-issuer-priv-key
    solvers:
    - dns01:
        cnameStrategy: Follow
        webhook:
          groupName: acme.exampleacmeverify.info
          solverName: alidns
          config:
            region: ""
            accessKeySecretRef:
              name: alidns-secret
              key: access-key
            secretKeySecretRef:
              name: alidns-secret
              key: secret-key
```

### 申请证书

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-net-tls
  namespace: sre
spec:
  secretName: example-net-tls
  dnsNames:
  - "*.example.net"
  issuerRef:
    kind: ClusterIssuer
    name: motofans-prod-issuer
```