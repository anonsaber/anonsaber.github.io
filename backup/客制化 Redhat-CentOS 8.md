## 背景

Redhat/CentOS 在 release 8 后，没有再提供 minimal 镜像。超微的 IPMI 通过网络挂载 ISO 要求镜像不得大于 4.7GB。

难道要每次使用 Boot 镜像来从网络安装系统吗？亦或是使用 JAVA Client 挂载 8GB 乃至 12GB 的 DVD 镜像（那东西有多卡谁用谁知道）？

## 新的方式

https://www.redhat.com/sysadmin/rhel-image-builder

Redhat 提供了在线 Build 镜像的服务，不仅可以生成适用于 GCP/AWS/Azure 云环境的镜像，还可以生成适用于私有云的 qcow2/vmdk，以及适用于裸机环境的 ISO 镜像。

不仅如此，你还可以在镜像中预置一些你喜欢的 packages，比如 memcached 或是 curl 等等，如果是制作 Redhat 镜像，你也可以将 Redhat 账号预先附加到镜像当中。

你要做的仅仅是简单的配置，然后点击生成按钮，等待一段时间后即可下载你需要的镜像。

!https://blog.motofans.club/static/img/bf1dd60404ab6f122fa66b45af688ee8.image.png

## 美中不足

这项服务没有提供将 kickstart 自动装机脚本一同打包到 ISO 镜像当中的服务。如果我们希望获得一个无人值守的 ISO 安装镜像，那么 kickstart 是必不可少的。

## 更进一步

使用 mkksiso 将生成的镜像改造成 kickstart 安装镜像。

```bash
yum install lorax -y
```

将你准备好的 kickstart 文件与生成的 ISO 镜像放置在同一个机器当中，执行下面的命令以生产新的镜像。

```bash
mkksiso /PATH/TO/KICKSTART /PATH/TO/ISO /PATH/TO/NEW-ISO
```

<!-- ##{"timestamp":1683734400}## -->