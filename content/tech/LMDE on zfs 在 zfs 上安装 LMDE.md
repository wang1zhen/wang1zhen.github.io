+++
title = "LMDE on zfs 在 zfs 上安装 LMDE"
author = ["wang1zhen"]
description = "分步骤在 ZFS 阵列上安装 LMDE 并配置系统。"
date = 2025-10-17T00:00:00+09:00
draft = false
+++

## 第一部分：准备安装介质 {#第一部分-准备安装介质}


### 下载LMDE Live镜像 {#下载lmde-live镜像}

-   从 Linux Mint 官网获取 LMDE 最新 ISO（例如 LMDE 7 Virginia 基于 Debian 13/trixie）。


### 制作启动U盘 {#制作启动u盘}

-   可使用 balenaEtcher、Rufus 或 \`dd\` 写入 U 盘。


## 第二部分：启动并配置网络环境 {#第二部分-启动并配置网络环境}


### 启动到Live环境 {#启动到live环境}

-   从 U 盘启动，进入 LMDE Live 桌面。


### 配置网络连接 {#配置网络连接}

-   使用 NetworkManager 图形界面或 \`nmcli\` 连接网络。


### 设置系统时间 {#设置系统时间}

```sh
sudo timedatectl set-ntp true
timedatectl status
```


### 启用SSH（可选） {#启用ssh-可选}

```sh
sudo apt update && sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```


## 第三部分：磁盘分区 {#第三部分-磁盘分区}


### 安装必要工具 {#安装必要工具}

```sh
sudo apt update
sudo apt install -y gdisk dosfstools parted arch-install-scripts
```


### 识别目标磁盘 {#识别目标磁盘}

```sh
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
```


### 设置磁盘变量 {#设置磁盘变量}

```sh
# 使用 by-id 更稳健，以下为示例占位，请替换为实际 by-id 设备
DISK1=/dev/disk/by-id/nvme-EXAMPLE_DEVICE_ID
echo "DISK1: $DISK1"
```


### 创建GPT分区表 {#创建gpt分区表}

```sh
# 清空并新建分区：EFI(1G) + SWAP(16G) + ZFS(余下)
sudo sgdisk --zap-all "$DISK1"
sudo sgdisk -n 1:1M:+1G  -t 1:EF00 "$DISK1"   # EFI
sudo sgdisk -n 2:0:+16G  -t 2:8200 "$DISK1"   # SWAP
sudo sgdisk -n 3:0:0     -t 3:BF00 "$DISK1"   # ZFS
sudo sgdisk -p "$DISK1"
```


### 格式化EFI和交换分区 {#格式化efi和交换分区}

```sh
sudo mkfs.fat -F32 ${DISK1}-part1
sudo mkswap ${DISK1}-part2
sudo swapon ${DISK1}-part2
```


### 查看磁盘ID {#查看磁盘id}

```sh
ls -lh /dev/disk/by-id/ | grep -E "nvme|ssd|ata"
```


## 第四部分：安装ZFS支持 {#第四部分-安装zfs支持}


### 添加ZFS仓库（可选） {#添加zfs仓库-可选}

-   trixie 默认仓库已有 ZFS；若需更高版本，可添加 backports：

<!--listend-->

```sh
echo "deb http://deb.debian.org/debian trixie-backports main contrib non-free-firmware non-free" | sudo tee -a /etc/apt/sources.list
sudo apt update
```


### 安装ZFS包 {#安装zfs包}

```sh
sudo apt install -y zfs-dkms zfsutils-linux
sudo modprobe zfs
```


## 第五部分：ZFS配置 {#第五部分-zfs配置}


### 创建ZFS存储池 {#创建zfs存储池}

```sh
# 变量
ZFS_PART=${DISK1}-part3
POOL=rpool
sudo zpool create -f \
     -o ashift=12 \
     -o autotrim=on \
     -O compression=zstd \
     -O atime=off \
     -O xattr=sa \
     -O acltype=posixacl \
     -O mountpoint=none \
     "$POOL" "$ZFS_PART"
```


### 创建ZFS数据集 {#创建zfs数据集}

```sh
sudo zfs create -o mountpoint=none $POOL/ROOT
sudo zfs create -o mountpoint=/ -o canmount=noauto $POOL/ROOT/lmde
sudo zfs create -o mountpoint=/home $POOL/home
sudo zfs create -o mountpoint=/var  $POOL/var
sudo zfs create -o mountpoint=/var/log $POOL/var/log
sudo zfs create -o mountpoint=/var/cache $POOL/var/cache
```


### 验证数据集创建 {#验证数据集创建}

```sh
zpool status
zfs list
```


### 设置ZFS缓存文件 {#设置zfs缓存文件}

```sh
sudo zpool set cachefile=/etc/zfs/zpool.cache $POOL
```


### 重新导入ZFS池 {#重新导入zfs池}

```sh
sudo zpool export $POOL
sudo zpool import -R /mnt $POOL
```


### 验证挂载 {#验证挂载}

```sh
sudo zfs mount $POOL/ROOT/lmde
sudo mkdir -p /mnt/{boot,boot/efi,home,var,var/log,var/cache}
sudo mount ${DISK1}-part1 /mnt/boot/efi
zfs list
```


## 第六部分：安装LMDE系统 {#第六部分-安装lmde系统}


### 挂载EFI分区作为/boot {#挂载efi分区作为-boot}

```sh
sudo mount ${DISK1}-part1 /mnt/boot/efi
```


### 安装基础系统 {#安装基础系统}

```sh
sudo apt install -y debootstrap
sudo debootstrap --arch=amd64 trixie /mnt http://deb.debian.org/debian
```


### 配置系统挂载点 {#配置系统挂载点}

-   使用 ZFS 自挂载，无需在 fstab 中为根写入条目。


## 第七部分：系统配置 {#第七部分-系统配置}


### 进入chroot环境 {#进入chroot环境}

```sh
sudo mount -t proc /proc /mnt/proc
sudo mount --rbind /sys /mnt/sys
sudo mount --rbind /dev /mnt/dev
sudo arch-chroot /mnt
```


### 配置基本系统信息 {#配置基本系统信息}

```sh
echo "lmde-zfs" > /etc/hostname
printf "127.0.0.1\tlocalhost\n127.0.1.1\tlmde-zfs\n" > /etc/hosts
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen && locale-gen
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime || true
```


### 配置包管理器 {#配置包管理器}

```sh
cat > /etc/apt/sources.list << 'EOF'
deb http://deb.debian.org/debian trixie main contrib non-free-firmware non-free
deb http://security.debian.org/debian-security trixie-security main contrib non-free-firmware non-free
deb http://deb.debian.org/debian trixie-updates main contrib non-free-firmware non-free
EOF
apt update
```


### 安装必要软件包 {#安装必要软件包}

```sh
apt install -y linux-image-amd64 systemd-sysv zfs-initramfs zfsutils-linux \
    grub-efi-amd64 shim-signed sudo vim less network-manager
```


### 安装桌面环境（可选） {#安装桌面环境-可选}

-   如需 Mint 桌面，可在后续添加 LMDE 仓库与 keyring 后安装 \`mint-meta-cinnamon\`。


### 配置ZFS服务 {#配置zfs服务}

```sh
systemctl enable zfs-import-cache.service
systemctl enable zfs-mount.service
systemctl enable zfs-zed.service
```


### 配置initramfs {#配置initramfs}

```sh
update-initramfs -u -k all
```


### 配置GRUB {#配置grub}

```sh
sed -i 's|^GRUB_CMDLINE_LINUX=.*|GRUB_CMDLINE_LINUX="root=ZFS=rpool/ROOT/lmde"|g' /etc/default/grub
update-grub
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=LMDE --recheck
```


### 设置root密码 {#设置root密码}

```sh
echo "root:changeme" | chpasswd
```


### 创建普通用户 {#创建普通用户}

```sh
user_new="YOUR_USERNAME"
adduser "$user_new"
usermod -aG sudo "$user_new"
```


### 创建用户缓存和容器数据集 {#创建用户缓存和容器数据集}

```sh
zfs create -o mountpoint=/home/${user_new}/.cache rpool/cache-${user_new}
zfs create -o mountpoint=/home/${user_new}/.local/share/containers rpool/containers-${user_new}
```


### 重新挂载数据集 {#重新挂载数据集}

```sh
exit   # 退出 chroot
zfs mount -a
arch-chroot /mnt
```


### 设置用户目录权限 {#设置用户目录权限}

```sh
chown 1000:1000 "/home/${user_new}/.cache" \
      "/home/${user_new}/.local/share/containers"
chmod 755 "/home/${user_new}/.cache" "/home/${user_new}/.local/share/containers"
```


### 配置SMB共享 {#配置smb共享}

```sh
apt install -y samba
mkdir -p /share
chown 1000:1000 /share && chmod 700 /share
cat > /etc/samba/smb.conf << 'EOF'
[global]
    workgroup = WORKGROUP
    security = user
    map to guest = never
    server string = LMDE ZFS Server

[share]
    path = /share
    guest ok = no
    read only = no
    valid users = YOUR_USERNAME
    comment = Private ZFS Share
    create mask = 0660
    directory mask = 0770
EOF
smbpasswd -a ${user_new}
testparm -s
systemctl enable smbd
systemctl disable nmbd || true
```


### 启用基本服务 {#启用基本服务}

```sh
systemctl enable NetworkManager
systemctl enable systemd-resolved || true
systemctl enable ssh || true
```


## 第八部分：ZFS定期维护配置 {#第八部分-zfs定期维护配置}


### 配置ZFS定期维护服务 {#配置zfs定期维护服务}

```sh
# zpool trim 定制单元与定时器（与 Debian 同步）
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

systemctl enable zfs-scrub-weekly@rpool.timer
systemctl enable zfs-trim@rpool.timer
```


## 第九部分：完成安装 {#第九部分-完成安装}


### 退出chroot并清理 {#退出chroot并清理}

```sh
exit
umount /mnt/boot || true
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
sudo zpool status
sudo zfs list
df -h
mount | grep zfs
systemctl status zfs.target
systemctl status NetworkManager
```


### 启动ZFS定期维护服务 {#启动zfs定期维护服务}

```sh
sudo systemctl start zfs-scrub-weekly@rpool.timer
sudo systemctl start zfs-trim@rpool.timer
systemctl status zfs-scrub-weekly@rpool.timer
systemctl status zfs-trim@rpool.timer
systemctl list-timers | grep zfs
```


### 验证SMB共享 {#验证smb共享}

```sh
sudo systemctl status smbd
sudo testparm -s
smbclient -L localhost -U ${USER}
ls -la /share
```


### SMB用户管理 {#smb用户管理}

```sh
sudo pdbedit -L
sudo pdbedit -L -v -u ${USER}
sudo smbpasswd -a new_username
sudo smbpasswd ${USER}
sudo smbpasswd -d ${USER}
sudo smbpasswd -e ${USER}
sudo smbpasswd -x ${USER}
sudo smbstatus
```


### 系统更新 {#系统更新}

```sh
sudo apt update && sudo apt upgrade -y
```


### 安装 znapzend 自动快照系统 {#安装-znapzend-自动快照系统}

```sh
sudo apt install -y znapzend
```


### 配置 znapzend 自动快照 {#配置-znapzend-自动快照}

```sh
# 保留=>间隔：1d=>1h, 7d=>1d, 4w=>1w（无需递归）
sudo znapzendzetup create --tsformat='znapzend-%Y-%m-%d-%H%M%S' SRC '1d=>1h,7d=>1d,4w=>1w' rpool/ROOT/lmde
sudo znapzendzetup create --tsformat='znapzend-%Y-%m-%d-%H%M%S' SRC '1d=>1h,7d=>1d,4w=>1w' rpool/home

sudo systemctl enable znapzend
sudo systemctl start znapzend
sudo systemctl kill -s HUP znapzend || true
sudo znapzendzetup list
```


## 第十一部分：创建APT快照管理系统 {#第十一部分-创建apt快照管理系统}


### 创建快照管理脚本 {#创建快照管理脚本}

```sh
sudo tee /usr/local/bin/zfs-apt-snapshot << 'EOF'
#!/usr/bin/env bash
# Create and prune ZFS snapshots around apt transactions.
# Snapshots are named: apt_{pre|post}_YYYYmmdd_HHMMSS

set -uo pipefail

SNAPSHOT_PREFIX="apt"
MAX_SNAPSHOTS=50
DATASETS=("rpool/ROOT/lmde" "rpool/home")
LOG_FILE="/var/log/zfs-apt-snapshots.log"
TIMESTAMP="$(date +%Y%m%d_%H%M%S)"

log() { local msg="$1"; mkdir -p "$(dirname "$LOG_FILE")" 2>/dev/null || true; printf '[%s] %s\n' "$(date '+%F %T')" "$msg" >>"$LOG_FILE"; }
require_cmd() { command -v "$1" >/dev/null 2>&1 || { log "missing: $1"; return 1; }; }
acquire_lock() { exec 9>/run/zfs-apt-snapshot.lock || exec 9>/tmp/zfs-apt-snapshot.lock; flock -n 9 || { log "another instance"; return 1; }; }

create_snapshot() {
  local phase="$1"; local snap="${SNAPSHOT_PREFIX}_${phase}_${TIMESTAMP}"; log "creating $phase: $snap"
  for ds in "${DATASETS[@]}"; do
    if zfs list -H -o name "$ds" >/dev/null 2>&1; then
      zfs snapshot "${ds}@${snap}" >/dev/null 2>&1 && log "created: ${ds}@${snap}" || log "error: ${ds}@${snap}"
    else log "missing dataset: $ds"; fi
  done
}

cleanup_snapshots() {
  log "pruning old apt snapshots (keep newest ${MAX_SNAPSHOTS})"
  for ds in "${DATASETS[@]}"; do
    zfs list -H -o name "$ds" >/dev/null 2>&1 || continue
    mapfile -t snaps < <(zfs list -H -t snapshot -o name -S creation -r "$ds" 2>/dev/null | awk -v ds="$ds" -F'@' '$1==ds && $2 ~ /^apt_/ {print $0}')
    local count=${#snaps[@]}; (( count<=MAX_SNAPSHOTS )) && { log "no prune: $ds ($count)"; continue; }
    for ((i=MAX_SNAPSHOTS;i<count;i++)); do zfs destroy "${snaps[$i]}" >/dev/null 2>&1 && log "destroyed: ${snaps[$i]}" || log "destroy failed: ${snaps[$i]}"; done
  done
}

main() {
  require_cmd zfs || return 0; acquire_lock || return 0
  case "${1:-}" in
    pre) create_snapshot pre;;
    post) create_snapshot post; cleanup_snapshots;;
    *) echo "Usage: $0 {pre|post}";;
  esac
  return 0
}

main "$@" || true; exit 0
EOF
sudo chmod +x /usr/local/bin/zfs-apt-snapshot
```


### 创建APT钩子 {#创建apt钩子}

```sh
sudo tee /etc/apt/apt.conf.d/00-zfs-snapshot-pre << 'EOF'
DPkg::Pre-Invoke { "/usr/local/bin/zfs-apt-snapshot pre"; };
EOF
sudo tee /etc/apt/apt.conf.d/99-zfs-snapshot-post << 'EOF'
DPkg::Post-Invoke { "/usr/local/bin/zfs-apt-snapshot post"; };
EOF
```


### 测试快照系统 {#测试快照系统}

```sh
sudo /usr/local/bin/zfs-apt-snapshot pre
sleep 2
sudo /usr/local/bin/zfs-apt-snapshot post
zfs list -t snapshot | grep apt || echo "no apt snapshots"
```


## 第十二部分：ZFS维护管理 {#第十二部分-zfs维护管理}


### 常用ZFS命令 {#常用zfs命令}

```sh
zpool status
zfs list
sudo zfs snapshot rpool/ROOT/lmde@manual-$(date +%Y%m%d)
zfs list -t snapshot
sudo zfs destroy rpool/ROOT/lmde@manual-20240101 || true
```


### 磁盘健康监控 {#磁盘健康监控}

```sh
zpool status -v rpool
zpool iostat -v rpool
sudo smartctl -a $DISK1
```


### 性能监控 {#性能监控}

```sh
zpool iostat 1
zpool iostat -v 1
zfs get used,available,referenced,compressratio
```


## 第十三部分：故障排除 {#第十三部分-故障排除}


### 常见问题 {#常见问题}

**问题1：启动时找不到ZFS池**

```sh
sudo zpool import -f rpool
sudo zpool set cachefile=/etc/zfs/zpool.cache rpool
```

**问题2：磁盘故障处理**

```sh
sudo zpool status -v
sudo journalctl -u zfs-import-cache.service -n 50
sudo smartctl -a ${DISK1}
```


### 紧急恢复 {#紧急恢复}

1.  用 LMDE/Debian Live 启动
2.  \`apt update &amp;&amp; apt install -y zfsutils-linux\`
3.  \`zpool import -R /mnt rpool\`
4.  \`zfs mount rpool/ROOT/lmde &amp;&amp; zfs mount -a\`
5.  \`mount ${DISK1}-part1 /mnt/boot\`
6.  \`arch-chroot /mnt\`


## 常用命令速查 {#常用命令速查}

```sh
# ZFS管理
sudo zpool status
sudo zfs list
sudo zpool scrub rpool
sudo zpool trim rpool

# 磁盘健康
sudo zpool status -v rpool
zpool iostat -v rpool
sudo smartctl -a $DISK1

# 定时器管理
systemctl list-timers | grep zfs
sudo systemctl start zfs-scrub-weekly@rpool.service
sudo systemctl start zfs-trim@rpool.service

# 快照管理
sudo zfs snapshot rpool/ROOT/lmde@backup-$(date +%Y%m%d)
zfs list -t snapshot
sudo /usr/local/bin/zfs-apt-snapshot pre
tail -f /var/log/zfs-apt-snapshots.log
zfs list -t snapshot | grep apt
sudo systemctl status znapzend
sudo journalctl -u znapzend -f

# 休眠/交换
swapon --show
sudo systemctl hibernate
cat /sys/power/state
free -h

# zram
sudo zramctl
sudo systemctl status systemd-zram-setup@zram0.service
cat /proc/sys/vm/swappiness
```


### 安装 Podman 与兼容层 {#安装-podman-与兼容层}

```sh
apt install -y podman podman-docker podman-compose
podman info
podman run --rm hello-world
```


## 附录：Podman 常用命令与日常维护 {#附录-podman-常用命令与日常维护}


### 基本信息与运行 {#基本信息与运行}

```sh
podman info
podman run --rm hello-world
```


### 镜像与容器管理 {#镜像与容器管理}

```sh
podman images
podman ps -a
podman pull alpine
podman run -d --name web -p 8080:80 nginx:alpine
podman logs -f web
podman exec -it web sh
podman stop web && podman rm web
```


### 清理与空间回收 {#清理与空间回收}

```sh
podman image prune -a
podman container prune
podman volume ls
podman volume prune
podman system df
podman system prune -a
```


### Quadlet 自启（简要） {#quadlet-自启-简要}

-   用户路径：\`~/.config/containers/systemd/\`
-   系统路径：\`/etc/containers/systemd/\`
-   常用步骤：写入 .container -&gt; \`systemctl --user daemon-reload\` -&gt; \`systemctl --user start &lt;name&gt;.service\` -&gt; \`loginctl enable-linger $USER\`（可选）


## 附录：NVIDIA显卡禁用配置 {#附录-nvidia显卡禁用配置}


### 禁用NVIDIA显卡（使用bbswitch） {#禁用nvidia显卡-使用bbswitch}

```sh
echo "blacklist nouveau" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
echo "options nouveau modeset=0" | sudo tee -a /etc/modprobe.d/blacklist-nouveau.conf
sudo update-initramfs -u
sudo reboot
sudo apt install -y bbswitch-dkms
sudo modprobe bbswitch
echo OFF | sudo tee /proc/acpi/bbswitch
echo "bbswitch" | sudo tee -a /etc/modules
echo "options bbswitch load_state=0 unload_state=0" | sudo tee /etc/modprobe.d/bbswitch.conf
```


### 验证NVIDIA显卡禁用状态 {#验证nvidia显卡禁用状态}

```sh
cat /proc/acpi/bbswitch
lspci | grep -i nvidia
lsmod | grep -E "(nouveau|nvidia|bbswitch)"
sudo powertop || true
```
