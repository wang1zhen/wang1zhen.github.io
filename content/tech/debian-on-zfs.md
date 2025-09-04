+++
title = "Debian on zfs 在 zfs 上安装 Debian linux"
author = ["wang1zhen"]
date = 2025-09-05T00:43:00+09:00
draft = false
+++

## 第一部分：准备安装介质 {#第一部分-准备安装介质}


### 下载Debian 13 Live镜像 {#下载debian-13-live镜像}

```sh
# 下载Debian 13 Live镜像（GNOME版本）
wget https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/debian-live-13.0.0-amd64-gnome.iso

# 验证校验和（可选）
wget https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing
```


### 制作启动U盘 {#制作启动u盘}

```sh
# 查看可用设备
lsblk

# 制作启动盘（请替换/dev/sdX为实际设备）
sudo dd if=debian-live-13.0.0-amd64-gnome.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

警告：这将完全擦除目标设备上的所有数据。


## 第二部分：启动并配置网络环境 {#第二部分-启动并配置网络环境}


### 启动到Live环境 {#启动到live环境}

1.  从U盘启动
2.  选择 "Advanced options" -&gt; "Expert install"
3.  或选择 "Rescue mode" 获得完整shell访问权限


### 配置网络连接 {#配置网络连接}

```sh
# 有线网络（通常自动配置）
ip a

# 无线网络配置
iwconfig
wpa_supplicant -B -i wlan0 -c <(wpa_passphrase "SSID" "password")
dhclient wlan0
```


### 设置系统时间 {#设置系统时间}

```sh
timedatectl set-ntp true
```


### 启用SSH（可选） {#启用ssh-可选}

```sh
# 设置root密码
passwd

# 启动SSH服务
systemctl start ssh
ip a    # 查看IP地址
```


## 第三部分：磁盘分区 {#第三部分-磁盘分区}


### 识别目标磁盘 {#识别目标磁盘}

```sh
lsblk
fdisk -l
```

以下示例假设两个目标磁盘为 `/dev/sda` 和 =/dev/sdb=，请根据实际情况调整。


### 创建GPT分区表（两块磁盘） {#创建gpt分区表-两块磁盘}

对每块磁盘执行以下操作：

```sh
# 第一块磁盘
gdisk /dev/sda

# 第二块磁盘
gdisk /dev/sdb
```

在gdisk中对每块磁盘执行以下操作：

1.  输入 `o` 创建新的GPT分区表
2.  创建EFI系统分区：
    -   输入 `n` 创建新分区
    -   分区号：1（默认）
    -   起始扇区：默认
    -   结束扇区：+1G
    -   分区类型：ef00（EFI System Partition）
3.  创建交换分区：
    -   输入 `n` 创建新分区
    -   分区号：2（默认）
    -   起始扇区：默认
    -   结束扇区：+32G
    -   分区类型：8200（Linux swap）
4.  创建ZFS根分区：
    -   输入 `n` 创建新分区
    -   分区号：3（默认）
    -   起始扇区：默认
    -   结束扇区：默认（使用剩余空间）
    -   分区类型：bf00（Solaris Root）
5.  输入 `w` 写入分区表并退出


### 格式化EFI和交换分区 {#格式化efi和交换分区}

```sh
# 格式化EFI分区（两块磁盘）
mkfs.fat -F32 /dev/sda1
mkfs.fat -F32 /dev/sdb1

# 设置交换分区（两块磁盘）
mkswap /dev/sda2
mkswap /dev/sdb2

# 启用交换分区
swapon /dev/sda2
swapon /dev/sdb2
```


### 查看磁盘ID {#查看磁盘id}

```sh
ls -lh /dev/disk/by-id/ | grep -E "(sda|sdb)"
```

记录两块目标磁盘的by-id路径，用于ZFS RAIDZ0配置。


## 第四部分：安装ZFS支持 {#第四部分-安装zfs支持}


### 添加ZFS仓库 {#添加zfs仓库}

```sh
# 添加Debian Backports仓库
echo "deb http://deb.debian.org/debian trixie-backports main contrib non-free non-free-firmware" >> /etc/apt/sources.list

# 更新包列表
apt update
```


### 安装ZFS包 {#安装zfs包}

```sh
# 安装ZFS相关包
apt install -y zfsutils-linux zfs-dkms

# 加载ZFS模块
modprobe zfs

# 验证ZFS模块加载
lsmod | grep zfs
```


## 第五部分：ZFS配置 {#第五部分-zfs配置}


### 创建ZFS存储池（RAID0条带） {#创建zfs存储池-raid0条带}

```sh
zpool create -f \
      -o ashift=12 \
      -o autotrim=on \
      -o compatibility=off \
      -O acltype=posixacl \
      -O compression=zstd \
      -O relatime=on \
      -O xattr=sa \
      -O normalization=formD \
      -O mountpoint=none \
      -O canmount=off \
      -O dnodesize=auto \
      -O sync=standard \
      -O primarycache=all \
      -O secondarycache=all \
      -O recordsize=128K \
      -R /mnt \
      rpool \
      /dev/disk/by-id/ata-DISK1-SERIAL-part3 \
      /dev/disk/by-id/ata-DISK2-SERIAL-part3
```

\*\*重要警告\*\*：这是RAID0条带配置，提供2x容量和更好的性能，但\*\*没有冗余保护\*\*。任何一块磁盘故障都会导致整个池的数据丢失！

注意：请替换磁盘ID为实际的by-id路径。

参数说明：

-   =ashift=12=：针对4K扇区磁盘优化
-   =autotrim=on=：自动启用TRIM（适用于SSD）
-   =compression=zstd=：使用zstd压缩算法
-   条带配置：数据分布在两块磁盘上，提升读写性能


### 创建ZFS数据集 {#创建zfs数据集}

```sh
# 创建根容器数据集
zfs create -o mountpoint=none rpool/ROOT

# 创建系统根数据集
zfs create -o mountpoint=/ -o canmount=noauto rpool/ROOT/debian

# 创建用户相关数据集
zfs create -o mountpoint=/home rpool/home

# 创建系统数据集
zfs create -o mountpoint=/var rpool/var
zfs create -o mountpoint=/var/log rpool/var/log
zfs create -o mountpoint=/var/cache rpool/var/cache

# 创建临时文件数据集 - 性能优化
zfs create \
    -o mountpoint=/tmp \
    -o compression=off \
    -o sync=disabled \
    -o atime=off \
    -o devices=off \
    -o exec=on \
    -o setuid=off \
    rpool/tmp

# 创建SMB共享数据集 - 私有共享
zfs create \
    -o mountpoint=/share \
    -o compression=zstd \
    -o sharesmb="name=share,guestok=false" \
    rpool/share
```


### 验证数据集创建 {#验证数据集创建}

```sh
zfs list -t filesystem
zpool status
```


### 优化ZFS设置 {#优化zfs设置}

```sh
# 设置压缩算法
zfs set compression=zstd rpool/ROOT/debian
zfs set compression=zstd rpool/home
zfs set compression=zstd rpool/var
zfs set compression=zstd rpool/var/log
zfs set compression=zstd rpool/var/cache
zfs set compression=off rpool/tmp

# 优化recordsize设置
zfs set recordsize=128K rpool/ROOT/debian
zfs set recordsize=128K rpool/home

# 设置缓存文件
zpool set cachefile=/etc/zfs/zpool.cache rpool
```


### 重新导入ZFS池 {#重新导入zfs池}

```sh
zpool export rpool
zpool import -d /dev/disk/by-id -R /mnt rpool -N

# 挂载文件系统
zfs mount rpool/ROOT/debian
zfs mount -a

# 设置启动文件系统
zpool set bootfs=rpool/ROOT/debian rpool
```


### 验证挂载 {#验证挂载}

```sh
df -h
mount | grep zfs
```


## 第六部分：安装Debian系统 {#第六部分-安装debian系统}


### 挂载EFI分区 {#挂载efi分区}

```sh
mkdir -p /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi
```


### 安装基础系统 {#安装基础系统}

```sh
# 使用debootstrap安装基础系统
debootstrap --include=openssh-server,vim,curl,wget \
            trixie /mnt http://deb.debian.org/debian/
```


### 配置系统挂载点 {#配置系统挂载点}

```sh
# 生成fstab
cat > /mnt/etc/fstab << EOF
# EFI分区
/dev/sda1 /boot/efi vfat defaults 0 2

# 交换分区
/dev/sda2 none swap sw 0 0
/dev/sdb2 none swap sw 0 0

# tmpfs
tmpfs /tmp tmpfs defaults,nodev,nosuid 0 0
EOF
```


## 第七部分：系统配置 {#第七部分-系统配置}


### 进入chroot环境 {#进入chroot环境}

```sh
# 挂载必要的虚拟文件系统
mount --bind /dev /mnt/dev
mount --bind /dev/pts /mnt/dev/pts
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys

# 进入chroot
chroot /mnt /bin/bash
```


### 配置基本系统信息 {#配置基本系统信息}

```sh
# 设置时区
ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
hwclock --systohc

# 配置语言环境
cat > /etc/locale.gen << EOF
en_US.UTF-8 UTF-8
ja_JP.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
EOF

locale-gen
echo 'LANG=en_US.UTF-8' > /etc/locale.conf

# 设置主机名
echo 'debian-zfs' > /etc/hostname

# 配置hosts文件
cat > /etc/hosts << EOF
127.0.0.1   localhost
::1         localhost
127.0.1.1   debian-zfs.localdomain debian-zfs
EOF
```


### 配置包管理器 {#配置包管理器}

```sh
# 配置APT源
cat > /etc/apt/sources.list << EOF
deb http://deb.debian.org/debian trixie main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian trixie main contrib non-free non-free-firmware

deb http://deb.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian-security trixie-security main contrib non-free non-free-firmware

deb http://deb.debian.org/debian trixie-updates main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian trixie-updates main contrib non-free non-free-firmware

deb http://deb.debian.org/debian trixie-backports main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian trixie-backports main contrib non-free non-free-firmware
EOF

# 更新包列表
apt update
```


### 安装必要软件包 {#安装必要软件包}

```sh
# 安装内核和ZFS支持
apt install -y linux-image-amd64 linux-headers-amd64 \
    zfsutils-linux zfs-dkms \
    grub-efi-amd64 grub-efi-amd64-signed \
    shim-signed efibootmgr \
    firmware-linux firmware-linux-nonfree \
    network-manager systemd-resolved \
    samba samba-common-bin

# 根据CPU类型安装微码
# Intel CPU:
# apt install -y intel-microcode
# AMD CPU:
apt install -y amd64-microcode
```


### 配置ZFS服务 {#配置zfs服务}

```sh
# 生成host ID
zgenhostid

# 确保ZFS缓存目录存在
mkdir -p /etc/zfs
zpool set cachefile=/etc/zfs/zpool.cache rpool

# 启用ZFS服务
systemctl enable zfs-import-cache.service
systemctl enable zfs-mount.service
systemctl enable zfs.target
```


### 配置initramfs {#配置initramfs}

```sh
# 编辑initramfs配置
cat >> /etc/initramfs-tools/modules << EOF
zfs
EOF

# 更新initramfs
update-initramfs -c -k all
```


### 安装和配置GRUB {#安装和配置grub}

```sh
# 安装GRUB到EFI分区
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=debian

# 配置GRUB
cat > /etc/default/grub << EOF
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="Debian"
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
GRUB_CMDLINE_LINUX="root=ZFS=rpool/ROOT/debian"
GRUB_PRELOAD_MODULES="zfs"
EOF

# 生成GRUB配置
update-grub
```


### 设置root密码 {#设置root密码}

```sh
passwd root
```


### 创建普通用户 {#创建普通用户}

```sh
useradd -m -s /bin/bash username  # 请替换username为实际用户名
passwd username
usermod -aG sudo username
```


### 配置SMB共享 {#配置smb共享}

```sh
# 设置/share目录权限（仅uid=1000用户可访问）
chown 1000:1000 /share
chmod 700 /share

# 为用户设置SMB密码（请替换username为实际用户名）
smbpasswd -a username

# 启用SMB服务
systemctl enable smbd
systemctl enable nmbd

echo "SMB共享配置完成：\\\\server\\share"
```


### 启用基本服务 {#启用基本服务}

```sh
systemctl enable NetworkManager
systemctl enable systemd-resolved
systemctl enable ssh
```


## 第八部分：ZFS定期维护配置 {#第八部分-zfs定期维护配置}


### 配置ZFS定期维护服务 {#配置zfs定期维护服务}

```sh
# 创建ZFS trim服务
cat > /etc/systemd/system/zfs-trim@.service << 'EOF'
[Unit]
Description=zpool trim on %i
Documentation=man:zpool-trim(8)
Requires=zfs.target
After=zfs.target
ConditionACPower=true
ConditionPathIsDirectory=/sys/module/zfs

[Service]
Nice=19
IOSchedulingClass=idle
KillSignal=SIGINT
ExecStart=/bin/sh -c '\
if /usr/sbin/zpool status %i | grep "trimming"; then\
exec /usr/sbin/zpool wait -t trim %i;\
else exec /usr/sbin/zpool trim -w %i; fi'
ExecStop=-/bin/sh -c '/usr/sbin/zpool trim -s %i 2>/dev/null || true'

[Install]
WantedBy=multi-user.target
EOF

# 创建ZFS trim定时器（每月执行）
cat > /etc/systemd/system/zfs-trim@.timer << 'EOF'
[Unit]
Description=Monthly zpool trim on %i

[Timer]
OnCalendar=monthly
AccuracySec=1h
Persistent=true

[Install]
WantedBy=multi-user.target
EOF

# 启用内置的scrub定时器和trim定时器
systemctl enable zfs-scrub-weekly@rpool.timer
systemctl enable zfs-trim@rpool.timer

echo "ZFS定期维护服务配置完成"
```


## 第九部分：完成安装 {#第九部分-完成安装}


### 退出chroot并清理 {#退出chroot并清理}

```sh
# 退出chroot环境
exit

# 卸载虚拟文件系统
umount /mnt/dev/pts
umount /mnt/dev
umount /mnt/proc
umount /mnt/sys

# 卸载EFI分区
umount /mnt/boot/efi

# 卸载ZFS文件系统
zfs umount -a
zpool export rpool
```


### 重启系统 {#重启系统}

```sh
reboot
```


## 第十部分：首次启动后配置 {#第十部分-首次启动后配置}


### 验证系统状态 {#验证系统状态}

```sh
# 检查ZFS状态
sudo zpool status
sudo zfs list

# 检查挂载点
df -h
mount | grep zfs

# 检查RAID0条带状态
zpool status -v

# 检查服务状态
systemctl status zfs.target
systemctl status NetworkManager
```


### 启动ZFS定期维护服务 {#启动zfs定期维护服务}

```sh
# 启动定时器
sudo systemctl start zfs-scrub-weekly@rpool.timer
sudo systemctl start zfs-trim@rpool.timer

# 验证定时器状态
systemctl status zfs-scrub-weekly@rpool.timer
systemctl status zfs-trim@rpool.timer

# 查看下次执行时间
systemctl list-timers | grep zfs
```


### 验证SMB共享 {#验证smb共享}

```sh
# 检查SMB服务状态
sudo systemctl status smbd
sudo systemctl status nmbd

# 验证ZFS SMB共享配置
zfs get sharesmb rpool/share

# 查看共享列表
smbclient -L localhost -U username

# 测试共享访问
ls -la /share

echo "SMB共享验证完成"
```


### 系统更新 {#系统更新}

```sh
sudo apt update && sudo apt upgrade -y
```


### 安装桌面环境（可选） {#安装桌面环境-可选}

```sh
# GNOME桌面环境
sudo apt install -y task-gnome-desktop

# 或者KDE Plasma
# sudo apt install -y task-kde-desktop

# 启用图形登录管理器
sudo systemctl enable gdm3
```


## 第十一部分：ZFS维护管理 {#第十一部分-zfs维护管理}


### 常用ZFS命令 {#常用zfs命令}

```sh
# 查看存储池状态
zpool status

# 查看数据集
zfs list

# 手动创建快照
sudo zfs snapshot rpool/ROOT/debian@manual-$(date +%Y%m%d)

# 列出快照
zfs list -t snapshot

# 删除快照
sudo zfs destroy rpool/ROOT/debian@manual-20240101
```


### RAID0条带管理 {#raid0条带管理}

```sh
# 查看条带状态
zpool status -v rpool

# 查看条带性能统计
zpool iostat -v rpool

# 检查磁盘健康状态（RAID0中任何磁盘故障都是致命的）
sudo smartctl -a /dev/sda
sudo smartctl -a /dev/sdb
```

\*\*重要提醒\*\*：RAID0条带配置下：

-   任何一块磁盘故障都会导致整个池不可用
-   无法离线单个磁盘进行维护
-   磁盘故障时只能从备份恢复整个系统
-   建议定期备份重要数据到外部存储


### 性能监控 {#性能监控}

```sh
# ZFS性能统计
zpool iostat 1

# 实时监控ZFS I/O
zpool iostat -v 1

# 数据集使用情况
zfs get used,available,referenced,compressratio
```


## 第十二部分：故障排除 {#第十二部分-故障排除}


### 常见问题 {#常见问题}

**问题1：启动时找不到ZFS池**

```sh
# 手动导入池
sudo zpool import -f rpool

# 重新生成缓存文件
sudo zpool set cachefile=/etc/zfs/zpool.cache rpool
```

**问题2：磁盘故障处理**

```sh
# 检查池状态
sudo zpool status -v

# 查看详细错误信息
sudo journalctl -u zfs-import-cache.service -n 50

# 检查磁盘健康状态
sudo smartctl -a /dev/sda
sudo smartctl -a /dev/sdb
```


### 紧急恢复 {#紧急恢复}

如果系统无法启动，可以使用Debian Live ISO：

1.  启动到Live环境
2.  安装ZFS支持：=apt update &amp;&amp; apt install -y zfsutils-linux=
3.  导入ZFS池：=zpool import -R /mnt rpool=
4.  挂载文件系统：=zfs mount rpool/ROOT/debian &amp;&amp; zfs mount -a=
5.  挂载EFI分区：=mount /dev/sda1 /mnt/boot/efi=
6.  进入chroot：=chroot /mnt=
7.  执行修复操作


## 常用命令速查 {#常用命令速查}
