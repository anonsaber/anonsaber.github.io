## 基础

在一个在未经过修改的系统上，内建了以下几张路由表

- **local**: 本地表，处理服务器内部的流量，并包括以太网适配器和广播流量，在大部分情况下不应该修改或是删除这张表。
- **main**: 主路由表，实际上的默认表，`ip route` ****命令在没有指定表的情况下都会对主表进行操作。
- **default**: 这个名字让人觉得这是默认情况下使用的路由表，然而实际上与其名称的逻辑相反，这是一张空表，你可以删除它。
- **unspec**：当所有其他路由路径失败时，内核可以遵循，修改或是删除这张表，它不会显示在 `ip rule show` 里面。

内建的路由表可以使用 `ip rule show` 命令查看

```
0: from all lookup local
32766: from all lookup main
32767: from all lookup default
```

- from all 表示这条规则匹配所有源地址的 IP 包（all 也可以换成具体的 IP 或者 IP 段，类似的条件还有 to all）。
- lookup xxx 表示去查 xxx 这张路由表。

因此，上面的输出翻译成人话就是：

1. 首先查 local 这张路由表，如果查到有一行匹配就按照这一行走；
2. 如果 local 表匹配不上，接着查 main 表；
3. 如果 main 表也匹配不上，就查 default 表；

Linux 的主路由表 (也就是main) 可以使用 `ip route show` 命令查看

```
default via 192.168.100.1 dev eth0 proto dhcp metric 600
169.254.0.0/16 dev eth0 scope link metric 1000 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
192.168.100.0/24 dev eth0 proto kernel scope link src 192.168.100.13 metric 600 
198.18.0.0/15 dev tun0 proto kernel scope link src 198.18.0.1
```

首先看第一列，Linux 会把一个 IP 包的目标地址和路由表第一列的网段进行匹配，如果发现匹配的行就按照那一行的路由规则进行转发，如果找不到匹配的，就按照 default 所在的行进行转发。

第一列后，路由表的每一行都是由一连串的属性组成的：

- via 表示这个 IP 包应该发往的网关地址。
- dev 表示应该这个 IP 包应该从哪张网卡发出。
- metric 表示这一行路由规则的优先级，如果有多行路由都能够转发目标地址，那么系统会按照 metric 数值较小的一行来转发。
- scope link 表示目标地址在链路层可以直达，即不需要通过其他设备进一步转发了。
- proto 记录了这一行路由规则的来源。例如 kernel 说明这一行是内核加的，dhcp 则表明这一行来源于 DHCP。
- linkdown 表示这一行对应的网卡设备处于离线状态。
- src 表示这个 IP 包的源地址最好是 xxx。这个属性只有在一个 IP 包还没有确定源地址（也就是刚刚发出）的时候才有效，如果一个 IP 包已经有了一个源地址，那么这条规则是不会去改的。

## 创建新的路由表

你可以在 `/etc/iproute2/rt_tables` 里面创建一些自定义的路由表，它们在文件中的顺序无关紧要。重要的是表名与编号，每个表在表文件中必须有唯一的名称/编号引用，路由表索引可以随时修改，与 DNS 不同，修改路由表文件不会有缓存问题，更改将立即生效，这是少数几个不需要重新启动服务即可使更改生效的包过滤组件。特别需要注意的是，文件中的编号并不能够代表优先级。

```
# reserved values
#
255     local
254     main
253     default
199     custom
0       unspec
#
# local
#
#1      inr.ruhep
```

比如我们设置 只有当数据包目的地是IP地址 1.1.1.1 时，系统才会根据 custom 表中的路由信息来处理这些数据包，而该规则的优先级为300。

首先我们需要在 custom 路由表里面创建一个去往 1.1.1.1 的路由规则

```bash
ip route add 1.1.1.1/32 via 192.168.0.1 dev eth0 table custom
```

然后我们需要设置去往 1.1.1.1 的流量优先查找 custom 表

```bash
ip rule add to 1.1.1.1 table custom priority 300
```