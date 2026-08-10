+++
date = '2025-10-16T19:36:00+08:00'
draft = false
title = 'Archlinux安装流程'
+++

## 准备安装介质

这部分内容通常需要在另外一台具有操作系统的电脑上操作。

下载archlinux live cd，在[官方下载网站](https://archlinux.org/download/)上找到对应区域的镜像，下载所需架构的iso文件。

准备一块不小于8GB的U盘，在Windows上使用[rufus](https://rufus.ie/zh/)将iso写入U盘，制作启动器。在这一部分你必须选择写入的iso适配BIOS或UEFI，大多数新机型都支持UEFI，但最好先进行一些了解，因为两种情况的安装方式并不相同。

将启动器插入待安装系统的物理机，从U盘启动。如果是预装系统，可能需要了解如何进入启动界面。

## 安装流程

> **WARNING**
>
> 以下内容并不适用于所有用户或潜在用户，请自行辨别哪些东西对自己有用。

### 在VirtualBox上安装

#### 硬件准备

新建虚拟机。4G内存，单核，启用UEFI，显卡选择VMSVGA(通常默认)，网络模式NAT(通常默认)。CD选择下载的iso。

#### 网络

进入Live环境，如果使用NAT，通常已有网络连接。使用`ip-link`查看详细的网络情况：

```bash
ip link
```

如果检出`eth`或`enp`接口有`UP`字样，说明该接口启用并有物理连接。这通常不能用来判断该主机网络联通性，需通过`ping`来确认：

```bash
ping bilibili.com
```

#### 系统时间

根据archwiki，在Live环境中systemd-timesyncd默认启用。欲手动检查时间同步，使用`timedatectl`：

```bash
timedatectl
```

这一步骤中，通常显示的是UTC标准时间，而非UTC+8时间（北京时间）。

#### 磁盘分区

使用`lsblk`列出可用磁盘

```bash
lsblk
```

在虚拟机上，通常这会列出`sda`或`sdb`，对应该虚拟机的硬盘。在真机上，这可能会列出形如`nvme0n1`的设备，注意鉴别你想使用的磁盘。


下面假定你使用的磁盘是`disk`，使用`cfdisk`来对硬盘分区：

```bash
cfdisk /dev/disk
```

对于UEFI，通常选用gpt分区表进行分区，规划如下：

> **NOTICE**
>
> 交换分区是可选的。TODO

| 分区 | 记号 | 分区类型 | 大小 |
| --- | --- | --- | --- |
| EFI系统分区 | \[efi\] | EFI System | 1G |
| 交换分区 | \[swap\] | Linux swap | 至少4G |
| 根分区 | \[root\] | Linux root (x86-64) 或 Linux root (x64) | 设备剩余空间，至少23-32G |

按照引导，将更改写入磁盘。

如果你使用了不同的规划，下面的操作需要你记得这些规划。你仍然可以通过`lsblk`列出所有分区表，或通过`fdisk -l`查看详细的分区表，并再次进入`cfdisk`更改你的选择。

#### 格式化


这里使用ext4文件系统，对你的根分区`/dev/[root]`格式化如下：

```bash
mkfs.ext4 /dev/[root]
```

对你的EFI系统`/dev/[efi]`格式化如下：

```bash
mkfs.fat -F 32 /dev/[efi]
```

对你的交换分区`/dev/[swap]`（如果有）：

```bash
mkswap /dev/[swap]
```

#### 挂载

```bash
mount /dev/[root] /mnt
mkdir -p /mnt/boot
mount /dev/[efi] /mnt/boot
swapon /dev/[swap]
```

#### 安装必要软件包

```bash
pacstrap -K /mnt base linux linux-firmware
```

#### 生成fstab

生成 fstab 文件以使需要的文件系统（如启动目录 /boot）在启动时被自动挂载，用 -U 或 -L 选项分别设置 UUID 或卷标：

```bash
genfstab -U /mnt > /mnt/etc/fstab
```

#### chroot到新系统

根据archwiki，这一部分使用`arch-chroot`而不是`chroot`：

```bash
arch-chroot /mnt
```

#### 设置时间和时区

```bash
ln -sf /usr/share/zoneinfo/地区名/城市名 /etc/localtime
```

例如，在中国大陆需要将时区设置为北京时间，那么请运行

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

时区名称是上海而非北京，是因为上海是该时区内人口最多的城市。

然后运行`hwclock`以生成 /etc/adjtime：

```bash
hwclock --systohc
```

这个命令假定已设置硬件时间为 UTC 时间。为了防止时钟漂移并确保时间准确，可以手动启用 NTP（网络时间协议，Network Time Protocol）客户端（例如 systemd-timesyncd）设置时间同步。

#### 区域和本地化设置

编辑 /etc/locale.gen，然后取消掉 en_US.UTF-8 UTF-8 和其他需要的 UTF-8 区域设置前的注释（#）。

接着执行`locale-gen`以生成 locale 信息：

```bash
locale-gen
```

然后创建 locale.conf文件，并编辑设定 LANG 变量，比如：

```bash
# /etc/locale.conf
LANG=en_US.UTF-8
```

#### 主机名

创建hostname文件：

```bash
# /etc/hostname
主机名
```

#### 设置root密码

```bash
passwd
```

### 安装启动引导器

#### GRUB

在本节的内容里，把 esp 替换成ESP分区挂载点，本教程中esp为`/boot`。

首先安装软件包 `grub`和 `efibootmgr`。其中“GRUB”是启动引导器，“efibootmgr”被 GRUB 脚本用来将启动项写入 NVRAM。

使用下面命令将GRUB EFI 应用 grubx64.efi 安装到 esp/EFI/GRUB/，并将其模块安装到 /boot/grub/x86_64-efi/。

```bash
grub-install --target=x86_64-efi --efi-directory=esp --bootloader-id=GRUB
```

然后为`grub`生成主配置，使用`grub-mkconfig`：

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

#### Systemd Init

TODO

### 重启电脑

使用`exit`退出chroot环境，通过`reboot`重启系统。你将看到tty登录界面。

## 参考
- [archwiki cn](https://wiki.archlinuxcn.org/wiki/首页)
