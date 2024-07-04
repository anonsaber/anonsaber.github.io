*本文的方案是为标准代理服务器 HTTP(S) 与 Socks 而设计的。*

## 背景

由于 ChromeOS 的生态链一直算不上完整，尽管同时拥有 ChromeOS Android Linux 三个平台的软件资源，但是就实际使用而言，并不能做到很好的互通有无，比如如何在 ChromeOS 上不使用开发者模式实现全局使用代理服务器，就是个比较麻烦的事情。为此要在网络层面下功夫，实现设备级的全局代理。

我们先看一下现状，看一下现在大家所采用的主流的代理服务器技术。因为在计算机行业，弄出来个新东西的根本思路就是把当前的技术东拼西凑，哪怕创造一门新的语言也是如此，所以接下来我们要怎么解决问题也会是如此。所以笔者比较赞成计算机科学是不存在的，只有计算机工程，你当然可以有你的理解，这不是本文要讨论的东西。言归正传，我们讲回正题。

标准的代理服务器常用的有两种，HTTP(S) 与 Socks。此外，基于 Iptables 实现的 Redirect 虽然实现了和代理服务器同样的效果，但是并不能称之为代理服务器，在这种现状下，我们这里同样将它也视为一种方案。

## 方案设计

现在，我们提出解决问题的思路:

1. 如果希望一个设备在接入网络后不需要做任何配置就能够实现全局代理（对该设备而言也是透明代理）的话，我们需要实现当设备访问到路由器时能够在路由器内部实现分流。
2. 我们需要在路由器里面设计 Iptables Redirect 到 HTTP(S)/Socks/Redir，或是使用路由表将流量引入到 HTTP(S)/Socks/Redir 代理所在的服务器。

到这里，可能 Iptables Redirect 是最受广大软路由玩家青睐的，这个做法我觉得是可行的，搭配一些脚本实现健康检查也不是非常困难的事情，它应该能工作的很好。

然而我并不喜欢使用 ipset 这样的方式去处理流量，因为我算不上是个软路由玩家，相比起来软路由，我更喜欢商业路由器，不过本文的方案依旧能够在 OPNSense 或是 OpenWRT 上工作。

如果我们放弃了使用 Iptables Redirect 的方案，那么就产生了下面两个问题:

1. 如何将路由表导入到路由器内部，进而将流量路由到特定的网络位置？
2. 如何将三层协议路由过去的流量导入到三层协议以上的 HTTP(S)/Socks/Redir 代理所在的服务器？

对于第一个问题，这个问题的解决办法有很多，我们暂时不做讨论。

再看第二个问题，对于 HTTP(S) 代理服务器，笔者找到了 tun2proxy，对于 Socks 代理服务器，tun2socks 就很好用，而对于 Redir，很抱歉，笔者没有什么建议，因此才有了本文的第一句话「本文的方案是为标准代理服务器 HTTP(S) 与 Socks 而设计的」。顺便一提，由于 Socks 代理比 HTTP(S) 有更好的性能，笔者推荐使用 Socks + tun2socks。而 tun2socks 与 tun2proxy，简单来说就是能够实现将进入到 tun 虚拟网卡的流量转发到 Socks/HTTP(S) 代理服务器。

接着，我们可以利用一种办法从路由器按照特定的路由表将流量引流到 tun 虚拟网卡上（tun 虚拟网卡的位置可以是在 HTTP(S)/Socks/Redir 代理所在的服务器，也可以是另外一台机器，笔者推荐将 tun 虚拟网卡启动在 HTTP(S)/Socks/Redir 代理所在的服务器上面）以实现设备的全局透明代理。

现在回看第一个问题，尽管第一个问题的解决方案有很多，但是如何较为轻松的维护这个路由表实际上还是值得讨论的。笔者这里使用了 [[idndx.com](https://idndx.com/use-routeros-ospf-and-raspberry-pi-to-create-split-routing-for-different-ip-ranges/)](https://idndx.com/use-routeros-ospf-and-raspberry-pi-to-create-split-routing-for-different-ip-ranges/) 提供的解决方案。该作者制作了非大陆 IP 的路由表，能够搭配 OSPF 实现动态的更新路由器内部的路由表。

## 实现过程

我们假设，你已经有了一个 socks 代理服务器，以及已经有了一个无污染的 DNS，随便他们部署在什么位置，只要他们是可用的。

后续的演示仅针对 IPv4，对于 Ipv6 同样可以照葫芦画瓢实现，对于防火墙放行规则，请自行查阅文档处理。

### Tun2socks

开启服务器的路由转发

```bash
sysctl -w net.ipv4.ip_forward=1
```

创建 TUN 虚拟网卡

```bash
ip tuntap add mode tun dev tun0
ip addr add 198.18.0.1/15 dev tun0
ip link set dev tun0 up
```

开启 tun 虚拟网卡到 socks5 的转发

```bash
tun2socks -device tun0 -proxy socks5://<your_server>:<your_port> -interface eth0
```

我们不需要按照 Tun2socks 的 Wiki 为 tun 虚拟网卡开启任何的路由规则，因为这些路由规则将有 Bird 来做。

### Bird

安装并启动 bird，略。

### 生成使用 tun 网卡的路由表规则

你可以使用 nchnroutes 来生成这张表，要记得注意修改出接口的名字，并且使用 exclude 选项排除一些地址，比如代理服务器的地址，请根据你的实际情况来落实。

/etc/bird/routes4.conf

```
...
route 1.1.1.1/32 via "tun0";
...
```

### 配置 OSPF 将路由表广播给路由器

/etc/bird/bird.conf

```
# This is a minimal configuration file, which allows the bird daemon to start
# but will not cause anything else to happen.
#
# Please refer to the documentation in the bird-doc package or BIRD User's
# Guide on <http://bird.network.cz/> for more information on configuring BIRD and
# adding routing protocols.

# Change this into your BIRD router ID. It's a world-wide unique identification
# of your router, usually one of router's IPv4 addresses.
router id 10.255.255.2;

# The Kernel protocol is not a real routing protocol. Instead of communicating
# with other routers in the network, it performs synchronization of BIRD's
# routing tables with the OS kernel.
protocol kernel {
	scan time 60;
	import none;
	export all;   # Actually insert routes into the kernel routing table
}

# The Device protocol is not a real routing protocol. It doesn't generate any
# routes and it only serves as a module for getting information about network
# interfaces from the kernel.
protocol device {
	scan time 60;
}

protocol static {
	include "routes4.conf";
}

protocol ospf {
	export all;
	area 0.0.0.0 {
		interface "eth1" {
			authentication simple;
			password "yourpassword";
		};
	};
}
```

- export all: 将路由插入到内核路由表
- router id: OSPF 实例的唯一标识符，一般填写本设备的 IP 地址。
- interface: 使用哪张网卡进行 OSPF 路由广播。
- authentication: 有多种选项，simple 是指明文传输，其他的请自己查阅文档
- password: 顾名思义

### Reload Bird 使配置生效

```bash
birdc configure
```

你可以使用 `curl --interface tun0 <https://1.1.1.1>` 测试你的 tun 虚拟网卡是否工作（访问的地址需要在上面的路由表内）。

### 配置路由器

在路由器上配置 OSFP 接受来自 Bird 的路由广播，请参考你的路由器使用手册，如果一切正常，你能够在你的路由器里面看到接收到的来自 Bird 的路由表。

## 宕机测试

关闭 bird 服务或是关闭 tun 网卡与 bird 所在服务器后，路由器中的路由表会消失。

## 注意事项

sometimes we need to disable rp_filter for the corresponding interface so that it can receive packets from other interfaces.

```bash
sysctl net.ipv4.conf.all.rp_filter=0
sysctl net.ipv4.conf.eth0.rp_filter=0
```