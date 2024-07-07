# CentOS 7-8-9

CentOS 7 与 CentOS 8/8-Stream 已经 EOL，然而作为 Red Hat Enterprise Linux 的免费社区版，一直以来在服务器操作系统领域占据着重要地位。本文将介绍如何将 CentOS 7 升级到 CentOS 9，如果你公司的 CTO 并没有计划更换到其他的 Linux 发行版，本文可能对你有所帮助。

**开始操作前，请确保已经完成重要数据的异地备份。**

**本文中的 mirror.rackspace.com 可以替换为 mirrors.tuna.tsinghua.edu.cn 或是其他的镜像源**

## 更新 CentOS 7

在开始升级到 CentOS 9 之前，我们需要确保当前的 CentOS 7 系统已经更新到最新版本。

1. 启用 CentOS 的归档源

   由于默认的 mirrorlist 已经停止解析，所以我们需要手动启用 CentOS 的归档源。

   ```bash
   sed -e 's|^mirrorlist=|#mirrorlist=|g' \
       -e 's|^#baseurl=http://mirror.centos.org/centos|baseurl=https://mirror.rackspace.com/centos-vault/centos|g' \
       -i /etc/yum.repos.d/*.repo
   ```
2. 清理 YUM 缓存并重新生成缓存

   ```bash
   yum clean all
   yum makecache 
   ```
3. 安装 yum-utils 工具包来增强 YUM 的功能

   ```bash
   yum install yum-utils -y
   ```
4. 安装 EPEL（Extra Packages for Enterprise Linux）仓库，以便能够访问更多的软件包

   ```bash
   yum install https://mirror.rackspace.com/epel/epel-release-latest-7.noarch.rpm -y
   ```
5. 修改 EPEL 仓库配置文件，以确保使用正确的镜像源

   ```bash
   sed -e 's|^metalink=|#metalink=|g' \
       -e 's|^#baseurl=|baseurl=|g' \
       -e 's|https\?://download\.fedoraproject\.org/pub/epel|https://mirror.rackspace.com/epel|g' \
       -e 's|https\?://download\.example/pub/epel|https://mirror.rackspace.com/epel|g' \
       -i /etc/yum.repos.d/epel{,-testing}.repo
   ```
6. 重新生成 YUM 缓存并进行系统升级

   ```bash
   yum makecache
   yum upgrade -y
   ```

## 准备系统环境

在更新完CentOS 7之后，我们安装一些必要工具，为升级到 CentOS 9 做准备。

1. 安装 rpmconf 工具，用于处理配置文件

   ```bash
   yum install rpmconf -y
   ```
2. 运行 rpmconf，检查并合并所有配置文件

   **注意：** 在执行此命令时，系统会询问你是否要合并不同版本的配置文件,你需要根据实际情况来做出决定。

   ```bash
   rpmconf -a
   ```
3. 使用 package-cleanup 工具清理系统

   **注意：** 在执行 package-cleanup --leaves 和 package-cleanup --orphans 命令时，系统会列出一些叶子包（无其他包依赖的包）和孤立包（不再被任何包依赖的包）。你需要根据具体情况来决定是否要移除这些包：
   * 如果这些包对你的应用和服务没有影响，可以选择删除它们，以减少系统的冗余。
   * 如果不确定某些包的作用，建议保留它们，以免影响系统运行。

   ```bash
   package-cleanup --leaves
   package-cleanup --orphans
   ```
4. 切换到 DNF 并移除 YUM

   YUM 是 CentOS 7 使用的包管理器，而在 CentOS 8 及以后的版本中，DNF 取代了 YUM。所以我们需要安装 DNF 并移除 YUM。

   ```bash
   yum install dnf -y
   ```
5. 移除 YUM 及其相关组件，并删除 YUM 配置文件夹

   ```bash
   dnf -y remove yum yum-metadata-parser && rm -rf /etc/yum
   ```
6. 使用 DNF 升级现有的软件包

   ```bash
   dnf upgrade -y
   ```
7. 重新启动服务器

   ```bash
   reboot
   ```

## 升级到 CentOS 8-Stream

1. 安装 CentOS 8 核心包

   ```bash
   dnf install https://mirror.rackspace.com/centos-vault/8.5.2111/BaseOS/x86_64/os/Packages/{centos-linux-repos-8-3.el8.noarch.rpm,centos-linux-release-8.5-1.2111.el8.noarch.rpm,centos-gpg-keys-8-3.el8.noarch.rpm} -y
   ```

2. 修改镜像源配置文件

   ```bash
   sed -e 's|^mirrorlist=|#mirrorlist=|g' \
       -e 's|^#baseurl=http://mirror.centos.org/$contentdir|baseurl=baseurl=https://mirror.rackspace.com/centos-vault/centos|g' \
       -i /etc/yum.repos.d/CentOS-*.repo
   ```

3. 移除 EPEL 7 并安装 EPEL 8

   ```bash
   dnf remove epel-release
   dnf install https://mirror.rackspace.com/epel/epel-release-latest-8.noarch.rpm -y
   ```

4. 修改 EPEL 镜像源配置文件

   ```bash
   sed -e 's|^metalink=|#metalink=|g' \
       -e 's|^#baseurl=https\?://download.fedoraproject.org/pub/epel/|baseurl=https://mirror.rackspace.com/epel/|g' \
       -e 's|^#baseurl=https\?://download.example/pub/epel/|baseurl=https://mirror.rackspace.com/epel/|g' \
       -i /etc/yum.repos.d/epel*.repo
   ```

 5. 再次运行 rpmconf 以处理配置文件

    ```bash
    rpmconf -a
    ```

 6. 删除旧的内核包

    ```bash
    rpm -e $(rpm -q kernel)
    ```

 7. 删除备份的镜像源配置文件

    ```bash
    rm -rf /etc/yum.repos.d/CentOS-*.repo.rpmsave
    ```

 8. 清理 DNF 缓存

    ```bash
    dnf clean all
    ```

 9. 移除不兼容的软件包

    ```bash
    dnf remove dracut-network sysvinit-tools dracut-network
    ```

10. 同步发行版软件包

    注意: 这里将会将当前系统的包升级到 CentOS 8.5.2111

    ```bash
    dnf -y --releasever=8.5.2111 --allowerasing --setopt=deltarpm=false distro-sync
    ```

11. 安装内核包

    ```bash
    dnf -y --releasever=8.5.2111 install kernel-core
    ```

12. 更新核心和最小安装组

    ```bash
    dnf -y --releasever=8.5.2111 groupupdate "Core" "Minimal Install"
    ```

13. 移除旧版本的 rpmconf

    ```bash
    dnf --releasever=8.5.2111 remove python36-rpmconf
    ```

14. 安装最新版本的 rpmconf 和 yum-utils

    ```bash
    dnf --releasever=8.5.2111 install rpmconf yum-utils
    ```

15. 再次运行 rpmconf 处理配置文件

    ```bash
    rpmconf -a 
    ```

16. 移除不需要的软件包和额外的软件包

    注意：这些软件包理论上都应该移除掉，但是请务必逐个确认是否已经有备份，以及后续是否有替代的工具。

    ```bash
    dnf --releasever=8.5.2111 repoquery --unneeded
    dnf --releasever=8.5.2111 remove -y $(dnf --releasever=8.5.2111 repoquery --unneeded)
    dnf --releasever=8.5.2111 repoquery --extras
    dnf --releasever=8.5.2111 remove -y $(dnf --releasever=8.5.2111 repoquery --extras)
    ```

17. 自动移除未使用的软件包

    ```bash
    dnf --releasever=8.5.2111 autoremove -y
    ```

18. 使用 package-cleanup 工具进一步清理系统

    ```bash
    package-cleanup --leaves
    package-cleanup --orphans
    ```

20. **列出仍然安装的 CentOS 7 软件包**

    注意：这些软件包理论上都应该移除掉，但是请务必逐个确认是否已经有备份，以及后续是否有替代的工具。

    ```bash
    dnf --releasever=8.5.2111 list --installed | grep el7
    ```

21. 刷新并更新软件包：

    ```bash
    dnf --releasever=8.5.2111 upgrade --refresh -y
    ```

22. 安装 CentOS Stream 发布包

    ```bash
    dnf --releasever=8.5.2111 install centos-release-stream
    ```

23. 修改 CentOS Stream 镜像源配置文件

    ```bash
    sed -e 's|^mirrorlist=|#mirrorlist=|g' \
        -e 's|^#baseurl=http://mirror.centos.org/$contentdir|baseurl=https://mirror.rackspace.com/centos-vault/centos|g' \
        -i /etc/yum.repos.d/CentOS-Stream-*.repo
    ```

24. 切换到 CentOS Stream 仓库

    ```bash
    dnf --releasever=8.5.2111 swap centos-{linux,stream}-repos
    ```

25. 再次运行 rpmconf 处理配置文件

    ```bash
    rpmconf -a
    ```

26. 删除旧的 CentOS Linux 镜像源备份

    ```bash
    rm -rf /etc/yum.repos.d/CentOS-Linux-*.repo.rpmsave
    ```

27. 再一次修改 CentOS Stream 镜像源配置文件

    ```bash
    sed -e 's|^mirrorlist=|#mirrorlist=|g' \
        -e 's|^#baseurl=http://mirror.centos.org/$contentdir|baseurl=https://mirror.rackspace.com/centos-vault/centos|g' \
        -i /etc/yum.repos.d/CentOS-Stream-*.repo
    ```

28. 再次刷新并更新软件包

    注意: 这里将会将当前系统的包升级到 CentOS 8-Stream

    ```bash
    dnf upgrade --refresh -y
    ```

29. 同步发行版软件包

    ```bash
    dnf -y distro-sync
    ```

30. 重新启动服务器

    ```bash
    reboot
    ```

## 升级到 CentOS 9 Stream


1. 禁用特定模块

   ```bash
   dnf module disable python36 virt
   ```
2. 替换镜像源

   视网络情况的可选步骤，建议参考 <https://mirrors.tuna.tsinghua.edu.cn/help/centos-stream/> 进行。
3. 安装 CentOS Stream 9 核心包

   ```bash
   dnf install https://mirror.rackspace.com/centos-stream/9-stream/BaseOS/x86_64/os/Packages/{centos-stream-repos-9.0-26.el9.noarch.rpm,centos-stream-release-9.0-26.el9.noarch.rpm,centos-gpg-keys-9.0-26.el9.noarch.rpm}
   ```
4. 再次运行 rpmconf 处理配置文件

   ```bash
   rpmconf -a
   ```
5. 移除旧的 EPEL 包

   ```bash
   dnf remove epel-release
   ```
6. 删除备份的镜像源配置文件

   ```bash
   rm -rf /etc/yum.repos.d/*.repo.rpmsave
   ```
7. 安装 EPEL 9 和 EPEL Next 9

   ```bash
   dnf install https://mirror.rackspace.com/epel/{epel-release-latest-9.noarch.rpm,epel-next-release-latest-9.noarch.rpm}
   ```


 8. 清理 DNF 缓存并重新生成缓存

    ```bash
    dnf clean all
    dnf makecache
    ```
 9. 删除旧的内核包

    ```bash
    rpm -e `rpm -q kernel`
    ```
10. 同步发行版软件包

    ```bash
    dnf -y --releasever=9 --allowerasing --setopt=deltarpm=false distro-sync
    ```
11. 再次清理 DNF 缓存

    ```bash
    dnf clean all
    ```
12. 强制重启服务器

    ```bash
    systemctl --force --force reboot
    ```
13. 重置特定模块

    实际情况可能不同，因此需要重置的模块也可能有所不同。请根据具体情况进行调整。

    ```bash
    dnf module reset perl perl-IO-Socket-SSL perl-libwww-perl satellite-5-client mysql
    ```
14. 重建 RPM 数据库

    ```bash
    rm -f /var/lib/rpm/__db*
    rpm --rebuilddb
    ```
15. 更新核心和最小安装组：

    ```bash
    dnf -y groupupdate "Core" "Minimal Install"
    ```
