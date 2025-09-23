+++
title = "Debian on zfs 在 zfs 上安装 Debian linux"
author = ["wang1zhen"]
description = "分步骤在 ZFS 阵列上安装 Debian 13 并配置系统。"
date = 2025-09-15T00:00:00+09:00
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

以下示例假设两个目标磁盘为 `/dev/nvme0n1` 和 =/dev/nvme1n1=，请根据实际情况调整。


### 设置磁盘变量 {#设置磁盘变量}

```sh
# 设置磁盘设备变量（使用by-id路径，请根据实际情况调整）
DISK1=/dev/disk/by-id/nvme-SAMSUNG_SSD_980_1TB_S649NJ0R123456A
DISK2=/dev/disk/by-id/nvme-SAMSUNG_SSD_980_1TB_S649NJ0R123456B

echo "DISK1: $DISK1"
echo "DISK2: $DISK2"
```


### 创建GPT分区表（两块磁盘） {#创建gpt分区表-两块磁盘}

\*\*第一块磁盘\*\*（包含启动分区）：

```sh
# 清除现有分区表并创建新的GPT分区表
sgdisk --zap-all $DISK1

# 创建EFI系统分区 (1G)
sgdisk -n 1:1M:+1G -t 1:EF00 -c 1:"EFI System Partition" $DISK1

# 创建交换分区 (16G)
sgdisk -n 2:0:+16G -t 2:8200 -c 2:"Linux swap" $DISK1

# 创建ZFS根分区（剩余空间）
sgdisk -n 3:0:0 -t 3:BF00 -c 3:"ZFS Root" $DISK1

# 显示分区信息
sgdisk -p $DISK1
```

\*\*第二块磁盘\*\*（仅ZFS存储）：

```sh
# 清除现有分区表并创建新的GPT分区表
sgdisk --zap-all $DISK2

# 创建ZFS根分区（全部空间）
sgdisk -n 1:1M:0 -t 1:BF00 -c 1:"ZFS Root" $DISK2

# 显示分区信息
sgdisk -p $DISK2
```


### 格式化EFI和交换分区 {#格式化efi和交换分区}

```sh
# 格式化EFI分区
mkfs.fat -F32 ${DISK1}-part1

# 设置交换分区
mkswap ${DISK1}-part2

# 启用交换分区
swapon ${DISK1}-part2
```


### 查看磁盘ID {#查看磁盘id}

```sh
ls -lh /dev/disk/by-id/ | grep -E "(nvme0n1|nvme1n1)"
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
# 安装必要工具和依赖
apt install -y linux-headers-$(uname -r) build-essential \
    debootstrap arch-install-scripts gdisk

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
      ${DISK1}-part3 \
      ${DISK2}-part1
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
zfs create -o mountpoint=/ -o canmount=noauto -o compression=zstd -o recordsize=128K rpool/ROOT/debian

# 创建用户相关数据集
zfs create -o mountpoint=/home -o compression=zstd -o recordsize=128K rpool/home

# 创建系统数据集
zfs create -o mountpoint=/var -o compression=zstd rpool/var
zfs create -o mountpoint=/var/log -o compression=zstd rpool/var/log
zfs create -o mountpoint=/var/cache -o compression=zstd rpool/var/cache

# 创建 Podman 数据集（rootful）- 小文件优化
zfs create \
    -o mountpoint=/var/lib/containers \
    -o compression=zstd \
    -o recordsize=64K \
    -o atime=off \
    rpool/containers

# 创建 Podman 数据集（rootless，可选）
# 设置变量：新系统的主要用户名（请替换为实际用户名）
user_debian="YOUR_USERNAME"

zfs create \
    -o mountpoint="/home/${user_debian}/.local/share/containers" \
    -o compression=zstd \
    -o recordsize=64K \
    -o atime=off \
    rpool/containers-${user_debian}

# 创建用户缓存数据集 - 缓存优化，性能导向
zfs create \
    -o mountpoint="/home/${user_debian}/.cache" \
    -o compression=zstd \
    -o recordsize=64K \
    -o atime=off \
    -o sync=disabled \
    rpool/cache-${user_debian}

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

# 创建SMB共享数据集 - 私有共享，性能优化（Samba 另行配置）
zfs create \
    -o mountpoint=/share \
    -o compression=zstd \
    -o recordsize=1M \
    -o atime=off \
    rpool/share
```


### 验证数据集创建 {#验证数据集创建}

```sh
zfs list -t filesystem
zpool status
```


### 设置ZFS缓存文件 {#设置zfs缓存文件}

```sh
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


### 挂载EFI分区作为/boot {#挂载efi分区作为-boot}

```sh
mkdir -p /mnt/boot
mount ${DISK1}-part1 /mnt/boot
```


### 安装基础系统 {#安装基础系统}

```sh
# 使用debootstrap安装基础系统
debootstrap --include=openssh-server,vim,curl,wget,locales \
            trixie /mnt http://deb.debian.org/debian/
```


### 配置系统挂载点 {#配置系统挂载点}

```sh
# 使用genfstab自动生成fstab（ZFS数据集会被自动处理）
genfstab -U /mnt >> /mnt/etc/fstab

# 检查生成的fstab
cat /mnt/etc/fstab

# 注意：ZFS数据集（包括/tmp）不需要在fstab中配置，ZFS会自动管理
```


## 第七部分：系统配置 {#第七部分-系统配置}


### 进入chroot环境 {#进入chroot环境}

```sh
# 使用arch-chroot（自动处理虚拟文件系统挂载）
arch-chroot /mnt
```


### 配置基本系统信息 {#配置基本系统信息}

```sh
# 设置时区
ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

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
    zfsutils-linux zfs-dkms zfs-initramfs \
    grub-efi-amd64 grub-efi-amd64-signed \
    shim-signed efibootmgr \
    firmware-linux firmware-linux-nonfree firmware-iwlwifi firmware-sof-signed \
    network-manager systemd-resolved \
    samba samba-common-bin

# 根据CPU类型安装微码
# Intel CPU:
# apt install -y intel-microcode
# AMD CPU:
apt install -y amd64-microcode
```


### 安装桌面环境（可选） {#安装桌面环境-可选}

```sh
# GNOME桌面环境
apt install -y task-gnome-desktop

# 或者KDE Plasma
# apt install -y task-kde-desktop
```


### 配置ZFS服务 {#配置zfs服务}

```sh
# 生成host ID（强制覆盖）
zgenhostid $(hostid) -f

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


### 配置GRUB {#配置grub}

```sh
# 配置GRUB
cat > /etc/default/grub << EOF
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=\`( . /etc/os-release && echo \${NAME} )\`
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
GRUB_CMDLINE_LINUX="root=ZFS=rpool/ROOT/debian"
EOF

# 生成GRUB配置
update-grub

# 安装GRUB到EFI分区（/boot就是EFI分区）
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=debian
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


### 设置用户缓存目录权限 {#设置用户缓存目录权限}

```sh
# 设置用户缓存目录权限（请替换username为实际用户名）
chown 1000:1000 "/home/username/.cache"
chmod 755 "/home/username/.cache"
```


### 配置SMB共享 {#配置smb共享}

```sh
# 设置/share目录权限（仅uid=1000用户可访问）
chown 1000:1000 /share
chmod 700 /share

# 配置Samba配置文件
cat > /etc/samba/smb.conf << 'EOF'
[global]
    workgroup = WORKGROUP
    security = user
    map to guest = never
    server string = Debian ZFS Server

[share]
    path = /share
    guest ok = no
    read only = no
    valid users = username
    comment = Private ZFS Share
    create mask = 0660
    directory mask = 0770
EOF

# 为用户设置SMB密码（请替换username为实际用户名）
smbpasswd -a username

# 验证Samba配置
testparm -s

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
# 退出chroot环境（按Ctrl+D或输入exit）
exit

# arch-chroot会自动清理虚拟文件系统，只需卸载EFI分区
umount /mnt/boot

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

# 验证Samba配置（ZFS sharesmb 在 Linux 上不适用）

# 验证Samba配置文件语法
sudo testparm -s

# 查看共享列表
smbclient -L localhost -U username

# 测试共享访问
ls -la /share

echo "SMB共享验证完成"
```


### SMB用户管理 {#smb用户管理}

```sh
# 查看当前所有SMB用户
sudo pdbedit -L

# 查看特定用户的详细信息
sudo pdbedit -L -v -u username

# 添加新的SMB用户（用户必须先是系统用户）
sudo smbpasswd -a new_username

# 更改SMB用户密码
sudo smbpasswd username

# 禁用SMB用户（不删除）
sudo smbpasswd -d username

# 启用被禁用的SMB用户
sudo smbpasswd -e username

# 删除SMB用户
sudo smbpasswd -x username

# 检查SMB配置是否合法
sudo testparm

# 查看当前SMB连接状态
sudo smbstatus

echo "SMB用户管理完成"
```


### 系统更新 {#系统更新}

```sh
sudo apt update && sudo apt upgrade -y
```


### 安装zrepl自动快照系统 {#安装zrepl自动快照系统}

```sh
# 使用官方APT仓库安装zrepl
(
    set -ex
    zrepl_apt_key_url=https://zrepl.cschwarz.com/apt/apt-key.asc
    zrepl_apt_key_dst=/usr/share/keyrings/zrepl.gpg
    zrepl_apt_repo_file=/etc/apt/sources.list.d/zrepl.list

    # Install dependencies for subsequent commands
    sudo apt update && sudo apt install curl gnupg lsb-release

    # Deploy the zrepl apt key.
    curl -fsSL "$zrepl_apt_key_url" | tee | gpg --dearmor | sudo tee "$zrepl_apt_key_dst" > /dev/null

    # Add the zrepl apt repo.
    ARCH="$(dpkg --print-architecture)"
    CODENAME="$(lsb_release -i -s | tr '[:upper:]' '[:lower:]') $(lsb_release -c -s | tr '[:upper:]' '[:lower:]')"
    echo "Using Distro and Codename: $CODENAME"
    echo "deb [arch=$ARCH signed-by=$zrepl_apt_key_dst] https://zrepl.cschwarz.com/apt/$CODENAME main" | sudo tee "$zrepl_apt_repo_file" > /dev/null

    # Update apt repos.
    sudo apt update
)

# 安装zrepl
sudo apt install -y zrepl
```


### 配置zrepl自动快照系统 {#配置zrepl自动快照系统}

```sh
sudo tee /etc/zrepl/zrepl.yml << 'EOF'
global:
  logging:
    - type: stdout
      level: info
      format: human

jobs:
  - name: "hourly_snapshots"
    type: snap
    filesystems: {
      "rpool/ROOT/debian": true,
      "rpool/home<": true,
      "rpool/cache-*": false
    }
    snapshotting:
      type: periodic
      prefix: hourly_
      interval: 1h
    pruning:
      keep:
        - type: regex
          regex: "^apt_.*"  # 保留由 APT 钩子创建的快照
        - type: last_n
          count: 24  # 保留最近24个小时快照

  - name: "daily_snapshots"
    type: snap
    filesystems: {
      "rpool/ROOT/debian": true,
      "rpool/home<": true,
      "rpool/cache-*": false
    }
    snapshotting:
      type: periodic
      prefix: daily_
      interval: 24h
    pruning:
      keep:
        - type: regex
          regex: "^apt_.*"  # 保留由 APT 钩子创建的快照
        - type: last_n
          count: 30   # 保留最近30天的每日快照

  - name: "weekly_snapshots"
    type: snap
    filesystems: {
      "rpool/ROOT/debian": true,
      "rpool/home<": true,
      "rpool/cache-*": false
    }
    snapshotting:
      type: periodic
      prefix: weekly_
      interval: 168h
    pruning:
      keep:
        - type: regex
          regex: "^apt_.*"  # 保留由 APT 钩子创建的快照
        - type: last_n
          count: 12   # 保留最近12周的每周快照

  - name: "monthly_snapshots"
    type: snap
    filesystems: {
      "rpool/ROOT/debian": true,
      "rpool/home<": true,
      "rpool/cache-*": false
    }
    snapshotting:
      type: periodic
      prefix: monthly_
      interval: 720h
    pruning:
      keep:
        - type: regex
          regex: "^apt_.*"  # 保留由 APT 钩子创建的快照
        - type: last_n
          count: 12   # 保留最近12个月的每月快照
EOF

# 启用并启动zrepl服务
sudo systemctl enable zrepl
sudo systemctl start zrepl
```


## 第十一部分：创建APT快照管理系统 {#第十一部分-创建apt快照管理系统}


### 创建快照管理脚本 {#创建快照管理脚本}

```sh
sudo tee /usr/local/bin/zfs-apt-snapshot << 'EOF'
#!/usr/bin/env bash
# /usr/local/bin/zfs-apt-snapshot
# Create and prune ZFS snapshots around apt transactions.
# Snapshots are named: apt_{pre|post}_YYYYmmdd_HHMMSS
# Per-dataset retention: keep newest MAX_SNAPSHOTS, prune older ones.

set -uo pipefail

SNAPSHOT_PREFIX="apt"
MAX_SNAPSHOTS=50
DATASETS=("rpool/ROOT/debian" "rpool/home")
LOG_FILE="/var/log/zfs-apt-snapshots.log"
TIMESTAMP="$(date +%Y%m%d_%H%M%S)"

# ----- utils -----
log() {
  local msg="$1"
  mkdir -p "$(dirname "$LOG_FILE")" 2>/dev/null || true
  printf '[%s] %s\n' "$(date '+%F %T')" "$msg" >>"$LOG_FILE"
}

require_cmd() {
  command -v "$1" >/dev/null 2>&1 || { log "missing command: $1"; return 1; }
}

# Single-instance lock to avoid overlapping hooks
acquire_lock() {
  exec 9>/run/zfs-apt-snapshot.lock || exec 9>/tmp/zfs-apt-snapshot.lock
  flock -n 9 || { log "another instance is running; skipping"; return 1; }
}

# ----- core -----
create_snapshot() {
  local phase="$1"            # "pre" or "post"
  local snap="${SNAPSHOT_PREFIX}_${phase}_${TIMESTAMP}"

  log "creating ${phase} snapshots with name: ${snap}"

  for ds in "${DATASETS[@]}"; do
    if zfs list -H -o name "$ds" >/dev/null 2>&1; then
      local full="${ds}@${snap}"
      if zfs snapshot "$full" >/dev/null 2>&1; then
        log "created: $full"
      else
        log "error creating: $full"
      fi
    else
      log "dataset not found, skip: $ds"
    fi
  done
}

cleanup_snapshots() {
  log "pruning old apt snapshots (keep newest ${MAX_SNAPSHOTS} per dataset)"

  for ds in "${DATASETS[@]}"; do
    zfs list -H -o name "$ds" >/dev/null 2>&1 || continue

    # List snapshots for this exact dataset, newest first.
    # Use -r to allow listing children then filter exact match on the left of '@'.
    # shellcheck disable=SC2016
    mapfile -t snaps < <(
      zfs list -H -t snapshot -o name -S creation -r "$ds" 2>/dev/null \
      | awk -v ds="$ds" -v pfx="$SNAPSHOT_PREFIX" -F'@' '$1==ds && $2 ~ "^"pfx"_" {print $0}'
    )

    local count="${#snaps[@]}"
    if (( count <= MAX_SNAPSHOTS )); then
      log "dataset $ds: $count snapshots, no pruning needed"
      continue
    fi

    local to_delete=$((count - MAX_SNAPSHOTS))
    log "dataset $ds: $count snapshots, deleting $to_delete older snapshots"

    local deleted=0
    for ((i=MAX_SNAPSHOTS; i<count; i++)); do
      s="${snaps[$i]}"
      if zfs destroy "$s" >/dev/null 2>&1; then
        ((deleted++))
        log "destroyed: $s"
      else
        log "failed to destroy: $s"
      fi
    done
    log "dataset $ds: prune complete, deleted $deleted"
  done
}

main() {
  # Defensive checks. Never abort apt; just log and exit 0.
  require_cmd zfs || return 0
  acquire_lock || return 0

  case "${1:-}" in
    pre)
      create_snapshot "pre"
      ;;
    post)
      create_snapshot "post"
      cleanup_snapshots
      ;;
    *)
      echo "Usage: $0 {pre|post}"
      echo "  pre  - create snapshots before apt transaction"
      echo "  post - create snapshots after apt transaction and prune old ones"
      ;;
  esac

  return 0
}

main "$@" || true
exit 0
EOF

# Set executable permission
sudo chmod +x /usr/local/bin/zfs-apt-snapshot
```


### 创建APT钩子 {#创建apt钩子}

```sh
# 创建APT pre-invoke钩子
sudo tee /etc/apt/apt.conf.d/00-zfs-snapshot-pre << 'EOF'
DPkg::Pre-Invoke { "/usr/local/bin/zfs-apt-snapshot pre"; };
EOF

# 创建APT post-invoke钩子
sudo tee /etc/apt/apt.conf.d/99-zfs-snapshot-post << 'EOF'
DPkg::Post-Invoke { "/usr/local/bin/zfs-apt-snapshot post"; };
EOF
```


### 测试快照系统 {#测试快照系统}

```sh
# Test pre and post snapshot creation
echo "Testing snapshot creation..."
sudo /usr/local/bin/zfs-apt-snapshot pre
sleep 2
sudo /usr/local/bin/zfs-apt-snapshot post

echo "Listing created snapshots:"
zfs list -t snapshot | grep apt

echo "Deleting test snapshots..."
zfs list -t snapshot | grep apt | awk '{print $1}' | xargs -r -n1 sudo zfs destroy

echo "Verifying snapshots are deleted:"
zfs list -t snapshot | grep apt || echo "No apt snapshots, test complete"

# Verify zrepl service status
sudo systemctl status zrepl
```


### 配置休眠支持（Hibernation） {#配置休眠支持-hibernation}

```sh
# 获取交换分区的UUID
SWAP_UUID=$(blkid -s UUID -o value ${DISK1}-part2)
echo "交换分区UUID: $SWAP_UUID"

# 获取交换分区的偏移量（如果使用swapfile则需要）
# 对于专用交换分区，通常偏移量为0

# 更新GRUB配置以支持休眠
sudo sed -i "s|GRUB_CMDLINE_LINUX=\"root=ZFS=rpool/ROOT/debian\"|GRUB_CMDLINE_LINUX=\"root=ZFS=rpool/ROOT/debian resume=UUID=$SWAP_UUID\"|" /etc/default/grub

# 更新GRUB配置
sudo update-grub

# 配置initramfs以支持休眠恢复
echo 'RESUME=UUID='$SWAP_UUID | sudo tee -a /etc/initramfs-tools/conf.d/resume

# 更新initramfs
sudo update-initramfs -u -k all
```


### 测试休眠功能 {#测试休眠功能}

```sh
# 检查交换分区状态
swapon --show
cat /proc/swaps

# 检查休眠支持
cat /sys/power/disk

# 检查可用的睡眠模式
cat /sys/power/state

# 检查内存使用情况（确保交换分区足够大）
free -h

# 测试休眠（谨慎使用，确保保存了重要工作）
echo "准备测试休眠，请确保保存了所有重要工作"
echo "执行: sudo systemctl hibernate"
```


### 配置zram压缩内存交换 {#配置zram压缩内存交换}

```sh
# 安装systemd-zram-generator（现代统一的zram管理工具）
sudo apt install -y systemd-zram-generator

# 配置zram
sudo tee /etc/systemd/zram-generator.conf << 'EOF'
# zram配置文件

[zram0]
# zram设备大小（总内存的百分比或绝对值）
# 建议设置为总内存的25-50%
zram-size = ram * 0.25

# 压缩算法（lz4, lzo, zstd）
compression-algorithm = zstd

# 交换优先级（高于磁盘交换）
swap-priority = 100

# 文件系统类型
fs-type = swap
EOF

# 启动zram设备
sudo systemctl daemon-reload
sudo systemctl start systemd-zram-setup@zram0.service
sudo systemctl enable systemd-zram-setup@zram0.service

# 验证zram配置
echo "zram配置完成，当前状态:"
sudo zramctl
swapon --show
```


### 优化zram和磁盘交换配置 {#优化zram和磁盘交换配置}

```sh
# 查看当前交换配置
swapon --show
cat /proc/swaps

# 确认zram优先级高于磁盘交换
# zram应该显示更高的优先级数值

# 设置内存交换倾向性（可选）
# 数值越低，越倾向于使用内存而非交换
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf

# 立即应用设置
sudo sysctl vm.swappiness=10

# 验证设置
cat /proc/sys/vm/swappiness

echo "zram和交换优化完成"
echo "zram设备: $(sudo zramctl --output-all | grep -c zram)"
echo "总交换空间: $(free -h | grep Swap | awk '{print $2}')"
```


## 第十二部分：ZFS维护管理 {#第十二部分-zfs维护管理}


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
sudo smartctl -a $DISK1
sudo smartctl -a $DISK2
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


## 第十三部分：故障排除 {#第十三部分-故障排除}


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
5.  挂载EFI分区：=mount /dev/disk/by-id/nvme-SAMSUNG_SSD_980_1TB_S649NJ0R123456A-part1 /mnt/boot=
6.  进入chroot：=chroot /mnt=
7.  执行修复操作


## 常用命令速查 {#常用命令速查}

```sh
# ZFS管理
sudo zpool status         # 查看存储池状态
sudo zfs list            # 查看所有数据集
sudo zpool scrub rpool   # 手动执行scrub
sudo zpool trim rpool    # 手动执行trim

# RAID0条带管理
sudo zpool status -v rpool           # 查看详细条带状态
zpool iostat -v rpool               # 查看条带性能统计
sudo smartctl -a $DISK1             # 检查磁盘健康（关键！）
sudo smartctl -a $DISK2             # 检查磁盘健康（关键！）

# 定时器管理
systemctl list-timers | grep zfs     # 查看ZFS定时器
sudo systemctl start zfs-scrub-weekly@rpool.service   # 手动执行scrub
sudo systemctl start zfs-trim@rpool.service    # 手动执行trim

# 快照管理
sudo zfs snapshot rpool/ROOT/debian@backup-$(date +%Y%m%d)   # 创建快照
zfs list -t snapshot                 # 查看快照
sudo /usr/local/bin/zfs-apt-snapshot pre   # 手动创建pre快照
tail -f /var/log/zfs-apt-snapshots.log     # 查看快照日志
zfs list -t snapshot | grep apt            # 查看apt快照
sudo systemctl status zrepl                # 查看zrepl状态
sudo journalctl -u zrepl -f               # 查看zrepl实时日志

# 休眠管理
swapon --show                              # 检查交换分区状态
sudo systemctl hibernate                  # 休眠系统
cat /sys/power/state                      # 查看可用睡眠模式
free -h                                   # 检查内存使用情况

# zram管理
sudo zramctl                               # 查看zram设备状态
sudo systemctl status systemd-zram-setup@zram0.service  # 查看zram服务状态
cat /proc/sys/vm/swappiness               # 查看交换倾向性设置
sudo systemctl restart systemd-zram-setup@zram0.service # 重启zram服务

# 性能监控
zpool iostat 1           # 实时I/O统计
sudo zfs get compressratio rpool/ROOT/debian    # 查看压缩比

# SMB共享管理
sudo systemctl status smbd              # 检查SMB服务状态
smbclient -L localhost -U username      # 查看共享列表
sudo smbpasswd -a username              # 添加SMB用户
ls -la /share                          # 检查共享目录权限

# SMB共享管理
sudo systemctl status smbd              # 检查SMB服务状态
sudo testparm -s                       # 验证Samba配置语法
smbclient -L localhost -U username      # 查看共享列表
sudo smbstatus                          # 查看当前SMB连接状态
ls -la /share                          # 检查共享目录权限

# SMB用户管理
sudo pdbedit -L                         # 查看所有SMB用户
sudo smbpasswd -a username              # 添加SMB用户
sudo smbpasswd username                 # 更改SMB用户密码
sudo smbpasswd -d username              # 禁用SMB用户
sudo smbpasswd -e username              # 启用SMB用户
sudo smbpasswd -x username              # 删除SMB用户

# 维护操作
sudo zpool status -v     # 详细池状态
sudo journalctl -u zfs-import-cache.service   # 查看ZFS日志
```


### 安装 Podman 与兼容层 {#安装-podman-与兼容层}

```sh
# 安装 Podman 及 docker 兼容层和 compose 支持
apt install -y podman podman-docker podman-compose

# 基本自检
podman info
podman run --rm hello-world
```


## 附录：Podman 常用命令与日常维护 {#附录-podman-常用命令与日常维护}


### 基本信息与运行 {#基本信息与运行}

```sh
podman info                 # 查看系统与存储信息
podman run --rm hello-world # 快速自检运行
```


### 镜像与容器管理 {#镜像与容器管理}

```sh
podman images               # 列出镜像
podman ps -a                # 列出容器（包含已停止）
podman pull alpine          # 拉取镜像
podman run -d --name web -p 8080:80 nginx:alpine
podman logs -f web          # 跟随日志
podman exec -it web sh      # 进入容器
podman stop web && podman rm web
```


### 清理与空间回收 {#清理与空间回收}

```sh
podman image prune -a       # 清理未使用镜像
podman container prune      # 清理已退出容器
podman volume ls            # 查看卷
podman volume prune         # 清理未使用卷
podman system df            # 存储占用统计
podman system prune -a      # 全面清理（谨慎）
```


### Quadlet 自启（推荐，替代 podman generate） {#quadlet-自启-推荐-替代-podman-generate}

Podman 现已推荐使用 Quadlet（.container 文件）而非 =podman generate systemd=。

-   rootless 放置路径：=~/.config/containers/systemd/=
-   rootful 放置路径：=/etc/containers/systemd/=

以 Caddy 为例，创建一个 =caddy.container=：

```ini
[Unit]
Description=Caddy file server
After=network-online.target
Wants=network-online.target

[Container]
Image=docker.io/caddy:latest
ContainerName=caddy
Volume=/share:/srv/share:ro
PublishPort=8080:8080
Exec=caddy file-server --root /srv/share --listen :8080 --browse
Pull=always
AutoUpdate=registry

[Service]
Restart=always

[Install]
WantedBy=default.target
```

Quadlet 运行机制要点：

-   段名：源文件使用 [Container]；[X-Container] 仅出现在生成输出中。
-   其他段：可额外写入 [Service]、[Install]，systemd 会照常解析。
-   重启策略：在 [Container] 中没有 Restart/RestartPolicy；需在 [Service] 中写 Restart=always（Podman 无 unless-stopped 与 always 的区别，always 等价）。
-   自启动：[Install] 中加入 WantedBy=default.target；执行 \`systemctl --user daemon-reload\` 后，Quadlet 会生成标记为 generated 的 .service 并挂到 default.target。generated 单元不能 enable，但已通过 [Install] 关联目标。
-   开机与 linger：
    -   未启用 linger 时，user manager 只在登录后运行，容器登录后才启动。
    -   启用 linger 后（\`sudo loginctl enable-linger $USER\`），user manager 开机即运行，容器随系统自启。

实际使用步骤：

1.  写 .container 到 \`~/.config/containers/systemd/\`，包含 [Container]、[Service] Restart=always、[Install] WantedBy=default.target。
2.  执行：
    ```sh
    systemctl --user daemon-reload
    # 立即运行（可选，但建议执行一次以立刻生效）
    systemctl --user start caddy.service
    ```
3.  确认：
    ```sh
    systemctl --user status caddy.service
    ```
4.  如需真正随开机启动（无需登录）：
    ```sh
    sudo loginctl enable-linger "$USER"
    ```

5.  rootless 部署与启动：

<!--listend-->

```sh
mkdir -p ~/.config/containers/systemd
cp caddy.container ~/.config/containers/systemd/
systemctl --user daemon-reload
systemctl --user start caddy.service
# 查看状态/日志
systemctl --user status caddy.service
journalctl --user -u caddy -f
# （可选）开机自启用户服务
loginctl enable-linger "$USER"
```

-   rootful 部署与启动（系统服务）：

<!--listend-->

```sh
sudo mkdir -p /etc/containers/systemd
sudo cp caddy.container /etc/containers/systemd/
sudo systemctl daemon-reload
sudo systemctl start caddy.service
```

-   自动更新（可选，对应 =AutoUpdate=registry=）：

<!--listend-->

```sh
# rootless
systemctl --user enable --now podman-auto-update.timer
# rootful
sudo systemctl enable --now podman-auto-update.timer
```

提示：rootless 模式下直接映射到宿主 1024 以下端口（如 80）可能受限，若失败可：

-   使用 &gt;=1024 的宿主端口（如 8080:80），或
-   调整 sysctl：sudo sysctl net.ipv4.ip_unprivileged_port_start=0，或改用 rootful。


### 存储位置（与 ZFS 数据集对应） {#存储位置-与-zfs-数据集对应}

-   rootful：/var/lib/containers/storage  （rpool/containers）
-   rootless：~/.local/share/containers/storage  （rpool/containers-${user_debian}）


## 附录：NVIDIA显卡禁用配置 {#附录-nvidia显卡禁用配置}

NVIDIA显卡使用开源驱动nouveau可能导致hibernate无法正常工作，因此在启用休眠功能的系统上建议禁用NVIDIA显卡以确保休眠稳定性。


### 禁用NVIDIA显卡（使用bbswitch） {#禁用nvidia显卡-使用bbswitch}

```sh
# 禁用nouveau驱动
echo "blacklist nouveau" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
echo "options nouveau modeset=0" | sudo tee -a /etc/modprobe.d/blacklist-nouveau.conf
sudo update-initramfs -u
sudo reboot

# 安装bbswitch
sudo apt install bbswitch-dkms

# 加载bbswitch模块
sudo modprobe bbswitch

# 关闭NVIDIA GPU
echo "OFF" | sudo tee /proc/acpi/bbswitch

# 检查状态（应该显示"OFF"）
cat /proc/acpi/bbswitch

# 使配置永久生效
echo "bbswitch" | sudo tee -a /etc/modules
echo "options bbswitch load_state=0 unload_state=0" | sudo tee /etc/modprobe.d/bbswitch.conf
```


### 验证NVIDIA显卡禁用状态 {#验证nvidia显卡禁用状态}

```sh
# 检查bbswitch状态
cat /proc/acpi/bbswitch

# 检查PCI设备状态
lspci | grep -i nvidia

# 检查模块加载状态
lsmod | grep -E "(nouveau|nvidia|bbswitch)"

# 检查电源管理
sudo powertop
```


## 附录：Micron 2100 NVMe硬盘优化 {#附录-micron-2100-nvme硬盘优化}


### 添加内核参数解决延迟问题 {#添加内核参数解决延迟问题}

```sh
# 针对Micron 2100 NVMe硬盘，可能需要禁用电源管理以避免延迟问题
# 编辑GRUB配置文件
sudo nano /etc/default/grub

# 在GRUB_CMDLINE_LINUX行中添加参数：
# GRUB_CMDLINE_LINUX="root=ZFS=rpool/ROOT/debian nvme_core.default_ps_max_latency_us=0"

# 或者使用sed自动添加参数
sudo sed -i 's|GRUB_CMDLINE_LINUX="root=ZFS=rpool/ROOT/debian"|GRUB_CMDLINE_LINUX="root=ZFS=rpool/ROOT/debian nvme_core.default_ps_max_latency_us=0"|' /etc/default/grub

# 更新GRUB配置
sudo update-grub

# 重启系统使参数生效
sudo reboot
```


### 验证内核参数生效 {#验证内核参数生效}

```sh
# 检查当前内核参数
cat /proc/cmdline

# 检查NVMe设备状态
lspci | grep -i nvme
nvme list

# 检查NVMe电源管理状态
cat /sys/module/nvme_core/parameters/default_ps_max_latency_us
```
