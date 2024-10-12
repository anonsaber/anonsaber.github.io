在生产环境中运行 zookeeper 需要考虑的关键点。

https://docs.confluent.io/platform/current/zookeeper/deployment.html

## 硬件层面

### 内存

Zookeeper Cluster 中的每个 Zookeeper 节点在任何时间都将所有 znode 内容保存在内存中。在生产用例中，至少应该有 4 GB 的 RAM 专供 Zookeeper 使用。此外必须要注意的是，Zookeeper 对 SWAP 很敏感，任何运行 Zookeeper 服务器的主机都应避免 SWAP。

### CPU

一般来说，Zookeeper 并不会占用大量 CPU 资源。

### 磁盘

磁盘性能对于维护健康的 ZooKeeper 集群至关重要。强烈建议使用固态驱动器 (SSD)，因为 ZooKeeper 必须具有低延迟的磁盘写入才能实现最佳性能。可以通过配置 `autopurge.purgeInterval` 和 `autopurge.snapRetainCount` 来自动清理 ZooKeeper 数据并降低维护开销。

## JVM 优化

Zookeeper 运行在 JVM中，为了避免垃圾回收造成延迟，通常建议设置堆内存为1GB。

## 配置选项

以下罗列了在生产环境中运行 Zookeeper Cluster 需要特别考虑的参数。最理想的情况，dataDir和 dataLogDir 所指定的目录应该是在不同的设备中，这减少了随机读写和顺序写入之间的争执。

- dataDir
    
    用于配置输出 in-memory database 的快照存储位置，一般应该是位于 SSD 上的目录
    
- dataLogDir
    
    用于配置输出transaction log 的存储位置，一般应该是位于 SSD 上的目录
    
- tickTime
    
    是用于计算时间的基本时间单元，最小会话超时将是tickTime的两倍
    
- maxClientCnxns
    
    ZooKeeper 服务器允许的最大客户端连接数。为避免用完允许的连接，将此设置为 0（无限制）。
    
- autopurge.snapRetainCount
    
    启用后，ZooKeeper 自动清除功能分别在 dataDir 和 dataLogDir 中保留 autopurge.snapRetainCount 最近的快照和相应的事务日志，并删除其余的。
    
- autopurge.purgeInterval
    
    触发清除任务的时间间隔（以小时为单位）。设置为正整数以启用自动清除。

<!-- ##{"timestamp":1656604800,"custom_url":"run-zookeeper-in-production-environment"}## -->