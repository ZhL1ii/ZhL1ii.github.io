---
title: "Omarchy/Arch 安装绿联网卡驱动"
pubDatetime: 2026-10-01T00:00:00+08:00
description: "在 Omarchy/Arch 系 Linux 安装绿联 Wi-Fi 网卡驱动，并解决网卡被识别为 USB 存储设备的问题。"
tags:
  - Linux
  - Tools
---

我在给 Omarchy 安装绿联 AX300 USB Wi-Fi 网卡驱动时遇到了一些比较典型的问题：
- 官方提供的网卡驱动没有 Arch 直接可安装的版本。
- 系统可以识别这个 USB 设备，但它首先会以一个 USB **存储**设备出现，而不是直接作为无线网卡出现。

我用的这款网卡使用的是 AIC8800DC 芯片。最终确认在 Linux 下工作后的设备信息为：

- 芯片：AIC8800DC
- USB VID/PID：a69c:88de
- 我的系统：Omarchy
- 内核：7.2.5-3-omarchy
- 网卡：绿联 AX300 USB Wi-Fi 网卡

## 1. 准备

首先查看当前内核版本：

```
uname -r
```

我的输出为 `7.2.5-3-omarchy`。

然后检查当前安装的内核和 headers：

```
pacman -Q | grep -E '^linux'
```

需要确保安装了与当前内核对应的 headers。Omarchy 使用自己的内核，因此我这里对应的是：

- linux-omarchy
- linux-omarchy-headers

例如我的版本都是 7.2.5-3。

还需要 DKMS 和基本的编译工具：

```
command -v gcc make patch gawk clang
```

如果这些都有输出，就说明编译环境基本齐全。

另外安装：

- dkms
- usb_modeswitch
- tcl

其中 `usb_modeswitch` 用来把网卡从“USB 存储设备模式”强制切换成真正的 Wi-Fi 模式。

如果当前 Omarchy 完全没有网络，可以在另一台联网电脑上下载 Arch Linux 对应的 `.pkg.tar.zst` 软件包，然后通过 U 盘复制过来，再通过 `pacman` 安装。

安装离线软件包：

```
sudo pacman -U dkms-*.pkg.tar.zst
sudo pacman -U tcl-*.pkg.tar.zst usb_modeswitch-*.pkg.tar.zst
```

如果相关包已经安装，就不需要重复安装。

## 2. 下载 AIC8800DC 驱动

我最终使用的是这个项目：

`Kiborgik/aic8800dc-linux-patched`

这个驱动针对 AIC8800DC，并且支持包括 Linux 7.2 在内的较新 Linux 内核。

在另一台联网电脑上下载仓库 ZIP，解压后，将整个目录复制到 U 盘。

例如目录名可能是：

`aic8800dc-linux-patched-main`

然后把 U 盘插入 Omarchy。

## 3. 安装 AIC8800DC 驱动

进入驱动目录，例如：

```sh
cd /run/media/$USER/TRANSFER/aic8800dc-linux-patched-main
```

如果 U 盘使用 ExFAT，脚本的可执行权限可能会丢失，所以先执行：

```sh
chmod +x install.sh test.sh uninstall.sh
```

然后安装：

```sh
sudo ./install.sh
```

这个安装脚本会完成几个操作：

- 使用 DKMS 编译并注册驱动
- 安装 AIC8800DC 对应 firmware
- 安装 USB 模式切换相关的 udev 规则

安装结束后可以运行项目自带的检查脚本：

```sh
sudo ./test.sh
```

还可以检查 DKMS 状态：

```sh
dkms status
```

## 4. 网卡的 U 盘

这款绿联网卡比较特殊。

插入电脑以后，它首先会模拟成一个非常小的 USB 存储设备，里面原本是给 Windows 使用的驱动。

可以使用：

```sh
lsblk -o NAME,TYPE,TRAN,VENDOR,MODEL,SIZE,FSTYPE,MOUNTPOINTS
```

我的机器上可以看到类似：

- 设备类型为 USB
- Vendor 为 AIC
- Model 为 flash
- 容量只有几 MB

例如它可能对应 `/dev/sda`，下面还有 `/dev/sda1`。

这里一定要根据自己的 `lsblk` 输出判断设备，不要直接假设一定是 `/dev/sda`。

## 5. 切换成 Wi-Fi 模式

首先读取这个 USB 存储设备的 VID 和 PID。

假设刚才确定它是 `/dev/sda`：

```sh
VID=$(udevadm info --query=property --name=/dev/sda | sed -n 's/^ID_VENDOR_ID=//p')
PID=$(udevadm info --query=property --name=/dev/sda | sed -n 's/^ID_MODEL_ID=//p')

echo "$VID:$PID"
```

如果文件管理器自动打开了这个“小 U 盘”，先关闭 Nautilus：

```sh
pkill nautilus
```

卸载分区：

```sh
sudo umount /dev/sda1
```

如果仍然因为自动挂载导致 busy，可以使用：

```sh
sudo umount -l /dev/sda1
```

然后使用 `usb_modeswitch` 强制切换设备工作模式：

```sh
sudo usb_modeswitch -v "$VID" -p "$PID" -KQ
```

执行之后，原来的几 MB USB 存储设备应该消失。

这时查看内核日志：

```sh
sudo dmesg | tail -n 50
```

我的机器在成功切换以后能够看到设备重新枚举为：

- Product：AIC8800DC
- Manufacturer：AICSemi
- VID/PID：a69c:88de

看到 `a69c:88de` 基本就说明网卡已经真正进入无线网卡模式。

## 6. 加载驱动

加载 AIC8800DC 驱动：

```sh
sudo modprobe aic8800_fdrv
```

查看模块：

```sh
lsmod | grep aic
```

正常情况下可以看到 `aic8800_fdrv` 和 `aic_load_fw` 等模块。

然后把网卡拔掉，等待两三秒，再重新插入。

等待几秒以后检查 NetworkManager：

```sh
nmcli device status
```

此时应该能够看到一个新的 `wifi` 类型设备，例如 `wlan0` 或 `wl...`。

也可以检查网络接口：

```sh
ip -br link
```

## 7. 连接 Wi-Fi

直接打开 NetworkManager 的终端界面：

```sh
nmtui
```

选择 `Activate a connection`，找到自己的 Wi-Fi，输入密码即可。

也可以继续使用：

```sh
nmcli device status
```

确认无线设备已经变为 connected。

至此，Omarchy 已经可以正常通过这张绿联 AX300 USB 网卡联网。

## 8. 最后

这里推荐使用 DKMS，而不是单纯手工编译一个 `.ko` 文件。

DKMS 会保存驱动源码，并在内核更新之后针对新内核重新构建模块。因此对于 Omarchy / Arch 这类滚动更新系统，这种方式更加合适。

驱动项目同时安装了对应的 firmware 和 udev 规则，所以正常配置完成后，后续启动系统、重新插入网卡时，不应该每次都重新走一遍完整安装流程。

可以通过下面两个命令检查状态：

```sh
dkms status
```

```sh
nmcli device status
```
