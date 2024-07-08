尽管 S3 似乎已经成为云存储的事实标准，不过 Azure Blob 并不支持。Minio 以前有个 Azure Blob 的网关服务，随着后面的更新也不再支持。

## S3Proxy

s3proxy 提供了一种在 Microsoft Azure 上实现 S3 API 兼容性的方法。详细的步骤和解决方案可在以下链接中找到：

[S3 API compatibility on Microsoft Azure](https://ventral.digital/posts/2020/10/11/s3-api-compatibility-on-microsoft-azure/)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: s3proxy
---
apiVersion: v1
kind: Service
metadata:
  name: s3proxy
spec:
  selector:
    app: s3proxy
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: s3proxy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: s3proxy
  template:
    metadata:
      labels:
        app: s3proxy
    spec:
      containers:
        - name: s3proxy
          image: andrewgaul/s3proxy:s3proxy-1.9.0
          ports:
            - containerPort: 80
              protocol: TCP
          env:
            - name: JCLOUDS_PROVIDER
              value: azureblob
            - name: JCLOUDS_REGIONS
              value: homelab
            - name: JCLOUDS_IDENTITY
              value: azure_storage_account_name
            - name: JCLOUDS_CREDENTIAL 
              value: azure_storage_account_key
            - name: JCLOUDS_ENDPOINT
              value: https://azure_storage_account_name.blob.core.chinacloudapi.cn
            - name: S3PROXY_IDENTITY
              value: custom_aws_access_key_id
            - name: S3PROXY_CREDENTIAL
              value: custom_aws_secret_access_key
```

其中，JCLOUDS_IDENTITY 是 Azure Storage Account 的 Name，对应的 JCLOUDS_CREDENTIAL 是 Azure Storage Account 的 Access KEY，由于 s3proxy 方面没有很好的处理特殊字符，请务必轮询出来一个没有斜杠等特殊字符的 Access KEY。

而 S3PROXY_IDENTITY 与 S3PROXY_CREDENTIAL 则是 AWS CLI 类似的工具在访问 S3Proxy 时所需要的。

## 测试 S3Proxy

### 创建一个 blob

在 Azure blob 中创建一个 Blob，比如名为 ares-test

### 向 blob 写入文件

本文以 AWS CLI 为例进行测试，请自行安装。

首先编辑 `~/.aws/credentials` 配置文件并添加以下行：

```
[default]
aws_access_key_id = custom_aws_access_key_id
aws_secret_access_key = custom_aws_secret_access_key
```

然后将以下行添加到您的 `~/.aws/config` 文件中：

```
[default]
endpoint_url=http://s3proxy
region=homelab
```

执行文件上传

```bash
aws s3 cp test-file.csv s3://ares-test/
```

<!-- ##{"timestamp":1694448000}## -->