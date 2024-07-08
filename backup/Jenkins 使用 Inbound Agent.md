在 Jenkins 使用 distributed builds 时，可能会遇到一些特定的网络环境使得 Jenkins Master 无法通过 SSH Agent 一类的方式实现主动的与 Jenkins Node 建立链接。这时候往往会在 Jenkins Master 上开启一个 TCP Inboud Port， 让 Agent 节点主动连接到 Jenkins Master。这样需要为 Jenkins Master 多开放了一个端口，而如果 Jenkins Master 部署在 Kubernettes 的话，则需要另外引入一个 config-map 来为 Nginx Ingress 增加 TCP 服务暴露。

## 更好的实现

Jenkins 于 2.217 版本的增加了 Websocket 支持。

https://www.jenkins.io/blog/2020/02/02/web-socket/

使用 Websocket，当 Inbound Agent 启动时会与  https://jenkins.yourdomain.com/wsagents/ 建立 Websoket 连接，这是一个双向流式的长连接，它并不需要在 Jenkins Master 上开启更多的端口。

## 启用 Websocket

如果你使用的是 Nginx，请为 PATH: /wsagents/ 打开 websocket 支持，这里不做说明

### 配置 Nginx Ingress

很多资料都提到了 Nginx Ingress 需要添加注解以支持 Websocket，但是下面的 Ingress 示例中并没有添加注解。原因如下：

https://github.com/nginxinc/kubernetes-ingress/issues/322

`The only reason I can think of where this would be the case, is if you are actually running the community ingress controller instead where the Upgrade and Connection headers are added based on NGINX map based conditions. Without the need for any annotation.`

也就是说 community ingress controller 并不需要添加注解。

```yaml
kind: Ingress
apiVersion: extensions/v1beta1
metadata:
  name: jenkins-ingress
  namespace: sre-devops
  annotations:
    kubesphere.io/creator: admin
spec:
  tls:
    - hosts:
        - jenkins.yourdomain.com
      secretName: aks-ingress-tls
  rules:
    - host: jenkins.yourdomain.com
      http:
        paths:
          - path: /
            pathType: ImplementationSpecific
            backend:
              serviceName: jenkins-http-api
              servicePort: 80

```

### 配置 Agent 节点

开启 Websocket 的方式很简单，在 Jenkins Master 中点击添加一个 Node，然后勾选 Use WebSocket 即可。

将 4.0 版本以后的 agent.jar 上传至 Agent 节点，执行启动命令即可。

```bash
java -jar agent.jar -jnlpUrl <https://jenkins.yourdomain.com/computer/test/jenkins-agent.jnlp> -secret @secret-file -workDir "/home/jenkins"
```

现在你可以在 Jenkins Master 中看到一个 Jenkins Agent 上线了。

<!-- ##{"timestamp":1662480000}## -->