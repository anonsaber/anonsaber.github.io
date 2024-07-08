## 背景

家里面没有固定的公网 IP 地址，路由器又不是最外面的一层，因此 DDNS 也没法搞。

一直以来使用 Wireguard 进行组网。然而，如果想要多个服务都使用 80/443 端口，需要配置一个 Nginx，还需要考虑使用 Certbot 进行自动申请证书。每次更新一个服务都需要调整 Nginx 的配置。使用 FRP 或是 Ngrok 也会有同样的问题。

## Boringproxy

偶然发现了 [Boringproxy](https://boringproxy.io) 这个东西。先看看它自己是怎么介绍的：

> boringproxy is a combination of a reverse proxy and a tunnel manager.
> 
> What that means is if you have a self-hosted web service (Nextcloud, Emby, Jellyfin, etherpad, personal website, etc.) running on a private network (such as behind a [[NAT](https://en.wikipedia.org/wiki/Network_address_translation)](https://en.wikipedia.org/wiki/Network_address_translation) at home), boringproxy aims to provide the easiest way to securely (i.e. HTTPS and optional password-protection) expose that server to the internet, so you can access it from anywhere.
> 

简单来说就是，它能同时帮你搞定 HTTPS 证书的申请（使用 ACME HTTP-01 challenge）以及跨 NAT 的内网穿透。

## 优势

只需要 client 和 server 之间建立链接，剩下的配置都可以在 1 分钟内在网页端完成。

## 劣势

目前没有很好的断线重连机制，不支持 UDP，暂时不建议用在生产环境。

## 部署

由于 Release 的版本功能有所缺失，建议拉取 master 分支自行 build。你也可以使用我 Build 的镜像，当然不要忘记在服务器端要开启防火墙的端口放行规则。

### Server

```yaml
version: '2'

services:
  boringproxy:
    container_name: boringproxy
    image: public.ecr.aws/motofansd/boringproxy:v0.9.1
    restart: unless-stopped
    network_mode: host
    environment:
      USER: "root"
      B_SERVER: "test.motofans.club"
      B_MAIL: "test@motofans.club"
      B_STOR: "/storage/certmagic"
      SSH_PORT: "2222"
    volumes:
      - /etc/ssl/certs/:/etc/ssl/certs/:ro
      - ./.state/storage:/storage
      - ./.state/ssh:/root/.ssh/
```

### Client

```yaml
version: '2'

services:
  boringproxy:
    container_name: boringproxy
    image: public.ecr.aws/motofansd/boringproxy:v0.9.1
    restart: unless-stopped
    network_mode: host
    environment:
      USER: "root"
      SERVER: "false"
      B_MAIL: "test@motofans.club"
      B_STOR: "/storage/certmagic"
      B_CLIENT: "jackal"
      B_SERVER: "test.motofans.club"
      B_USER: "admin"
      B_TOKEN: "xxxxxxxxxxxxxxxxxxxxxxxxxxx" # 从 server 端的启动日志里面获取
    volumes:
      - .state/storage:/storage'
      - /etc/ssl/certs/:/etc/ssl/certs/:ro
```

目前 proxmox.motofans.club 使用 Boringproxy 进行暴露

<!-- ##{"timestamp":1660579200}## -->