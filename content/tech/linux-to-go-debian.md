+++
title = "linux-to-go-debian"
author = ["wang1zhen"]
description = "利用 zfs boot menu 在 zfs 上 安装 debian linux， 并加密分区"
date = 2025-09-29T00:00:00+09:00
draft = false
+++

## 配置 Live 环境 {#配置-live-环境}


### 切换到 root shell {#切换到-root-shell}

```shell
sudo -i
```


### 定义系统 ID {#定义系统-id}

定义一个系统标识符，用作将要安装的文件系统的简短名称。

```shell
export ID=<id>
```


### 配置和更新 APT {#配置和更新-apt}

```shell
cat <<EOF > /etc/apt/sources.list
deb http://deb.debian.org/debian/ trixie main non-free-firmware contrib
deb-src http://deb.debian.org/debian/ trixie main non-free-firmware contrib
EOF
apt update
```

> **注意：** 将 `deb.debian.org` 替换为本地镜像源可能会获得更快的下载速度。如果要使用 HTTPS 传输，请确保已安装 `ca-certificates` 和 `apt-transport-https` 软件包，且您的镜像源具有有效证书；否则 apt 将拒绝使用该镜像源。


### 安装辅助工具 {#安装辅助工具}

```shell
apt install debootstrap gdisk dkms linux-headers-$(uname -r)
apt install zfsutils-linux arch-install-scripts
```


### 生成 `/etc/hostid` {#生成-etc-hostid}

```shell
zgenhostid -f
```


## 定义磁盘变量 {#定义磁盘变量}

为方便操作并减少出错可能性，设置环境变量来引用将在安装过程中配置的设备。

建议使用 `/dev/disk/by-id/` 路径来引用磁盘，这样可以确保磁盘标识的持久性和唯一性。

使用 `ls -l /dev/disk/by-id/` 查看可用的磁盘设备。

```shell
# 查看可用磁盘
ls -l /dev/disk/by-id/
```

对于许多用户来说，最方便的做法是将引导文件（即 ZFSBootMenu 及负责启动它的加载器）放在将要存储 ZFS 池的同一磁盘上。


### 定义引导磁盘变量 {#定义引导磁盘变量}

```shell
export BOOT_DISK="/dev/disk/by-id/<boot-disk-id>"
export BOOT_PART="1"
export BOOT_DEVICE="${BOOT_DISK}-part${BOOT_PART}"
```


### 定义 ZFS 池磁盘变量 {#定义-zfs-池磁盘变量}

```shell
export POOL_DISK="/dev/disk/by-id/<pool-disk-id>"
export POOL_PART="2"
export POOL_DEVICE="${POOL_DISK}-part${POOL_PART}"
```

> **注意：** 如果引导分区和 ZFS 池在同一磁盘上，=BOOT_DISK= 和 `POOL_DISK` 应该相同。如果使用独立的引导设备，请将它们设置为不同的磁盘 ID，并相应调整 =POOL_PART=。


## 磁盘准备 {#磁盘准备}


### 擦除分区 {#擦除分区}

```bash
zpool labelclear -f "$POOL_DISK"

wipefs -a "$POOL_DISK"
wipefs -a "$BOOT_DISK"

sgdisk --zap-all "$POOL_DISK"
sgdisk --zap-all "$BOOT_DISK"
```


### 创建 EFI 引导分区 {#创建-efi-引导分区}

```bash
sgdisk -n "${BOOT_PART}:1m:+512m" -t "${BOOT_PART}:ef00" "$BOOT_DISK"
```


### 创建 zpool 分区 {#创建-zpool-分区}

```bash
sgdisk -n "${POOL_PART}:0:-10m" -t "${POOL_PART}:bf00" "$POOL_DISK"
```


## ZFS 池创建 {#zfs-池创建}


### 创建 zpool {#创建-zpool}

```bash
zpool create -f -o ashift=12 \
      -O compression=zstd \
      -O acltype=posixacl \
      -O xattr=sa \
      -O relatime=on \
      -O encryption=aes-256-gcm \
      -O keylocation=prompt \
      -O keyformat=passphrase \
      -o autotrim=on \
      -m none zroot "$POOL_DEVICE"
```

执行此命令后，系统会提示您输入加密密码。

> **注意：** 主要选项说明：
>
> -   `compression=zstd` - 使用 zstd 压缩算法，提供更好的压缩率。
> -   `encryption=aes-256-gcm` - 您可以根据需要调整算法，但这在现代 x86_64 硬件上可能是性能最好的。
> -   `keylocation=prompt` - 设置为在需要时提示输入密码，而不是从文件读取。
> -   `keyformat=passphrase` - 密钥格式为密码短语。您的密码短语必须是可以在键盘上输入的内容，因为您需要在启动时输入它来解锁池。


### 创建初始文件系统 {#创建初始文件系统}

```shell
zfs create -o mountpoint=none zroot/ROOT
zfs create -o mountpoint=/ -o canmount=noauto zroot/ROOT/${ID}
zfs create -o mountpoint=/home zroot/home

zpool set bootfs=zroot/ROOT/${ID} zroot
```

> **注意：** 重要的是在任何 `mountpoint=/` 的文件系统上设置属性 `canmount=noauto=（即在您创建的任何其他引导环境上）。如果没有此属性，操作系统将尝试自动挂载所有 ZFS 文件系统，当多个文件系统尝试挂载到 =/` 时会失败；这将阻止系统启动。不需要自动挂载 `/` 因为根文件系统在引导过程中会被显式挂载。
>
> 还要注意，与许多 ZFS 属性不同，=canmount= 不可继承。因此，在 `zroot/ROOT` 上设置 `canmount=noauto` 是不够的，因为您随后创建的任何引导环境都将默认为 =canmount=on=。必须在您创建的每个引导环境上显式设置 =canmount=noauto=。


### 导出，然后使用临时挂载点 `/mnt` 重新导入 {#导出-然后使用临时挂载点-mnt-重新导入}

```shell
zpool export zroot
zpool import -N -R /mnt zroot
zfs load-key -L prompt zroot
```

```shell
zfs mount zroot/ROOT/${ID}
zfs mount zroot/home
```


### 验证所有内容是否正确挂载 {#验证所有内容是否正确挂载}

```shell
# mount | grep mnt
zroot/ROOT/debian on /mnt type zfs (rw,relatime,xattr,posixacl)
zroot/home on /mnt/home type zfs (rw,relatime,xattr,posixacl)
```


### 更新设备符号链接 {#更新设备符号链接}

```shell
udevadm trigger
```


## 安装 Debian {#安装-debian}

```bash
debootstrap trixie /mnt
```


### 将文件复制到新安装中 {#将文件复制到新安装中}

```bash
cp /etc/hostid /mnt/etc/hostid
cp /etc/resolv.conf /mnt/etc/
```


### 创建并挂载 EFI 分区 {#创建并挂载-efi-分区}

```bash
mkfs.vfat -F32 "$BOOT_DEVICE"
mkdir -p /mnt/boot/efi
mount "$BOOT_DEVICE" /mnt/boot/efi
```


### 生成 fstab {#生成-fstab}

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```


### Chroot 进入新系统 {#chroot-进入新系统}

```bash
arch-chroot /mnt
```


## Debian 基本配置 {#debian-基本配置}


### 设置主机名 {#设置主机名}

```bash
echo 'YOURHOSTNAME' > /etc/hostname
echo -e '127.0.1.1\tYOURHOSTNAME' >> /etc/hosts
```


### 设置 root 密码 {#设置-root-密码}

```bash
passwd
```


### 添加用户 {#添加用户}

```bash
useradd -m -G sudo -s /bin/bash <username>
passwd <username>
```

> **注意：** 将 `<username>` 替换为您想要创建的用户名。=-G sudo= 参数将用户添加到 sudo 组，使其能够使用管理员权限。


### 配置 `apt` 源 {#配置-apt-源}

```bash
cat <<EOF > /etc/apt/sources.list
deb http://deb.debian.org/debian/ trixie main non-free-firmware contrib
deb-src http://deb.debian.org/debian/ trixie main non-free-firmware contrib

deb http://deb.debian.org/debian-security trixie-security main non-free-firmware contrib
deb-src http://deb.debian.org/debian-security/ trixie-security main non-free-firmware contrib

# trixie-updates, to get updates before a point release is made;
deb http://deb.debian.org/debian trixie-updates main non-free-firmware contrib
deb-src http://deb.debian.org/debian trixie-updates main non-free-firmware contrib
EOF
```


### 更新仓库缓存 {#更新仓库缓存}

```bash
apt update
```


### 安装额外的基础软件包 {#安装额外的基础软件包}

```bash
apt install locales keyboard-configuration console-setup
```


### 配置本地化 {#配置本地化}

编辑 `/etc/locale.gen` 取消注释需要的语言（至少包括 =en_US.UTF-8=），然后生成 locale：

```bash
locale-gen
```

设置系统默认语言：

```bash
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```


### 配置时区 {#配置时区}

```bash
ln -sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime
```


### 配置键盘和控制台 {#配置键盘和控制台}

```bash
dpkg-reconfigure keyboard-configuration console-setup
```


## ZFS 配置 {#zfs-配置}


### 安装必需的软件包 {#安装必需的软件包}

```shell
apt install linux-headers-amd64 linux-image-amd64 zfs-initramfs dosfstools
apt install amd64-microcode intel-microcode
echo "REMAKE_INITRD=yes" > /etc/dkms/zfs.conf
```


### 启用 systemd ZFS 服务 {#启用-systemd-zfs-服务}

```shell
systemctl enable zfs.target
systemctl enable zfs-import-cache
systemctl enable zfs-mount
systemctl enable zfs-import.target
```


### 重建 initramfs {#重建-initramfs}

```shell
update-initramfs -c -k all
```


## 安装和配置 ZFSBootMenu {#安装和配置-zfsbootmenu}


### 在数据集上设置 ZFSBootMenu 属性 {#在数据集上设置-zfsbootmenu-属性}

分配引导最终内核时使用的命令行参数。因为 ZFS 属性是可继承的，所以将通用属性分配给 `ROOT` 数据集，以便所有子数据集默认继承通用参数。

```shell
zfs set org.zfsbootmenu:commandline="quiet" zroot/ROOT
```


### 安装 ZFSBootMenu {#安装-zfsbootmenu}

```bash
apt install curl
```

获取预构建的 ZFSBootMenu EFI 可执行文件，并将其保存到 EFI 系统分区：

```shell
mkdir -p /boot/efi/EFI/ZBM
curl -o /boot/efi/EFI/ZBM/VMLINUZ.EFI -L https://get.zfsbootmenu.org/efi
cp /boot/efi/EFI/ZBM/VMLINUZ.EFI /boot/efi/EFI/ZBM/VMLINUZ-BACKUP.EFI
```


### 更新 ZFSBootMenu {#更新-zfsbootmenu}

当需要更新 ZFSBootMenu 到最新版本时，执行以下步骤：

```shell
# 备份当前版本
cp /boot/efi/EFI/ZBM/VMLINUZ.EFI /boot/efi/EFI/ZBM/VMLINUZ-BACKUP.EFI

# 下载最新版本
curl -o /boot/efi/EFI/ZBM/VMLINUZ.EFI -L https://get.zfsbootmenu.org/efi
```

> **注意：** 如果更新后出现问题，可以通过 EFI 引导菜单选择 "ZFSBootMenu (Backup)" 使用备份版本启动。


### 配置 EFI 引导条目 {#配置-efi-引导条目}

检查 efivarfs 是否已挂载，如未挂载则手动挂载：

```shell
mount | grep efivarfs || mount -t efivarfs efivarfs /sys/firmware/efi/efivars
```

安装 efibootmgr 并创建引导条目：

```bash
apt install efibootmgr
```

```bash
efibootmgr -c -d "$BOOT_DISK" -p "$BOOT_PART" \
           -L "ZFSBootMenu (Backup)" \
           -l '\EFI\ZBM\VMLINUZ-BACKUP.EFI'

efibootmgr -c -d "$BOOT_DISK" -p "$BOOT_PART" \
           -L "ZFSBootMenu" \
           -l '\EFI\ZBM\VMLINUZ.EFI'
```

> **注意：** 某些系统可能存在 EFI 引导条目问题。如果您重新启动后在 EFI 选择屏幕（通常在 POST 期间通过 F 键访问）中看不到上述条目，则可能需要使用众所周知的 EFI 文件名。有关此问题的帮助，请参阅 Portable EFI 文档。
>
> 有关配置 ZFSBootMenu 引导时行为的详细信息，请参阅 `zbm-kcl.8` 和 =zfsbootmenu.7=。


## 准备首次启动 {#准备首次启动}


### 退出 chroot，卸载所有内容 {#退出-chroot-卸载所有内容}

```shell
exit
```

```shell
umount -n -R /mnt
```


### 导出 zpool 并重启 {#导出-zpool-并重启}

```shell
zpool export zroot
reboot
```
