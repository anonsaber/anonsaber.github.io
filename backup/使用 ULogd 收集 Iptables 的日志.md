## 背景

在充当 NAT 网关的两个 CentOS 7 服务器上，因为审计需求，需要打开 Iptables 的 Log。本来想着只需要使用`iptables –j LOG` 就可以轻松实现目标，然而由于这两台服务器上的流量是巨大的。使用这种方式直接导致了 CPU 单核被打爆。

于是找到了 [这篇文章](https://indico.cern.ch/event/1002953/contributions/4214017/subcontributions/328091/attachments/2186981/3697683/IRISSecurityWorkshopFeb2021_Centralized_Firewall_Logging_with_ulogd_and_loki.pdf)，它推荐了本文标题中提到的 ULogd。

## 简介

Ulogd 是 [netfilter](http://netfilter.org/) 的一个项目，它是一个运行在用户态的日志记录守护进程，用于 NetFilter/iptables 相关日志记录。

在 Ubuntu 上，你可以轻松的通过 apt 进行安装。而在 CentOS 上，由于官方的 Repo 里面并没有这个东西。所以你需要手动编译安装它，或是使用 [http://repo.iotti.biz/](http://repo.iotti.biz/) 提供的 Repo。

在我的使用场景下，ulogd 能极大的提升 Iptables Log 的性能，几乎没有开销。

## 配置

可惜的是，这个项目的文档写的不是太好，网上也少见现成的使用范例。这里只能介绍一下我本人的使用案例。

### ulogd

*/etc/ulogd.conf 的配置样例*

如果你使用的不是我在本文中提供的 RPM，请不要直接复制粘贴。对于较旧的版本，你可能需要调整 stack 部分为 ULOG 而不是 NFLOG，并且使用 `iptables -j ULOG --ulog-nlgroup <id>` 来记录 Log。

```
[global]
logfile="/var/log/ulogd/ulogd.log"

plugin="/usr/lib64/ulogd/ulogd_inppkt_NFLOG.so"
plugin="/usr/lib64/ulogd/ulogd_filter_IFINDEX.so"
plugin="/usr/lib64/ulogd/ulogd_filter_IP2STR.so"
plugin="/usr/lib64/ulogd/ulogd_filter_PRINTPKT.so"
plugin="/usr/lib64/ulogd/ulogd_output_LOGEMU.so"
plugin="/usr/lib64/ulogd/ulogd_raw2packet_BASE.so"

stack=log1:NFLOG,base1:BASE,ifi1:IFINDEX,ip2str1:IP2STR,print1:PRINTPKT,emu1:LOGEMU

[log1]
group=1

[emu1]
file="/var/log/iptables"
sync=1
```

### iptables

这里 nflog-group 后面的 ID 需要和上面保持一致

```bash
iptables -I POSTROUTING -o <public_nic > -m state --state NEW -j NFLOG --nflog-group 1 --nflog-prefix '[>]NAT '
```

现在你可以考虑使用 Grafana + Loki + Promtail 对日志进行查询、展示。

<!-- ##{"timestamp":1703260800}## -->