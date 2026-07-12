---
title: 正点原子 i.MX6ULL 通过 TFTP + NFS 启动全记录
tags: [嵌入式Linux, i.MX6ULL, NFS, TFTP, U-Boot, 内核编译]
index_img: /img/linux_alpha/alpha-nfs-boot-cover.jpg
date: 2026-07-12 17:00:00
author: xhd
---

# 🚀 正点原子 i.MX6ULL 通过 TFTP + NFS 启动全记录

## 为什么要折腾 NFS 启动？

正点原子 Alpha 开发板默认从 eMMC 或 SD 卡启动。每次改完驱动、调完内核，都要把镜像烧进 SD 卡、拔下来插板子上、上电……循环往复，一天下来 SD 卡槽都快拔烂了。

**TFTP + NFS 启动** 能彻底解决这个问题：

- **内核和 DTB** 通过 TFTP 下载，U-Boot 直接从 PC 拉取
- **根文件系统** 通过 NFS 挂载，板子上跑的文件直接存在 PC 硬盘上
- **改了驱动**，PC 上编译完，板子重启就能跑——不用烧录、不用拔卡

本文记录从零折腾这玩意的全过程，踩过的坑都写下来了。

## 硬件环境

| 项目 | 详情 |
|------|------|
| 开发板 | 正点原子 I.MX6U ALPHA (eMMC 版) |
| 芯片 | NXP i.MX6ULL, Cortex-A7, 792MHz |
| 内存 | 512MB DDR3 |
| 屏幕 | 7 寸 800×480 RGB LCD |
| U-Boot | NXP 2016.03 (`mx6ull_alientek_emmc_defconfig`) |
| PC | Arch Linux x86_64 |

![开发板连线全景](/img/linux_alpha/board_connection1.jpg)
![开发板近景](/img/linux_alpha/board_connection2.jpg)

## PC 侧环境搭建

### 0. 串口终端

开发板通过 **USB-TTL 串口模块** 连接到 PC，一根 Micro USB 线插在板子的 `USB_TTL` 口上，另一头插入 PC 的 USB 口，在 PC 上会识别为一个串口设备（如 `/dev/ttyUSB0`）。

这里用 [picocom](https://github.com/npat-efault/picocom) 作为串口终端——比 minicom 轻量，不需要配置文件，一行命令搞定：

```bash
# 安装
sudo pacman -S picocom

# 连接开发板 (115200 波特率，8N1)
picocom -b 115200 /dev/ttyUSB0
```

退出按 `Ctrl+A` 再按 `Ctrl+X`。

> 如果提示权限不够，把自己加到 `uucp` 组：`sudo usermod -aG uucp $USER`，重新登录后生效。
>
> 串口模块用的是 CH340 芯片，Arch Linux 内核自带驱动，即插即用。

### 1. TFTP 服务器

```bash
# 安装
sudo pacman -S tftp-hpa

# 配置 /etc/conf.d/tftpd
TFTPD_ARGS="--secure /home/xhd/linux/tftpboot/"

# 启动
sudo systemctl start tftpd
```

### 2. NFS 服务器

```bash
# 安装
sudo pacman -S nfs-utils

# /etc/exports
/home/xhd/linux/nfs_rootfs 192.168.1.200/32(rw,sync,no_root_squash,no_subtree_check)

# 生效
sudo exportfs -rv
sudo systemctl start nfs-server
```

### 3. 有线网口固定 IP

板子和 PC 通过 USB 以太网卡直连，需要给 PC 侧固定一个同网段的 IP：

```bash
# 用 NetworkManager 持久化配置
sudo nmcli con add type ethernet \
  ifname enp4s0f3u1u5c2 \
  con-name alpha-board \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.method manual

sudo nmcli con up alpha-board
```

> **坑**：别用 `ip addr add`，那是临时的，NetworkManager 一刷新就没，开发过程中网络反复断。

## 编译内核

正点原子资料里带了内核源码（`linux-imx-4.1.15`），但我们重新编译一遍，确保 NFS Root 支持正确配置。

### 配置

```bash
cd ~/linux/kernel/linux-imx-4.1.15
export PATH=/opt/toolchains/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin:$PATH

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- imx_alientek_emmc_defconfig
```

**必须确认以下选项全部是 `=y`（内置），不能是 `=m`（模块）**：

```
CONFIG_NFS_FS=y              # NFS 客户端
CONFIG_ROOT_NFS=y            # NFS 作为根文件系统
CONFIG_NFS_V3=y              # NFSv3（root mount 用 v3 最稳）
CONFIG_FEC=y                 # FEC 以太网驱动（必须内置！）
CONFIG_IP_PNP=y              # 内核级 IP 配置
```

> **坑 1**：`CONFIG_FEC=m` 是最坑的——模块在挂根文件系统时还没加载，内核找不到网卡，NFS mount 直接跪。
>
> **坑 2**：`CONFIG_NFS_V4` 虽然也开着，但 NFS root 走 v4 需要额外的用户态服务配合，v3 反而最简单可靠。

### 编译踩坑

**坑 3：GCC 10+ 编译旧内核 dtc 报错**

```
multiple definition of `yylloc'
```

原因是 GCC 10+ 默认开启 `-fno-common`，旧内核的 dtc 源码有全局变量重复定义。修复：

```bash
# 编辑 scripts/dtc/Makefile，找到这一行：
HOSTCFLAGS_DTC := -I$(src) -I$(src)/libfdt

# 加上 -fcommon：
HOSTCFLAGS_DTC := -fcommon -I$(src) -I$(src)/libfdt
```

**坑 4：lzop 未安装**

内核默认用 LZO 压缩，但系统没装 lzop。要么装包，要么切 gzip：

```bash
scripts/config --disable KERNEL_LZO --enable KERNEL_GZIP
```

### 编译

```bash
# 编译 zImage
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage -j$(nproc)

# 编译设备树（根据屏幕选）
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- \
  imx6ull-14x14-emmc-7-800x480-c.dtb \
  imx6ull-14x14-emmc-4.3-800x480-c.dtb \
  imx6ull-14x14-emmc-4.3-480x272-c.dtb \
  imx6ull-14x14-emmc-hdmi.dtb

# 安装到 TFTP 目录
cp arch/arm/boot/zImage ~/linux/tftpboot/
cp arch/arm/boot/dts/imx6ull-14x14-emmc-*.dtb ~/linux/tftpboot/
```

## U-Boot 启动配置

开发板上电，按任意键进 U-Boot 命令行：

![U-Boot 启动界面](/img/linux_alpha/uboot_start.png)

```bash
# 网络配置
setenv serverip 192.168.1.100      # PC 的 IP
setenv ipaddr 192.168.1.200        # 开发板的 IP

# 启动参数（一整行，不要换行！）
setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.1.100:/home/xhd/linux/nfs_rootfs,v3,tcp ip=192.168.1.200::192.168.1.1:255.255.255.0::eth0:off'

# 下载并启动
tftp 80800000 zImage
tftp 83000000 imx6ull-14x14-emmc-7-800x480-c.dtb
bootz 80800000 - 83000000
```

![TFTP 下载内核和 DTB](/img/linux_alpha/uboot-tftp.png)

确认启动正常后，固化：

```bash
setenv bootcmd 'tftp 80800000 zImage; tftp 83000000 imx6ull-14x14-emmc-7-800x480-c.dtb; bootz 80800000 - 83000000'
saveenv
```

### bootargs 参数详解

```
ip=192.168.1.200::192.168.1.1:255.255.255.0::eth0:off
```

`ip=` 的 7 个字段（冒号分隔）：

```
板子IP : 服务器IP : 网关 : 掩码 : 主机名 : 网卡 : 是否DHCP
  200   :   (空)   :  1  : 255.0 : (空)  : eth0 :  off
```

> **坑 5**：别用 `ip=dhcp`！板子和 PC 直连的网段没有 DHCP 服务器，内核会一直卡在 "Sending DHCP requests ..."。

> **坑 6**：`nfsroot` 里加 `,v3,tcp` 是关键——尽管内核和 NFS 服务器都支持 v4，但 NFS root over v4 在旧内核上问题很多，强制 v3 省心。

## 启动流程

```
板子上电
  │
  └→ U-Boot (eMMC)
       ├─ tftp 80800000 zImage     ← 从 PC 下载内核 (5.9MB)
       ├─ tftp 83000000 *.dtb       ← 从 PC 下载设备树 (40KB)
       └─ bootz 启动
            │
            ├─ FEC 网卡驱动初始化 (CONFIG_FEC=y, 内置)
            ├─ 配置静态 IP (192.168.1.200)
            ├─ NFS mount 192.168.1.100:/home/xhd/linux/nfs_rootfs
            └─ /sbin/init → 启动完成
```

## 常见问题速查

| 现象 | 原因 | 解决 |
|------|------|------|
| U-Boot ping 不通 PC | PC 有线网口没配 IP 或被 NetworkManager 重置 | `nmcli` 配置静态连接 |
| TFTP 下载 `Abort` / `File not found` | tftpd 服务没启动或目录配错 | `systemctl start tftpd`，检查 `/etc/conf.d/tftpd` |
| 内核卡在 "Sending DHCP requests" | 没有 DHCP 服务器 | `ip=` 改用静态 IP |
| NFS mount 失败 "Unable to mount root fs" | 内核缺 `CONFIG_ROOT_NFS=y` 或 FEC 是模块 | 检查 `.config`，重新编译 |
| NFS 挂载超时 | NFS 版本不匹配 | `nfsroot` 加 `,v3` |

## 目录结构整理

折腾完后的整洁布局：

```
~/linux/
├── kernel/linux-imx-4.1.15/    内核源码 (841MB, 已 make clean)
├── uboot/uboot-imx-2016.03/    U-Boot 源码 (134MB)
├── firmware/OS_Firmware/        mfgtool 烧写固件 (250MB)
├── tftpboot/                   TFTP 目录 (zImage + DTB)
├── nfs_rootfs/                 NFS 根文件系统
└── vendor/                     正点原子原始资料
```

![PC 端目录结构](/img/linux_alpha/dir_layout.png)

## 总结

搞通 TFTP + NFS 启动后，开发效率提升巨大：

- 改完代码 → `make` → 板子 `reset` → 跑起来了，全程在 PC 完成
- 不用反复拔插 SD 卡，不用每次都烧写
- 板子上跑的文件直接存在 PC 的 `nfs_rootfs` 里，想看日志、改配置文件直接 PC 上操作

折腾过程中最核心的三个教训：

1. **FEC 驱动必须内置（`=y`）**，不能是模块
2. **NFS root 用 v3 别用 v4**，旧内核上 v4 坑多
3. **静态 IP 别用 `ip addr add`**，NetworkManager 会抢回去，用 `nmcli` 配置持久连接

希望这篇文章帮你少踩几个坑。欢迎在评论区交流 🚀
