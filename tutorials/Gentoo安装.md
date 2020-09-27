# Gentoo AMD64 安装

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

刻录光盘

启动安装媒介

[Arch官方GNU Screen帮助界面](https://wiki.archlinux.org/index.php/GNU_Screen_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

配置网络

> ping -c 3 www.gentoo.org

确保网络可用

### 准备磁盘

| 分区 | 大小             | 文件系统     | Name   | Flags     |
| ---- | ---------------- | :----------- | ------ | --------- |
| 1    | 2M               | (bootloader) | grub   | bios_grub |
| 2    | 256M             | fat32        | boot   | boot      |
| 3    | 1024M            | (swap)       | swap   |           |
| 4    | Rest of the disk | ext4         | rootfs |           |

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
mkfs.fat -F 32 /dev/sda2
mkswap /dev/sda3
swapon /dev/sda3
mkfs.ext4 /dev/sda4
mount /dev/sda4 /mnt/gentoo
```



| 分区 | 大小             | 文件系统     | Name   | Flags     |
| ---- | ---------------- | :----------- | ------ | --------- |
| 1    | 2M               | (bootloader) | grub   | bios_grub |
| 2    | 256M             | fat32        | boot   | boot      |
| 3    | 1024M            | (swap)       | swap   |           |
| 4    | Rest of the disk | ext4         | rootfs |           |

### 安装stage3

设置正确的日期时间

> ntpd -q -g
>
> date

下载stage3

```bash
cd /mnt/gentoo
#选择镜像站进入/releases/amd64/autobuilds/current-stage3-amd64/ 下载
links https://www.gentoo.org/downloads/mirrors/
# 校验
openssl dgst -r -sha512 stage3-amd64-<release>.tar.?(bz2|xz) > sha512.hash
openssl dgst -r -whirlpool stage3-amd64-<release>.tar.?(bz2|xz) > whirlpool.hash
# to be continued
gpg --verify stage3-amd64-<release>.tar.?(bz2|xz){.DIGESTS.asc,} 
tar xpvf stage3-*.tar.bz2 --xattrs-include='*.*' --numeric-owner
```

配置/mnt/gentoo/etc/portage/make.conf

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

配置/mnt/gentoo/etc/portage/repos.conf/gentoo.conf

```bash
[gentoo]
location = /usr/portage
sync-type = rsync
sync-uri = rsync://mirrors.tuna.tsinghua.edu.cn/gentoo-portage/
#sync-uri = rsync://rsync.mirrors.ustc.edu.cn/gentoo-portage/
auto-sync = yes
```



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

# 若要暂停安装
exit
cd
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
reboot # poweroff

# 若要继续安装
mount /dev/sda4 /mnt/gentoo
然后继续复制DNS信息等步骤
```

## 配置基本系统

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

## 配置bootloader