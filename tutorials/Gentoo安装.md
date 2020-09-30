# Gentoo AMD64 安装

[^2020/9/30]: 最后更新时间

[TOC]

本安装只适用于amd64架构 UEFI固件 的安装

选择方面：

- GPT分区
- GRUB2 bootloader
- stage3-amd-64  (openrc) (multilib) (32位和64位)

选择GPT分区 GRUB2 bootloader

参考资料：

[gentoo官方安装教程](https://wiki.gentoo.org/wiki/Handbook:AMD64)

[YangMame安装教程](https://blog.yangmame.org/Gentoo%E5%AE%89%E8%A3%85%E6%95%99%E7%A8%8B.html)

## 准备环境

### 准备安装媒介

在[镜像站](https://www.gentoo.org/downloads/mirrors/)下载Gentoo最小化安装CD,路径为

> releases/amd64/autobuilds/current-install-amd64-minimal/install-amd64-minimal-<release>.iso

校验文件

```bash
# 获取公钥
gpg --keyserver hkps://hkps.pool.sks-keyservers.net --recv-keys 0xBB572E0E2D182910
# 将输出的密钥指纹与 https://www.gentoo.org/downloads/signatures/ 比对
gpg --verify stage3-amd64-<release>.tar.?(bz2|xz){.DIGESTS.asc,}
# 校验hash
grep -A 1 -i sha512 stage3-amd64-<release>.tar.?(bz2|xz){.DIGESTS.asc,} | sha512sum -c
```

刻录光盘

启动安装媒介

[Arch官方GNU Screen帮助界面](https://wiki.archlinux.org/index.php/GNU_Screen_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

配置网络

> ping -c 3 www.gentoo.org

确保网络可用

### 准备磁盘

参考分区

| 分区 | 大小             | 文件系统     | Name   | Flags     |
| ---- | ---------------- | :----------- | ------ | --------- |
| 1    | 2M               | (bootloader) | grub   | bios_grub |
| 2    | 256M             | fat32        | boot   | boot      |
| 3    | 1024M            | (swap)       | swap   |           |
| 4    | Rest of the disk | ext4         | rootfs |           |

设置分区

```bash
parted -a optimal /dev/sda
> print
> mklabel gpt #所有数据将消失
> unit mib
> mkpart primary 1 3
> name 1 grub
> set 1 bios_grub on
> mkpart primary 3 259
> name 2 boot
> mkpart primary 259 1283
> name 3 swap
> mkpart primary 1283 -1
> name 4 rootfs
> set 2 boot on
> print
> quit
```

挂载文件系统

```bash
mkfs.fat -F 32 /dev/sda2
mkswap /dev/sda3
swapon /dev/sda3
mkfs.ext4 /dev/sda4
mount /dev/sda4 /mnt/gentoo
```

至此，分区信息应该完全吻合下表

| 分区 | 大小             | 文件系统     | Name   | Flags     |
| ---- | ---------------- | :----------- | ------ | --------- |
| 1    | 2M               | (bootloader) | grub   | bios_grub |
| 2    | 256M             | fat32        | boot   | boot      |
| 3    | 1024M            | (swap)       | swap   |           |
| 4    | Rest of the disk | ext4         | rootfs |           |

### 安装stage3

设置正确的日期时间

```bash
ntpd -q -g
date
```

下载stage3

```bash
cd /mnt/gentoo
#选择镜像站进入/releases/amd64/autobuilds/current-stage3-amd64/ 下载
links https://www.gentoo.org/downloads/mirrors/
# 校验 参照上文
# 解压
tar xpvf stage3-*.tar.bz2 --xattrs-include='*.*' --numeric-owner
```

## 配置基本系统

### 配置portage

/mnt/gentoo/etc/portage/make.conf 文件应该增加的内容

```bash
# GCC
COMMON_FLAGS="-march=native -O2 -pipe"
# CPU_FLAGS_X86="" 稍后使用用cpuid2cpuflags输出值
MAKEOPTS="-j5"

# USE
USE="-bindst"

# Portage
GENTOO_MIRRORS="https://mirrors.tuna.tsinghua.edu.cn/gentoo/"
EMERGE_DEFAULT_OPTS="--ask --verbose=y --keep-going"
# FEATURES="${FEATURES} -userpriv -usersandbox -sandbox"
ACCEPT_LICENSE="*"

# Language
L10N="en-US zh-CN en zh"
LINGUAS="en_US zh_CN en zh"

# Else
VIDEO_CARDS="intel i965 nvidia"

GRUB_PLATFORMS="efi-64"
```

创建 /mnt/gentoo/etc/portage/repos.conf/gentoo.conf 文件并写入以下内容

```bash
[gentoo]
location = /usr/portage
sync-type = rsync
sync-uri = rsync://mirrors.tuna.tsinghua.edu.cn/gentoo-portage/
auto-sync = yes
```

### chroot



若chroot之后要关机，暂停安装，输入：

```bash
exit
cd
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
# 无法卸载时，输入以下命令查看信息
fuser -mv /mnt
poweroff
```

若要继续安装，引导进入LiveCD之后挂载，然后进行正常chroot步骤

```bash
mount /dev/sda4 /mnt/gentoo
```

正常chroot步骤

```bash
# 复制DNS信息
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
# 挂载必要文件系统
mount -t proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
# chroot
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
# 挂载boot分区(不一定是/dev/sda2)
mount /dev/sda2 /boot
```



至此chroot完毕，可以中断安装并稍后恢复到安装中以弹性确定安装时间

```bash
emerge-webrsync
eselect news list
eselect news read
emerge vim
emerge cpuid2cpuflags
cpuid2cpuflags #将输出值改入CPU_FLAGS_X86
eselect profile list
# 选择一个profile
emerge -auvDN --with-bdeps=y @world
```

配置时区地区

```bash
echo "Asia/Shanghai" > /etc/timezone
emerge --config sys-libs/timezone-data

echo "en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8" >> /etc/locale.gen

locale-gen

eselect locale list
```

配置/etc/fstab,UUID通过blkid查看

```bash
UUID=x    /boot    vfat    noatime    0 2
UUID=x    none    swap    sw   	 0 0
UUID=x    /    ext4    noatime    0 1
UUID=x  /mnt/cdrom   auto    noauto,user          0 0
```



## 配置内核

```bash
emerge -av gentoo-sources
emerge -av genkernel
genkernel --menuconfig all
genkernel --install initramfs
```



## 配置bootloader

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Gentoo
grub-mkconfig -o /boot/grub/grub.cfg
```

## 收尾工作

```bash
useradd -m -G users,wheel,portage,usb,video 这里换成你的用户名(小写)
passwd 用户名
```

