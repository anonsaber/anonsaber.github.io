## 前言

为了提高虚拟机的性能，或是扩展其使用场景。有时候，我们需要让虚拟机直接访问到硬件资源。

开启 PCI Passthrough 需要硬件支持。

## IOMMU

### IOMMU Enable

### 查看当前主机是否支持

```bash
dmesg | grep -e DMAR -e IOMMU

```

如果没有输出，则表示不支持。

### 修改内核启动参数

修改 /etc/default/grub

- Intel CPU

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"

```

- AMD CPU

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on"

```

### 更新 grub

```bash
Proxmox:  update-grub
OpenSUSE: grub2-mkconfig -o /etc/grub2-efi.cfg
CentOS:   grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg

```

### 重新启动主机

检查 grub 是否更新成功

```bash
cat /proc/cmdline | grep iommu

```

### 检查 IOMMU 是否开启

```bash
dmesg | grep "Virtualization Technology for Directed I/O"

```

或

```bash
dmesg | grep -e DMAR -e IOMMU | grep enable

```

<!-- more -->

### PT Mode

Intel 与 AMD 芯片组均能够通过添加 "iommu=pt" 参数开启 PT 模式，以 Intel 为例:

按照如下修改 /etc/default/grub 并更新 grub

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"

```

PT 模式只为直通模式中使用的设备启用 IOMMU，这将防止 Linux 试图接触 (touching) 无法直通的设备。

## Interrupt Remapping

**要使用 PCI passthrough，Interrupt Remapping 是必须的。**

Interrupt Remapping 能够工作在较新的设备上，你可以使用下面的命令查看当前的主机是否支持 Interrupt Remapping:

```bash
dmesg | grep 'remapping'

```

如果你看到了下面任何一行

```bash
"AMD-Vi: Interrupt remapping enabled"
"DMAR-IR: Enabled IRQ remapping in x2apic mode" ('x2apic' can be different on old CPUs, but should still work)

```

那么当前的主机是能够支持 Interrupt Remapping 的。

然而，如果当前的主机不支持 interrupt remapping，你可以使用如下命令开启 `unsafe interrupt remapping`:

```bash
echo "options vfio_iommu_type1 allow_unsafe_interrupts=1" > /etc/modprobe.d/iommu_unsafe_interrupts.conf

```

完成上述修改后，你需要更新 initramfs 并重新启动系统

```bash
update-initramfs -u -k all

```

另外，你也可以尝试配置 Interrupt Remapping 的内核引导参数

```bash
intremap={on,off,nosid,no_x2apic_optout}

```

## 验证 IOMMU 分组

```bash
find /sys/kernel/iommu_groups/ -type l

```

## VFIO

### Required Moudules

修改 /etc/modules，开启以下 Kernel Moudules

```bash
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd

```

### 为 PCI 设备开启 VFIO

以 Intel 核心显卡为例:

### 查询 PCI 设备的 ID

```bash
lspci -nn| grep 'Intel Corporation' | grep HD

00:02.0 VGA compatible controller [0300]: Intel Corporation HD Graphics 630 [8086:5912] (rev 04)
00:1f.3 Audio device [0403]: Intel Corporation 100 Series/C230 Series Chipset Family HD Audio Controller [8086:a170] (rev 31)

```

### 绑定为 VFIO 设备

修改 /etc/modprobe.d/vfio.conf

```bash
options vfio-pci ids=8086:5912,8086:a170 disable_vga=1

```

在物理机禁用 Intel 核心显卡和音频，创建并修改 /etc/modprobe.d/custom-blacklist.conf

```bash
blacklist snd_hda_intel
blacklist snd_hda_codec_hdmi
blacklist i915
```

完成上述修改后，你需要更新 initramfs 并重新启动系统

```bash
update-initramfs -u -k all

```

重新启动系统后，确认 Intel 核心显卡使用的 driver 是不是 vfio-pci

```bash
lspci -nnk| grep -C1 vfio
```

现在你可以使用 virt-manager 或 Proxmox 在虚拟机上分配 PCI 设备了。

## 其他的内核启动参数

这里列举可能会用到的部分内核启动参数

- 尽可能多的将 iommu groups 拆分，方便直通一些板载的设备

```bash
pcie_acs_override=downstream,multifunction
```

- 禁用显卡的 frame buffering

```bash
video=vesafb:off
video=efifb:off
video=simplefb:off
initcall_blacklist=sysfb_init
```

- 禁用显卡的 vesa driver

```bash
video=vesa:off
```

<!-- ##{"timestamp":1653235200}## -->