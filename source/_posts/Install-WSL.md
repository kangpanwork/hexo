---
title: Install-WSL
date: 2022-03-09 14:28:42
tags: WSL
---



为简单起见，通常建议使用 [`wsl --install`](https://docs.microsoft.com/zh-cn/windows/wsl/install) 安装适用于 Linux 的 Windows 子系统，但如果运行的是旧版 Windows，则可能不支持这种方式。 下面介绍了手动安装步骤。 如果在安装过程中遇到问题，请查看[疑难解答指南的安装部分](https://docs.microsoft.com/zh-cn/windows/wsl/troubleshooting#installation-issues)。

## 步骤 1 - 启用适用于 Linux 的 Windows 子系统

需要先启用“适用于 Linux 的 Windows 子系统”可选功能，然后才能在 Windows 上安装 Linux 分发。

以管理员身份打开 PowerShell（“开始”菜单 >“PowerShell”> 单击右键 >“以管理员身份运行”），然后输入以下命令：

PowerShell复制

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

建议现在转到步骤 #2，更新到 WSL 2，但如果只想安装 WSL 1，现在可以重新启动计算机，然后继续执行[步骤 6 - 安装所选的 Linux 发行版](https://docs.microsoft.com/zh-cn/windows/wsl/install-manual#step-6---install-your-linux-distribution-of-choice)。 若要更新到 WSL 2，请等待重新启动计算机，然后继续执行下一步。

## 步骤 2 - 检查运行 WSL 2 的要求

若要更新到 WSL 2，需要运行 Windows 10。

- 对于 x64 系统：**版本 1903** 或更高版本，采用**内部版本 18362** 或更高版本。
- 对于 ARM64 系统：**版本 2004** 或更高版本，采用**内部版本 19041** 或更高版本。
- 低于 18362 的版本不支持 WSL 2。 使用 [Windows Update 助手](https://www.microsoft.com/software-download/windows10)更新 Windows 版本。

若要检查 Windows 版本及内部版本号，选择 Windows 徽标键 + R，然后键入“winver”，选择“确定”。 更新到“设置”菜单中的[最新 Windows 版本](ms-settings:windowsupdate)。

 备注

如果运行的是 Windows 10 版本1903 或 1909，请在 Windows 菜单中打开“设置”，导航到“更新和安全性”，然后选择“检查更新”。 内部版本号必须是 18362.1049+ 或 18363.1049+，次要内部版本号需要高于 .1049。 阅读详细信息：[WSL 2 即将支持 Windows 10 版本 1903 和 1909](https://devblogs.microsoft.com/commandline/wsl-2-support-is-coming-to-windows-10-versions-1903-and-1909/)。

## 步骤 3 - 启用虚拟机功能

安装 WSL 2 之前，必须启用“虚拟机平台”可选功能。 计算机需要[虚拟化功能](https://docs.microsoft.com/zh-cn/windows/wsl/troubleshooting#error-0x80370102-the-virtual-machine-could-not-be-started-because-a-required-feature-is-not-installed)才能使用此功能。

以管理员身份打开 PowerShell 并运行：

PowerShell复制

```powershell
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

**重新启动**计算机，以完成 WSL 安装并更新到 WSL 2。

## 步骤 4 - 下载 Linux 内核更新包

1. 下载最新包：

   - [适用于 x64 计算机的 WSL2 Linux 内核更新包](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)

    备注

   如果使用的是 ARM64 计算机，请下载 [ARM64 包](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_arm64.msi)。 如果不确定自己计算机的类型，请打开命令提示符或 PowerShell，并输入：`systeminfo | find "System Type"`。 **Caveat：** 在非英文版 Windows 上，你可能必须修改搜索文本，对“System Type”字符串进行翻译。 你可能还需要对引号进行转义来用于 find 命令。 例如，在德语版中使用 `systeminfo | find '"Systemtyp"'`。

2. 运行上一步中下载的更新包。 （双击以运行 - 系统将提示你提供提升的权限，选择“是”以批准此安装。）

安装完成后，请继续执行下一步 - 在安装新的 Linux 分发时，将 WSL 2 设置为默认版本。 （如果希望将新的 Linux 安装设置为 WSL 1，请跳过此步骤。）

 备注

有关详细信息，请参阅 [Windows 命令行博客](https://aka.ms/cliblog)上的文章[对更新 WSL2 Linux 内核的更改](https://devblogs.microsoft.com/commandline/wsl2-will-be-generally-available-in-windows-10-version-2004)。

## 步骤 5 - 将 WSL 2 设置为默认版本

打开 PowerShell，然后在安装新的 Linux 发行版时运行以下命令，将 WSL 2 设置为默认版本：

PowerShell复制

```powershell
wsl --set-default-version 2
```

## 步骤 6 - 安装所选的 Linux 分发

1. 打开 [Microsoft Store](https://aka.ms/wslstore)，并选择你偏好的 Linux 分发版。
