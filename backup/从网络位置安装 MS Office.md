可不可以在局域网内使用 Samba (共享文件夹) 来安装 Office 2016/2019 呢？

当然是可以的。

可不可以在安装 Office 2016/2019 时选择选择自己所需要的功能呢？

当然也是可以的。

## 创建描述文件

这是一份将会只生成 Word，Excel，PowerPoint，以及 Visio 的描述文件。

```xml
<Configuration>
  <Add OfficeClientEdition="64" Channel="PerpetualVL2019" AllowCdnFallback="True">
    <Product ID="ProPlus2019Volume">
      <Language ID="zh-cn" />
      <ExcludeApp ID="Access" />
      <ExcludeApp ID="Groove" />
      <ExcludeApp ID="Lync" />
      <ExcludeApp ID="OneDrive" />
      <ExcludeApp ID="OneNote" />
      <ExcludeApp ID="Outlook" />
      <ExcludeApp ID="Publisher" />
    </Product>
    <Product ID="VisioPro2019Volume">
      <Language ID="zh-cn" />
      <ExcludeApp ID="Groove" />
      <ExcludeApp ID="OneDrive" />
    </Product>
  </Add>
</Configuration>

```

描述文件也可以使用 Office Tool Plus 完成创建。

下载地址: https://otp.landian.vip/zh-cn/

## 下载安装文件

下载 Office Deployment Tool

https://www.microsoft.com/en-us/download/details.aspx?id=49117

安装后，能够看到一个 setup.exe 和若干个 xml 文件，你可以删掉那几个 xml 文件，也可以置之不理。

将我们的描述文件 configuration.xml 放到 setup.exe 所在的目录，并使用 CMD 执行如下命令

```
setup.exe /download configuration.xml
```

一切顺利的话，你会看到一个名为 Office 的文件夹。

## 创建共享文件夹

使用 Windows 搭建一个共享文件夹，当然你也可以使用 Linux 搭建一个 Samba 服务器，然后将 setup.exe configuration.xml 以及 Office 放到这个共享文件夹中。

确保你能够在其他计算机上使用类似 `\\\\192.168.10.254\\OFFICE2019\\` 的方式访问到安装文件。

## 创建一个安装脚本

- Network-INSTALL-OFFICE.bat

```
@echo off

setlocal

::Store working directory to return after finished
set WORKDIR=%CD%

::Map and switch to a network drive and give it unmapped drive letter
pushd \\\\192.168.10.254\\OFFICE2019\\

::Store the name of the network drive so it can be unmapped when finished
Set NETDRIVE=%CD%
set NETDRIVE=%NETDRIVE:\\=%

::Change back to the original directory
cd /d %WORKDIR%

::Run commands with the network drive mapped
%NETDRIVE%\\setup.exe /configure

::Uncomment this to catch errors from the executable
::if %errorlevel% GTR o exit %errorlevel%

::Unmap the network drive
net use %NETDRIVE% /delete /y

endlocal
```

## 执行安装

只需要把安装脚本 Network-INSTALL-OFFICE.bat 拷贝到目标计算机上，然后双击执行即可安装。

根据本文的实际测试，千兆有线网络下，在 PD 虚拟机内安装完 Word, Excel, PowePoint 以及 Visio 总计耗时不到 5 分钟。