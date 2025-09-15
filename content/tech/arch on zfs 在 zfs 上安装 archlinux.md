+++
title = "arch on zfs 在 zfs 上安装 archlinux"
author = ["wang1zhen"]
date = 2025-09-15T00:00:00+09:00
draft = false
+++

## 第一部分：准备安装介质 {#第一部分-准备安装介质}


### 安装archiso工具 {#安装archiso工具}

```sh
sudo pacman -S archiso
```


### 复制并自定义配置 {#复制并自定义配置}

```sh
# 复制官方配置
cp -r /usr/share/archiso/configs/releng/ ~/archlive
cd ~/archlive
```


### 修改包列表 {#修改包列表}

```sh
# 删除不需要的包并添加新包
sed -i '/^linux$/d; /^linux-headers$/d; /^broadcom-wl$/d' packages.x86_64
echo -e "linux-lts\nlinux-lts-headers\nzfs-utils\nzfs-dkms" >> packages.x86_64

echo "packages.x86_64 文件已更新完成"
```


### 配置pacman源 {#配置pacman源}

```sh
# 在文件末尾添加archzfs仓库
cat >> pacman.conf << 'EOF'

[archzfs]
SigLevel = TrustAll Optional
Server = http://archzfs.com/$repo/$arch
EOF

pacman-key --recv-keys DDF7DB817396A49B2A2723F7403BD972F75D9D76
pacman-key --lsign-key DDF7DB817396A49B2A2723F7403BD972F75D9D76

# 更新包数据库
pacman -Sy

echo "pacman.conf 已自动配置archzfs仓库"
```


### 更新启动配置文件 {#更新启动配置文件}

```sh
# 定义需要修改的文件列表
config_files=(
    "airootfs/etc/mkinitcpio.d/linux.preset"
    "efiboot/loader/entries/01-archiso-x86_64-linux.conf"
    "efiboot/loader/entries/02-archiso-x86_64-speech-linux.conf"
    "syslinux/archiso_pxe-linux.cfg"
    "syslinux/archiso_sys-linux.cfg"
    "grub/loopback.cfg"
)

# 批量处理所有配置文件
for file in "${config_files[@]}"; do
    if [[ -f "$file" ]]; then
        echo "正在更新 $file"
        sed -i 's/vmlinuz-linux\([^-]\|$\)/vmlinuz-linux-lts\1/g; s/initramfs-linux/initramfs-linux-lts/g' "$file"
        echo "✓ $file 已更新"
    else
        echo "⚠ 警告: $file 不存在，跳过"
    fi
done

echo "所有启动配置文件已更新完成"
```


### 构建ISO {#构建iso}

```sh
mkdir -p ~/isobuild
sudo mkarchiso -v -r -w /tmp/archiso-tmp -o ~/isobuild ~/archlive
```

构建完成后，ISO文件将位于 `~/isobuild` 目录中。


## 第二部分：启动并配置网络环境 {#第二部分-启动并配置网络环境}


### 启动到Live环境 {#启动到live环境}


### 配置无线网络 {#配置无线网络}

```sh
iwctl
```

在iwctl命令行中：

```text
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect SSID_NAME
exit
```


### 设置系统时间 {#设置系统时间}

```sh
timedatectl set-ntp true
```


### 配置archzfs仓库和密钥（Live环境） {#配置archzfs仓库和密钥-live环境}

为了pacstrap安装zfs包，需要在Live环境中也配置archzfs：

```sh
# 添加archzfs仓库到Live环境的pacman.conf
cat >> /etc/pacman.conf << 'EOF'

[archzfs]
SigLevel = TrustAll Optional
Server = http://archzfs.com/$repo/$arch
EOF

# 添加ZFS密钥
pacman-key --recv-keys DDF7DB817396A49B2A2723F7403BD972F75D9D76
pacman-key --lsign-key DDF7DB817396A49B2A2723F7403BD972F75D9D76

# 更新包数据库
pacman -Sy

echo "Live环境archzfs配置完成"
```


### 启用SSH {#启用ssh}

```sh
passwd  # 设置root密码
systemctl start sshd
ip a    # 查看IP地址
```


## 第三部分：磁盘分区 {#第三部分-磁盘分区}


### 识别目标磁盘 {#识别目标磁盘}

```sh
lsblk
fdisk -l
```

以下示例假设目标磁盘为 =/dev/nvme0n1=，请根据实际情况调整。


### 创建GPT分区表 {#创建gpt分区表}

```sh
gdisk /dev/nvme0n1
```

在gdisk中执行以下操作：

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


### 格式化分区 {#格式化分区}

```sh
# 格式化EFI分区
mkfs.fat -F32 /dev/nvme0n1p1

# 设置交换分区
mkswap /dev/nvme0n1p2
swapon /dev/nvme0n1p2
```


### 查看磁盘ID {#查看磁盘id}

```sh
ls -lh /dev/disk/by-id/
```

记录目标磁盘的by-id路径，用于ZFS配置。


## 第四部分：ZFS配置 {#第四部分-zfs配置}


### 创建ZFS存储池 {#创建zfs存储池}

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
      zroot /dev/disk/by-id/nvme-SAMSUNG_SSD_970_EVO_Plus_1TB_S4EWNX0M123456A-part3
```

参数说明：

-   =ashift=12=：针对4K扇区磁盘优化
-   =autotrim=on=：自动启用TRIM（适用于SSD）
-   =compression=zstd=：使用zstd压缩算法
-   =relatime=on=：相对atime更新，性能更好
-   =xattr=sa=：扩展属性存储在系统属性中
-   =dnodesize=auto=：自动调整dnode大小
-   =recordsize=128K=：默认记录大小，适合大多数用途


### 创建ZFS数据集 {#创建zfs数据集}

```sh
# 创建根容器数据集
zfs create -o mountpoint=none zroot/ROOT

# 创建系统根数据集
zfs create -o mountpoint=/ -o canmount=noauto zroot/ROOT/root

# 创建用户相关数据集
zfs create -o mountpoint=/home zroot/home

# 创建系统数据集
zfs create -o mountpoint=/var zroot/var
zfs create -o mountpoint=/var/log zroot/var/log
zfs create -o mountpoint=/var/cache zroot/var/cache

# 创建游戏数据集 - 大文件优化
zfs create \
    -o mountpoint=/games \
    -o compression=zstd \
    -o recordsize=1M \
    -o atime=off \
    zroot/games

# 创建 Podman 数据集（rootful）- 小文件优化
zfs create \
    -o mountpoint=/var/lib/containers \
    -o compression=zstd \
    -o recordsize=64K \
    -o atime=off \
    zroot/containers

# 创建 Podman 数据集（rootless，可选）
# 设置变量：新系统的主要用户名（请替换为实际用户名）
user_arch="YOUR_USERNAME"

zfs create \
    -o mountpoint="/home/${user_arch}/.local/share/containers" \
    -o compression=zstd \
    -o recordsize=64K \
    -o atime=off \
    zroot/containers-${user_arch}
# 创建临时文件数据集 - 性能优化
zfs create \
    -o mountpoint=/tmp \
    -o compression=off \
    -o sync=disabled \
    -o atime=off \
    -o devices=off \
    -o exec=on \
    -o setuid=off \
    zroot/tmp

# 创建SMB共享数据集 - 私有共享，性能优化（Samba 另行配置）
zfs create \
    -o mountpoint=/share \
    -o atime=off \
    zroot/share
```


### 验证数据集创建 {#验证数据集创建}

```sh
zfs list -t filesystem
```


### 优化ZFS设置 {#优化zfs设置}

```sh
# 设置压缩算法
zfs set compression=zstd zroot/ROOT/root
zfs set compression=zstd zroot/home
zfs set compression=zstd zroot/var
zfs set compression=zstd zroot/var/log
zfs set compression=zstd zroot/var/cache
zfs set compression=zstd zroot/share

# 优化recordsize设置
zfs set recordsize=128K zroot/ROOT/root
zfs set recordsize=128K zroot/home
zfs set recordsize=1M zroot/share

# 设置缓存文件
zpool set cachefile=/etc/zfs/zpool.cache zroot
```


### 重新导入ZFS池（确保配置正确） {#重新导入zfs池-确保配置正确}

```sh
swapoff -a
zpool export zroot
zpool import -d /dev/disk/by-id -R /mnt zroot -N

# 挂载文件系统
zfs mount zroot/ROOT/root
zfs mount -a

# 设置启动文件系统
zpool set bootfs=zroot/ROOT/root zroot
```


### 验证挂载 {#验证挂载}

```sh
df -h
mount | grep zfs
```


## 第五部分：安装Arch Linux {#第五部分-安装arch-linux}


### 挂载EFI分区 {#挂载efi分区}

```sh
mkdir -p /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot

# 重新启用交换分区
swapon /dev/nvme0n1p2
```


### 安装基础系统 {#安装基础系统}

```sh
pacstrap -K /mnt base base-devel linux-lts linux-lts-headers linux-firmware \
         amd-ucode vim man-db man-pages texinfo grub efibootmgr \
         networkmanager openssh git wget curl \
         zfs-dkms zfs-utils plasma-meta sddm \
         samba
```

说明：

-   使用 `linux-lts` 而非标准内核以确保ZFS兼容性
-   根据CPU类型选择 `amd-ucode` 或 `intel-ucode`
-   直接安装桌面环境 `plasma-meta` 和显示管理器 `sddm`
-   包含ZFS支持包 `zfs-dkms` 和 `zfs-utils`


### 生成fstab {#生成fstab}

```sh
genfstab -U -p /mnt >> /mnt/etc/fstab
```


### 检查并编辑fstab {#检查并编辑fstab}

```sh
vim /mnt/etc/fstab
```

确保只保留EFI分区和swap分区的条目，移除ZFS相关条目（ZFS自己管理挂载）。


## 第六部分：系统配置 {#第六部分-系统配置}


### 进入chroot环境 {#进入chroot环境}

```sh
arch-chroot /mnt
```


### 配置时区 {#配置时区}

```sh
ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
hwclock --systohc
```


### 配置语言环境 {#配置语言环境}

```sh
# 配置语言生成
cat > /etc/locale.gen << EOF
en_US.UTF-8 UTF-8
ja_JP.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
EOF

# 生成语言文件
locale-gen

# 设置系统语言
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
```


### 配置网络 {#配置网络}

```sh
# 设置主机名
echo 'ganymede' > /etc/hostname

# 配置hosts文件
cat > /etc/hosts << EOF
127.0.0.1   localhost
::1         localhost
127.0.1.1   ganymede.localdomain ganymede
EOF
```


### 配置pacman和ZFS {#配置pacman和zfs}

```sh
# 添加archzfs仓库到pacman.conf
cat >> /etc/pacman.conf << 'EOF'

[archzfs]
SigLevel = TrustAll Optional
Server = http://archzfs.com/$repo/$arch
EOF
```

添加ZFS密钥：

```sh
pacman-key --recv-keys DDF7DB817396A49B2A2723F7403BD972F75D9D76
pacman-key --lsign-key DDF7DB817396A49B2A2723F7403BD972F75D9D76
```


### 配置initramfs {#配置initramfs}

```sh
vim /etc/mkinitcpio.conf
```

修改HOOKS行为：

```text
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block zfs filesystems)
```

**确保keyboard在zfs之前，zfs在filesystems之前。**

重新生成initramfs：

```sh
mkinitcpio -P
```


### 配置ZFS服务 {#配置zfs服务}

```sh
# 生成host ID
zgenhostid $(hostid)

# 设置缓存文件
mkdir -p /etc/zfs
zpool set cachefile=/etc/zfs/zpool.cache zroot

# 启用ZFS服务
systemctl enable zfs.target
systemctl enable zfs-import-cache.service
systemctl enable zfs-mount.service
systemctl enable zfs-import.target
```

**zgenhostid的作用与必要性：** `zgenhostid`
用于生成ZFS系统的唯一主机ID，这个ID存储在 `/etc/hostid`
文件中。它的作用是：

-   防止意外导入属于其他主机的ZFS池，避免数据损坏
-   确保ZFS池只能被正确的主机导入和访问
-   在系统迁移时提供池所有权验证
-   是ZFS多主机环境下的安全机制


### 安装和配置GRUB {#安装和配置grub}

```sh
# 安装GRUB到EFI分区
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ArchLinux

# 编辑GRUB配置
vim /etc/default/grub
```

修改GRUB_CMDLINE_LINUX_DEFAULT行：

```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet root=ZFS=zroot/ROOT/root"
```

生成GRUB配置：

```sh
grub-mkconfig -o /boot/grub/grub.cfg
```


### 创建用户 {#创建用户}

```sh
useradd -m -G wheel -s /bin/bash username
passwd username
```

配置sudo权限：

```sh
EDITOR=vim visudo
```

取消注释并修改以下行：

```text
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
```


### 设置/games目录权限 {#设置-games目录权限}

```sh
# 创建游戏目录并设置权限，确保uid1000用户可写入
mkdir -p /games
chown 1000:1000 /games
chmod 755 /games
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
    server string = Arch ZFS Server

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
systemctl enable smb
systemctl enable nmb

echo "SMB共享配置完成：\\\\server\\share"
```


### 启用基本服务 {#启用基本服务}

```sh
systemctl enable NetworkManager
systemctl enable sddm  # 启用SDDM显示管理器
```


### Hibernation配置 {#hibernation配置}

```sh
# 获取swap分区的UUID
SWAP_UUID=$(blkid -s UUID -o value /dev/nvme0n1p2)

# 使用sed添加resume参数到GRUB配置
sed -i "s/root=ZFS=zroot\/ROOT\/root/& resume=UUID=$SWAP_UUID/" /etc/default/grub

# 在mkinitcpio.conf中添加resume钩子
sed -i '/^HOOKS=/ { /resume/!s/ zfs/ resume zfs/ }' /etc/mkinitcpio.conf

# 重新生成initramfs
mkinitcpio -P

# 重新生成GRUB配置
grub-mkconfig -o /boot/grub/grub.cfg

echo "Hibernation配置完成"
```


### 配置ZFS定期维护服务 {#配置zfs定期维护服务}

```sh
# OpenZFS 2.1.3+ 已包含内置的scrub定时器，无需手动创建
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
if /usr/bin/zpool status %i | grep "trimming"; then\
exec /usr/bin/zpool wait -t trim %i;\
else exec /usr/bin/zpool trim -w %i; fi'
ExecStop=-/bin/sh -c '/usr/bin/zpool trim -s %i 2>/dev/null || true'

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

# 启用内置的scrub定时器和自定义的trim定时器（针对zroot池）
systemctl enable zfs-scrub-monthly@zroot.timer
systemctl enable zfs-trim@zroot.timer

echo "ZFS定期维护服务配置完成"
```


### 最终验证 {#最终验证}

```sh
# 验证ZFS状态
zpool status
zfs list

# 验证initramfs包含ZFS和resume
lsinitcpio /boot/initramfs-linux-lts.img | grep -E "(zfs|resume)"

# 验证GRUB配置
grep -E "(zroot|resume)" /boot/grub/grub.cfg

# 验证定时器
systemctl list-timers | grep zfs
```


## 第七部分：创建Pacman快照管理系统 {#第七部分-创建pacman快照管理系统}


### 创建快照管理脚本 {#创建快照管理脚本}

```sh
cat > /usr/local/bin/zfs-pacman-snapshot << 'EOF'
#!/usr/bin/env bash
# /usr/local/bin/zfs-pacman-snapshot
# Create and prune ZFS snapshots around pacman transactions.
# Snapshots are named: pacman_{pre|post}_YYYYmmdd_HHMMSS
# Per-dataset retention: keep newest MAX_SNAPSHOTS, prune older ones.

set -uo pipefail

SNAPSHOT_PREFIX="pacman"
MAX_SNAPSHOTS=50
DATASETS=("zroot/ROOT/root" "zroot/var" "zroot/home")
LOG_FILE="/var/log/zfs-pacman-snapshots.log"
TIMESTAMP="$(date +%Y%m%d_%H%M%S)"

# ----- utils -----
log() {
  local msg="$1"
  # ensure dir exists
  mkdir -p "$(dirname "$LOG_FILE")" 2>/dev/null || true
  printf '[%s] %s\n' "$(date '+%F %T')" "$msg" >>"$LOG_FILE"
}

require_cmd() {
  command -v "$1" >/dev/null 2>&1 || { log "missing command: $1"; return 1; }
}

# Single-instance lock to avoid overlapping hooks
acquire_lock() {
  exec 9>/run/zfs-pacman-snapshot.lock || exec 9>/tmp/zfs-pacman-snapshot.lock
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
  log "pruning old pacman snapshots (keep newest ${MAX_SNAPSHOTS} per dataset)"

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

    # Delete from index MAX_SNAPSHOTS onward (older ones)
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
  # Defensive checks. Never abort pacman; just log and exit 0.
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
      echo "  pre  - create snapshots before pacman transaction"
      echo "  post - create snapshots after pacman transaction and prune old ones"
      ;;
  esac

  return 0
}

main "$@" || true
exit 0
EOF

# 设置执行权限
chmod +x /usr/local/bin/zfs-pacman-snapshot
```


### 测试快照脚本 {#测试快照脚本}

```sh
# 测试创建pre和post快照
echo "测试创建快照..."
/usr/local/bin/zfs-pacman-snapshot pre
sleep 2
/usr/local/bin/zfs-pacman-snapshot post

echo "列出创建的快照:"
zfs list -t snapshot | grep pacman

echo "删除测试快照..."
zfs list -t snapshot | grep pacman | awk '{print $1}' | xargs -r -n1 zfs destroy

echo "验证快照已删除:"
zfs list -t snapshot | grep pacman || echo "无pacman快照，测试完成"
```


### 创建Pacman快照钩子 {#创建pacman快照钩子}

```sh
# 创建pacman hook目录
mkdir -p /etc/pacman.d/hooks

# 创建pre-transaction hook
cat > /etc/pacman.d/hooks/00-zfs-pre.hook << 'EOF'
[Trigger]
Operation = Install
Operation = Upgrade
Operation = Remove
Type = Package
Target = *

[Action]
Description = Creating ZFS snapshot before package transaction...
When = PreTransaction
Exec = /usr/local/bin/zfs-pacman-snapshot pre
EOF

# 创建post-transaction hook
cat > /etc/pacman.d/hooks/99-zfs-post.hook << 'EOF'
[Trigger]
Operation = Install
Operation = Upgrade
Operation = Remove
Type = Package
Target = *

[Action]
Description = Creating ZFS snapshot after package transaction...
When = PostTransaction
Exec = /usr/local/bin/zfs-pacman-snapshot post
EOF
```


## 第八部分：完成安装 {#第八部分-完成安装}


### 退出chroot并清理 {#退出chroot并清理}

```sh
# 退出chroot环境
exit

# 卸载文件系统
umount /mnt/boot
zfs umount -a
zpool export zroot
```


### 重启系统 {#重启系统}

```sh
systemctl reboot
```


## 第九部分：首次启动后配置 {#第九部分-首次启动后配置}


### 验证系统状态 {#验证系统状态}

```sh
# 检查ZFS状态
sudo zpool status
sudo zfs list

# 检查挂载点
df -h
mount | grep zfs

# 检查服务状态
systemctl status zfs.target
systemctl status NetworkManager
systemctl status sddm

# 检查定时器状态
systemctl list-timers | grep zfs
```


### 启动ZFS定期维护服务 {#启动zfs定期维护服务}

```sh
# 启动定时器
sudo systemctl start zfs-scrub-monthly@zroot.timer
sudo systemctl start zfs-trim@zroot.timer

# 验证定时器状态
systemctl status zfs-scrub-monthly@zroot.timer
systemctl status zfs-trim@zroot.timer

# 查看下次执行时间
systemctl list-timers | grep zfs
```


### 验证/games目录权限 {#验证-games目录权限}

```sh
# 检查/games目录权限
ls -la /games
id  # 确认当前用户ID

# 测试写入权限（假设当前用户是uid1000）
touch /games/test_file
rm /games/test_file
echo "✓ /games目录权限配置正确"
```


### 配置SSH公钥认证 {#配置ssh公钥认证}

```sh
# 在需要SSH连接的客户端机器上，复制公钥到服务器
ssh-copy-id username@server_ip

# 在服务器上配置SSH仅允许公钥登录
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config

# 启用SSH服务
sudo systemctl enable sshd
sudo systemctl restart sshd

echo "SSH公钥认证配置完成"
```


### 验证SMB共享 {#验证smb共享}

```sh
# 检查SMB服务状态
sudo systemctl status smb
sudo systemctl status nmb

# 验证ZFS SMB共享配置
zfs get sharesmb zroot/share

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
sudo pacman -Syu
```


### 安装yay和zrepl {#安装yay和zrepl}

```sh
# 安装yay-bin（AUR helper）
sudo pacman -S --needed git base-devel && git clone https://aur.archlinux.org/yay-bin.git && cd yay-bin && makepkg -si

# 使用yay安装zrepl
yay -S zrepl
```


### 配置zrepl自动快照系统 {#配置zrepl自动快照系统}

```sh
sudo mkdir -p /etc/zrepl
sudo tee /etc/zrepl/zrepl.yml << 'EOF'
global:
  logging:
    - type: stdout
      level: info
      format: human

jobs:
  - name: "system_snapshots"
    type: snap
    filesystems: {
      "zroot/ROOT<": true,
      "zroot/home<": true,
      "zroot/var<": true
    }
    snapshotting:
      type: periodic
      prefix: auto_
      interval: 1h
    pruning:
      keep:
        - type: last_n
          count: 24  # 保留最近24个小时快照

  - name: "daily_snapshots"
    type: snap
    filesystems: {
      "zroot/ROOT<": true,
      "zroot/home<": true,
      "zroot/var<": true
    }
    snapshotting:
      type: periodic
      prefix: daily_
      interval: 24h
    pruning:
      keep:
        - type: last_n
          count: 30   # 保留最近30天的每日快照

  - name: "weekly_snapshots"
    type: snap
    filesystems: {
      "zroot/ROOT<": true,
      "zroot/home<": true
    }
    snapshotting:
      type: periodic
      prefix: weekly_
      interval: 168h
    pruning:
      keep:
        - type: last_n
          count: 12   # 保留最近12周的每周快照

  - name: "monthly_snapshots"
    type: snap
    filesystems: {
      "zroot/ROOT<": true,
      "zroot/home<": true
    }
    snapshotting:
      type: periodic
      prefix: monthly_
      interval: 720h
    pruning:
      keep:
        - type: last_n
          count: 12   # 保留最近12个月的每月快照
EOF

# 启用并启动zrepl服务
sudo systemctl enable zrepl
sudo systemctl start zrepl
```


### 安装常用软件 {#安装常用软件}

```sh
# 开发工具
sudo pacman -S code firefox

# 系统工具
sudo pacman -S htop neofetch tree
```


### 测试hibernation {#测试hibernation}

```sh
# 检查swap状态
swapon --show
cat /proc/swaps

# 检查hibernation支持
cat /sys/power/disk

# 测试hibernation
sudo systemctl hibernate
```


### 配置zram压缩内存交换 {#配置zram压缩内存交换}

```sh
# 安装zram-generator（现代统一的zram管理工具）
sudo pacman -S zram-generator

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


## 第十部分：ZFS维护管理 {#第十部分-zfs维护管理}


### 常用ZFS命令 {#常用zfs命令}

```sh
# 查看存储池状态
zpool status

# 查看数据集
zfs list

# 手动创建快照
sudo zfs snapshot zroot/ROOT/root@manual-$(date +%Y%m%d)

# 列出快照
zfs list -t snapshot

# 删除快照
sudo zfs destroy zroot/ROOT/root@manual-20240101
```


### ZFS维护操作 {#zfs维护操作}

```sh
# 手动执行scrub
sudo zpool scrub zroot

# 查看scrub状态
sudo zpool status -v

# 停止正在进行的scrub
sudo zpool scrub -s zroot

# 手动执行trim
sudo zpool trim zroot

# 查看trim状态
sudo zpool status -t

# 停止正在进行的trim
sudo zpool trim -s zroot

# 查看ZFS ARC统计
cat /proc/spl/kstat/zfs/arcstats | grep -E "(hits|miss|size)"

# 查看压缩比
sudo zfs get compressratio zroot/ROOT/root

# 查看数据集使用情况
sudo zfs get used,available,referenced,compressratio
```


### 定时器管理 {#定时器管理}

```sh
# 查看所有ZFS相关定时器
systemctl list-timers | grep zfs

# 查看scrub定时器详细信息
systemctl status zfs-scrub-monthly@zroot.timer

# 查看trim定时器详细信息
systemctl status zfs-trim@zroot.timer

# 手动触发scrub服务
sudo systemctl start zfs-scrub-monthly@zroot.service

# 手动触发trim服务
sudo systemctl start zfs-trim@zroot.service

# 查看服务日志
sudo journalctl -u zfs-scrub-monthly@zroot.service
sudo journalctl -u zfs-trim@zroot.service
```


### 性能监控 {#性能监控}

```sh
# ZFS性能统计
zpool iostat 1

# 实时监控ZFS I/O
zpool iostat -v 1

# ARC缓存统计
cat /proc/spl/kstat/zfs/arcstats

# 数据集使用情况
zfs get used,available,referenced,compressratio

# zrepl状态
sudo systemctl status zrepl
sudo journalctl -u zrepl -f
```


### 快照恢复操作 {#快照恢复操作}

```sh
# 列出可用快照
zfs list -t snapshot | grep zroot/ROOT/root

# 回滚到特定快照（危险操作，会丢失快照后的更改）
sudo zfs rollback zroot/ROOT/root@pacman_pre_20240101_120000

# 克隆快照（安全操作）
sudo zfs clone zroot/ROOT/root@pacman_pre_20240101_120000 zroot/ROOT/recovery

# 挂载克隆的数据集
sudo zfs set mountpoint=/mnt/recovery zroot/ROOT/recovery
sudo zfs mount zroot/ROOT/recovery

# 比较当前系统与快照的差异
sudo zfs diff zroot/ROOT/root@pacman_pre_20240101_120000
```


## 第十一部分：故障排除 {#第十一部分-故障排除}


### 常见问题 {#常见问题}

**问题1：启动时找不到ZFS池**

```sh
# 手动导入池
sudo zpool import -f zroot

# 重新生成缓存文件
sudo zpool set cachefile=/etc/zfs/zpool.cache zroot
sudo systemctl restart zfs-import-cache
```

**问题2：ZFS模块未加载**

```sh
# 手动加载ZFS模块
sudo modprobe zfs

# 检查内核版本兼容性
uname -r
dkms status
```

**问题3：快照创建失败**

```sh
# 检查磁盘空间
df -h
zpool list

# 检查快照日志
tail -f /var/log/zfs-pacman-snapshots.log

# 手动清理快照
sudo zfs destroy zroot/ROOT/root@old_snapshot_name
```

**问题4：pacman钩子不工作**

```sh
# 检查钩子文件
ls -la /etc/pacman.d/hooks/
cat /etc/pacman.d/hooks/00-zfs-pre.hook

# 检查脚本权限
ls -la /usr/local/bin/zfs-pacman-snapshot

# 测试脚本
sudo /usr/local/bin/zfs-pacman-snapshot pre
```

**问题5：scrub或trim失败**

```sh
# 检查池状态
sudo zpool status -v

# 查看详细错误信息
sudo journalctl -u zfs-scrub@zroot.service -n 50

# 检查磁盘健康状态
sudo smartctl -a /dev/nvme0n1

# 手动停止并重新启动操作
sudo zpool scrub -s zroot
sudo zpool scrub zroot
```

**问题6：定时器未按预期执行**

```sh
# 检查定时器状态
systemctl status zfs-scrub-monthly@zroot.timer
systemctl status zfs-trim@zroot.timer

# 查看定时器日志
sudo journalctl -u zfs-scrub-monthly@zroot.timer

# 重新启用定时器
sudo systemctl disable zfs-scrub-monthly@zroot.timer
sudo systemctl enable zfs-scrub-monthly@zroot.timer
sudo systemctl start zfs-scrub-monthly@zroot.timer
```


### 紧急恢复 {#紧急恢复}

如果系统无法启动，可以使用制作的Live ISO：

1.  启动到Live环境
2.  配置archzfs仓库和密钥（参考第二部分2.4节）
3.  导入ZFS池：=zpool import -R /mnt zroot=
4.  挂载文件系统：=zfs mount zroot/ROOT/root &amp;&amp; zfs mount -a=
5.  挂载EFI分区：=mount /dev/nvme0n1p1 /mnt/boot=
6.  进入chroot：=arch-chroot /mnt=
7.  执行修复操作

**系统回滚示例：**

```sh
# 在Live环境中回滚到pacman操作前的快照
zpool import -R /mnt zroot
zfs rollback zroot/ROOT/root@pacman_pre_YYYYMMDD_HHMMSS
zfs mount zroot/ROOT/root
mount /dev/nvme0n1p1 /mnt/boot
arch-chroot /mnt
grub-mkconfig -o /boot/grub/grub.cfg
exit
reboot
```


## 常用命令速查 {#常用命令速查}

```sh
# ZFS管理
sudo zpool status         # 查看存储池状态
sudo zfs list            # 查看所有数据集
sudo zpool scrub zroot   # 手动执行scrub
sudo zpool trim zroot    # 手动执行trim
sudo systemctl status zrepl  # 查看zrepl状态

# 定时器管理
systemctl list-timers | grep zfs     # 查看ZFS定时器
sudo systemctl start zfs-scrub-monthly@zroot.service   # 手动执行scrub
sudo systemctl start zfs-trim@zroot.service    # 手动执行trim

# Hibernation
sudo systemctl hibernate # 休眠系统

# zram管理
sudo zramctl                               # 查看zram设备状态
sudo systemctl status systemd-zram-setup@zram0.service  # 查看zram服务状态
cat /proc/sys/vm/swappiness               # 查看交换倾向性设置
swapon --show                             # 显示所有交换设备（包括zram）

# 快照管理
sudo /usr/local/bin/zfs-pacman-snapshot pre   # 手动创建pre快照
tail -f /var/log/zfs-pacman-snapshots.log     # 查看快照日志
zfs list -t snapshot | grep pacman            # 查看pacman快照

# 性能监控
zpool iostat 1           # 实时I/O统计
cat /proc/spl/kstat/zfs/arcstats | grep size  # ARC缓存大小
sudo zfs get compressratio zroot/ROOT/root    # 查看压缩比

# SMB共享管理
zfs get sharesmb zroot/share            # 查看ZFS SMB共享配置
sudo systemctl status smb               # 检查SMB服务状态
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
sudo zpool status -v     # 详细池状态（包含scrub信息）
sudo zpool status -t     # 查看trim状态
sudo journalctl -u zfs-scrub-monthly@zroot.service   # 查看scrub日志
sudo journalctl -u zfs-trim@zroot.service    # 查看trim日志
```


## 附录：休眠唤醒设备管理 {#附录-休眠唤醒设备管理}


### 配置休眠唤醒设备（仅电源键唤醒） {#配置休眠唤醒设备-仅电源键唤醒}

在某些系统上，键盘、鼠标等USB设备可能会意外唤醒休眠状态，导致电池耗尽。以下配置确保只有电源键能够唤醒系统。

```sh
# 查看当前唤醒设备状态
cat /proc/acpi/wakeup

# 禁用USB控制器唤醒功能
# 根据你的系统，可能需要禁用以下设备（示例）：
echo XHC0 | sudo tee /proc/acpi/wakeup  # USB3.0 控制器1
echo XHC1 | sudo tee /proc/acpi/wakeup  # USB3.0 控制器2
echo XHC2 | sudo tee /proc/acpi/wakeup  # USB3.0 控制器3
echo XH00 | sudo tee /proc/acpi/wakeup  # USB控制器

# 验证设置（disabled表示已禁用唤醒）
cat /proc/acpi/wakeup
```


### 测试休眠唤醒功能 {#测试休眠唤醒功能}

```sh
# 休眠系统
sudo systemctl hibernate

# 休眠后，尝试以下操作验证：
# 1. 按键盘任意键 - 应该无法唤醒
# 2. 移动鼠标 - 应该无法唤醒
# 3. 按电源键 - 应该能正常唤醒系统

# 唤醒后检查休眠日志
sudo journalctl -b | grep -i hibernate

# 检查唤醒设备状态是否保持
cat /proc/acpi/wakeup | grep -E "(XHC|XH00)"
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
loginctl enable-linger "${USER}"
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

-   rootful：/var/lib/containers/storage  （zroot/containers）
-   rootless：~/.local/share/containers/storage  （zroot/containers-${user_arch}）


### 创建系统服务持久化配置 {#创建系统服务持久化配置}

```sh
# 创建systemd服务来持久化唤醒设备配置
sudo tee /etc/systemd/system/disable-wakeup.service << 'EOF'
[Unit]
Description=Disable USB device wakeup for hibernation
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo XHC0 > /proc/acpi/wakeup; echo XHC1 > /proc/acpi/wakeup; echo XHC2 > /proc/acpi/wakeup; echo XH00 > /proc/acpi/wakeup'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

# 启用服务
sudo systemctl enable disable-wakeup.service
sudo systemctl start disable-wakeup.service

# 验证服务状态
sudo systemctl status disable-wakeup.service
```
