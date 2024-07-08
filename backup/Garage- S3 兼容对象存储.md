## 背景

一直以来，要怎么能够让在 EPIC-KBS9 上部署的 Seafile 能够稳定高效的访问到 Riverbed CX570 的 ZFS 阵列都是让我觉得需要改进的地方。Samba/GlusterFS/JuiceFS/BeeGFS 使用 FUSE Mount 的方式，性能损失较大。而 iSCSI 的方式又十分的不灵活，扩容也不够简单。至于 Minio，这东西 BugFix 的频率让我总是怀疑它是否应该使用在生产环境（尽管很多企业都将其开源版本部署在自家服务器）上。

## [Garage](https://garagehq.deuxfleurs.fr/)

终于，在某一天我找到了 [Garage](https://garagehq.deuxfleurs.fr/)，先来看看其官网对自己的表述以及愿景

https://garagehq.deuxfleurs.fr/documentation/design/goals/

Garage 使用 Rust 实现，性能十分优秀，你能在其官网上找到它的性能对比，这里不做过多表述。

> [!IMPORTANT]
> Garage 是一个优劣十分明显的分布式存储系统，如果你计划将其使用在生产环境中，请务必深刻理解 Garage。
> 
> 本文不演示如何使用 Garage，这一点在 Garage 的官方文档中写的十分详细，请直接查看官方文档。

## 优势

对于 Homelab 来说，很难实现像数据中心一样稳定的网络和电力环境。Garage 最为吸引我的一点就是它**对网络故障、网络延迟、磁盘故障、系统管理员故障具有高度弹性**。与 Minio 使用纠删码进行工作不同的是，Garage 对网络延迟的要求远远没有那么苛刻（这是因为 Garage 的愿景不是设计一个拥有纠删码的分布式文件系统）

## 劣势

与 Glusterfs 复制卷相同，没有设计纠删码，Garage 使用仲裁机制进行读取和访问，因此存在脑裂风险。对于维护 Garage 来说，备份节点或是镜像（Rclone）的存在是必须的。在我的使用场景，搭配 ZFS 的快照功能，Garage 能够很好的保障数据的安全性。

## 结合使用

Garage + SFTPGo

Garage + Seafile

Garage + Minio Gateway（Garage 仅支持 V4 版本的签名，如需 V2 版本，可使用 Minio Gateway 进行转换）

<!-- ##{"timestamp":1667836800}## -->