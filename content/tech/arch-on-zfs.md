+++
title = "arch on zfs 在 zfs 上安装 archlinux"
author = ["user name"]
date = 2025-09-04T23:26:00+09:00
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

# 创建Docker数据集 - 小文件优化
zfs create \
    -o mountpoint=/var/lib/docker \
    -o compression=zstd \
    -o recordsize=64K \
    -o atime=off \
    zroot/docker

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
zfs set compression=off zroot/tmp

# 优化recordsize设置
zfs set recordsize=128K zroot/ROOT/root
zfs set recordsize=128K zroot/home
zfs set recordsize=1M zroot/games
zfs set recordsize=64K zroot/docker

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
         zfs-dkms zfs-utils plasma-meta sddm
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
#!/bin/bash

# ZFS Pacman快照管理脚本
# 在每次pacman操作前后创建快照，最多保留50个pacman相关快照

set -euo pipefail

SNAPSHOT_PREFIX="pacman"
MAX_SNAPSHOTS=50
DATASETS=("zroot/ROOT/root" "zroot/var" "zroot/home")

# 获取当前时间戳
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# 日志函数
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> /var/log/zfs-pacman-snapshots.log
}

# 创建快照函数
create_snapshot() {
    local phase=$1
    local snapshot_name="${SNAPSHOT_PREFIX}_${phase}_${TIMESTAMP}"

    log "开始创建 ${phase} 快照: ${snapshot_name}"

    for dataset in "${DATASETS[@]}"; do
        if zfs list -H -o name "$dataset" >/dev/null 2>&1; then
            local full_snapshot_name="${dataset}@${snapshot_name}"
            if zfs snapshot "$full_snapshot_name"; then
                log "成功创建快照: $full_snapshot_name"
            else
                log "错误: 创建快照失败: $full_snapshot_name"
            fi
        else
            log "警告: 数据集不存在: $dataset"
        fi
    done

    log "完成创建 ${phase} 快照"
}

# 清理旧快照函数
cleanup_snapshots() {
    log "开始清理旧的pacman快照"

    for dataset in "${DATASETS[@]}"; do
        if ! zfs list -H -o name "$dataset" >/dev/null 2>&1; then
            continue
        fi

        # 获取所有pacman快照，按创建时间排序（最新的在前）
        local snapshots=($(zfs list -H -t snapshot -o name -s creation | grep "${dataset}@${SNAPSHOT_PREFIX}_" | head -n 100))
        local snapshot_count=${#snapshots[@]}

        if [ $snapshot_count -gt $MAX_SNAPSHOTS ]; then
            log "数据集 $dataset 有 $snapshot_count 个pacman快照，需要清理"

            # 删除超出数量限制的快照（保留最新的MAX_SNAPSHOTS个）
            local to_delete=$((snapshot_count - MAX_SNAPSHOTS))
            local deleted_count=0

            for ((i=$((snapshot_count-1)); i>=$((snapshot_count-to_delete)); i--)); do
                if zfs destroy "${snapshots[$i]}"; then
                    log "删除旧快照: ${snapshots[$i]}"
                    ((deleted_count++))
                else
                    log "错误: 删除快照失败: ${snapshots[$i]}"
                fi
            done

            log "数据集 $dataset 清理完成，删除了 $deleted_count 个旧快照"
        else
            log "数据集 $dataset 有 $snapshot_count 个pacman快照，无需清理"
        fi
    done

    log "快照清理完成"
}

# 主函数
main() {
    local phase=$1

    # 确保日志目录存在
    mkdir -p "$(dirname /var/log/zfs-pacman-snapshots.log)"

    case "$phase" in
        pre)
            create_snapshot "pre"
            ;;
        post)
            create_snapshot "post"
            cleanup_snapshots
            ;;
        *)
            echo "用法: $0 {pre|post}"
            echo "  pre  - 在pacman操作前创建快照"
            echo "  post - 在pacman操作后创建快照并清理旧快照"
            exit 1
            ;;
    esac
}

# 执行主函数
main "$@"
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

# 快照管理
sudo /usr/local/bin/zfs-pacman-snapshot pre   # 手动创建pre快照
tail -f /var/log/zfs-pacman-snapshots.log     # 查看快照日志
zfs list -t snapshot | grep pacman            # 查看pacman快照

# 性能监控
zpool iostat 1           # 实时I/O统计
cat /proc/spl/kstat/zfs/arcstats | grep size  # ARC缓存大小
sudo zfs get compressratio zroot/ROOT/root    # 查看压缩比

# 维护操作
sudo zpool status -v     # 详细池状态（包含scrub信息）
sudo zpool status -t     # 查看trim状态
sudo journalctl -u zfs-scrub-monthly@zroot.service   # 查看scrub日志
sudo journalctl -u zfs-trim@zroot.service    # 查看trim日志
```
