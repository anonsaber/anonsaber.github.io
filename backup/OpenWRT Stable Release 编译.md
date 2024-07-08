## 前言

我是不太喜欢 OpenWRT 的，相比来说 RouterOS 是我更中意的。不过为了实现 CPE，还希望功能能够丰富一些，还想要网口多一些，还想要成本低一些，看起来也只能选择一些开发板，所以在前段时间我买了一个 MT7621 的开发板。

然后插上 WIFI (MT7612EN + MT7922) 和 4G (移远 EC20) 模块后，发现这些模块并不能正常工作，虽然他们确实显示在 lsusb 中了。

简单的搜了一些文档，发现需要 kmod，然后去装 kmod 的时候发现装不上。完球了，我猜我需要编译 OpenWRT 了。

于是和商家要到了个开发资料，商家很爽快的就给了。解压之后，我先照着商家发的资料和文档照葫芦画瓢的编译了一个固件出来，并且很快的刷入了设备，发现问题并不像我之前想的那么简单...

## 调查

因为我一直不怎么关注 OpenWRT，我此时并不知道应该怎么处理下面这些在我脑子里浮现出来的问题：

- 我需要在编译的时候就将 kmod 一起编译出来？
- 如果我的问题没有得到解决，或是我更换了其他的硬件，需要更多的 kmod，我就可能需要不断的编译更多的东西？
- 需要强制安装 kmod？
- 官方的 opkg Repo，难道不能使用在自编译的固件上？

这是当时我脑子里意识到的东西（虽然有些现在来看并不是那么准确）

在我整理了思路后，我认为官方的固件也必然是个 CI/CD 来完成编译的，所以一定有办法在自编译固件上使用官方的 opkg Repo。至于强制安装或许能解决当下的问题，但是肯定是下下策。

基于上面的判断，经过了一些搜索后我从下面的两个文章里面弄清了原因。

https://hamy.io/post/0015/how-to-compile-openwrt-and-still-use-the-official-repository/

https://libremesh.org/development-kernel_vermagic.html

你可以直接去看这两个文章，或是你可以留在这里继续看我的总结。

## 原因

自编译固件的 kernel 版本与官方 Repo 里面的 kmod 安装包中记录的 kernel 版本不匹配，准确的说是 vermagic。

> This is because OpenWRT has a system called “vermagic” that ensures you can only install kmod packages with the same settings as the build config settings of the kernel you are running.
> 

OpenWRT 通过一个名为 vermagic 的记录来确保你只能安装具有同样设置的 kmod 安装包。

你可以用下面的命令查看你当前运行的固件的 vermagic

```bash
opkg list-installed kernel
```

## 重新编译

现在，我们需要编译一个与官方 Stable Release 具有同样 vermagic 的自编译版本，并确保在刷入设备后能够从官方的 opkg Repo 安装 kmod。

### 查看官方固件的 vermagic

官方固件的 vermagic 从下载地址即可以看到，比如下面这个。

https://downloads.openwrt.org/releases/23.05.2/targets/ramips/mt7621/kmods/

里面能够看到这样一个目录: 5.15.137-1-9c242f353867f49a96054ff8c9f2c460，其中 9c242f353867f49a96054ff8c9f2c460 就是官方固件的 vermagic

### 编译特定 vermagic 的固件

### 检出代码

由于上面我们查看的是 Stable Release 23.05.2 的 vermagic，所以我们需要检出 Stable Release 23.05.2 的 Tag。

```bash
git clone https://git.openwrt.org/openwrt/openwrt.git
cd openwrt
git checkout v23.05.2
```

### 更新并安装 feeds

```bash
# 更新库
./scripts/feeds update -a

# 下载软件包
./scripts/feeds install -a
```

### 调整代码

<aside>
💡 如果你需要修改 DTS 请在更新并安装 feeds 后开始。比如我的开发板硬件与 mqmaker WiTi Board 很像，所以我需要将此设备的 DTS 按照商家所提供的 DTS 进行修改。在编译时即可以将机型选择为 mqmaker WiTi Board。而如果你的设备无需修改 DTS，它是官方固件下载地址里面有的，那你又何必如此折腾的编译固件呢?

</aside>

### 设置 .config

获取 Stable Release 23.05.2 官方固件的 config.buildinfo

```bash
wget -O .config https://downloads.openwrt.org/releases/23.05.2/targets/ramips/mt7621/config.buildinfo
```

补全 .config

上面的 config 是一份精简的配置，我们首先需要对其进行补全。

```bash
make defconfig
```

### 检查 vermagic

编译完整的固件会花费非常久的时间，你肯定不想完成构建后发现 vermagic 依旧是不匹配的。所以在编译完整固件之前，我们可以通过下面的命令来以仅编译内核并检查 vermagic。

```bash
make target/linux/{clean,compile}
```

<aside>
⚠️ 在 `make target/linux/{clean,compile}` 执行之后，最终它很可能会失败，并且你此时无法构建完整的内核（因为你尚未首先构建必要的先决条件）。

但不要害怕！我们所需要的 .vermagic 文件应该在失败之前就会生成，这就是我们现阶段所需要的。

</aside>

```bash
find build_dir/ -name .vermagic -exec cat {} \;
```

确认你看到的 vermagic 与前面步骤中得到的官方固件的 vermagic 是一致的。

### 做适当的调整

在这一步，你可以做一些自定义的改动。但是请注意，不恰当的改动会影响 vermagic，**我的建议是这里只做机型选择，别的不要动**。

或是如果你能忍耐漫长的编译时间，可以跳过这一步骤，它将编译所有的机型。

```bash
make menuconfig
```

### 再次检查 vermagic

如果你执行过 `make menuconfig`，我们需要重新确认 vermagic 的

```bash
make target/linux/{clean,compile}
```

当然，像前面的步骤一样，这个编译依旧很可能不会成功。

```bash
find build_dir/ -name .vermagic -exec cat {} \;
```

如果你看到的 vermagic 值依旧是官方固件的 vermagic 是一致的。那么我们可以进行下面的步骤了。如果不是，你需要重新执行上面的几个步骤。或是考虑跳过 `make menuconfig`。

### 下载编译所需要的资源

```bash
make download
```

### 执行编译

```bash
make V=s
```

这一步会花费很长时间。编译完成后，你将得到一个包含了基础软件的固件。

在我的例子当中，它会存放在 `bin/targets/ramips/mt7621/openwrt-23.05.2-ramips-mt7621-mqmaker_witi-squashfs-sysupgrade.bin`。

一切正常的话，它已经能够刷入并完成启动，你可以直接使用官方 opkg Repo 镜像 kmod 或是其他 opkg 包的安装。

## 添加更多的包

这是一个可选步骤。

<aside>
💡 个人认为，如果是自用的情况，没有必要再做这些步骤，因为现在我们已经能够从官方 opkg Repo 镜像执行 opkg 的安装。

</aside>

如果想要在固件中引入更多特定的包（以实现在恢复出厂设置时不会丢失他们），除了可以在编译的时候使用 `make menuconfig` 进行添加（这可能需要反复调整以保证 vermagic 的一致性，**使用 Image Builder（这个工具也会在上一步的编译过程中一起获得。） 来进行后续的处理是更好的选择。**

### 解压 Image Builder

```bash
cp bin/targets/ramips/mt7621/openwrt-imagebuilder-23.05.2-ramips-mt7621.Linux-x86_64.tar.xz /home/ares

tar -xf /home/ares/openwrt-imagebuilder-23.05.2-ramips-mt7621.Linux-x86_64.tar.xz
```

### 复制 Base-files

将上一步编译出来的 `base-files_*_mipsel_24kc.ipk` (位于 bin/targets/ramips/mt7621/packages) 拷贝到 openwrt-imagebuilder-23.05.2-ramips-mt7621.Linux-x86_64/packages。

*这是针对我们的硬件的 OpenWRT 框架文件，除了这个其他的包我们都可以使用 Image Builder 从官方下载到的 opkg file。*

```bash
cp bin/targets/ramips/mt7621/base-files_*_mipsel_24kc.ipk /home/ares/openwrt-imagebuilder-23.05.2-ramips-mt7621.Linux-x86_64/packages
```

### 生成新的固件

```bash
cd /home/ares/imagebuilder-23.05.2-ramips-mt7621.Linux-x86_64

make image \\
PROFILE=mqmaker_witi \\
PACKAGES="pkg1 pkg2 pkg3 -pkg4 -pkg5 -pkg6" \\
FILES="files" \\
DISABLED_SERVICES="svc1 svc2 svc3"
```

<!-- ##{"timestamp":1700236800}## -->