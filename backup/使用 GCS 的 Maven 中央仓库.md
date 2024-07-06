本文最后修订时间为 2022 年 07 月 25 日，以下描述基于本文最后修订时间可能会有调整。

## 前言

[Maven Central](http://repo1.maven.org) 的服务器地址位于美国，即便是使用亚太的代理去访问有时候速度也是不怎么理想。

而国内的镜像站关的关，慢的慢，总之没有十分理想的替代品。

所幸的是，Apache Maven 的发起人 Jason van Zyl 创建了一个完整的中央仓库的镜像并托管在了 Google 云存储上。

## 项目地址

https://storage-download.googleapis.com/maven-central/index.html

Jason van Zyl 在 Blog 里面介绍了为什么要创建这个中央仓库的副本。

http://takari.io/2015/10/28/google-maven-central.html

得益于 GCS 覆盖了美国，欧洲，亚太的服务器节点，这个仓库要比 [Maven Central](http://repo1.maven.org) 的速度快得多。