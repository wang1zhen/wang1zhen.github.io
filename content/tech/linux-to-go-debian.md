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
export ID=debian
```


### 配置和更新 APT {#配置和更新-apt}

```shell
cat <<EOF > /etc/apt/sources.list
deb http://deb.debian.org/debian/ trixie main non-free-firmware contrib non-free
deb-src http://deb.debian.org/debian/ trixie main non-free-firmware contrib non-free
EOF
apt update
```

> **注意：**
>
> -   已添加 `non-free` 源以支持 NVIDIA 驱动等专有软件
> -   将 `deb.debian.org` 替换为本地镜像源可能会获得更快的下载速度


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

> **重要：** 对于 Linux To Go 系统，\*\*必须使用\*\* `/dev/disk/by-id/` 路径来引用磁盘。这样可以确保在不同机器上插入 USB 设备时，磁盘标识保持持久性和唯一性，避免因设备顺序变化导致的启动问题。

使用 `ls -l /dev/disk/by-id/` 查看可用的磁盘设备，选择您的 USB 设备。

```shell
# 查看可用磁盘
ls -l /dev/disk/by-id/
```

> **提示：** USB 设备通常包含 `usb` 字样，例如：
>
> -   `usb-SanDisk_Extreme_SSD_xxx`
> -   `usb-Samsung_Portable_SSD_xxx`


### 定义引导磁盘变量 {#定义引导磁盘变量}

```shell
export BOOT_DISK="/dev/disk/by-id/<usb-device-id>"
export BOOT_PART="1"
export BOOT_DEVICE="${BOOT_DISK}-part${BOOT_PART}"
```


### 定义 ZFS 池磁盘变量 {#定义-zfs-池磁盘变量}

```shell
export POOL_DISK="/dev/disk/by-id/<usb-device-id>"
export POOL_PART="2"
export POOL_DEVICE="${POOL_DISK}-part${POOL_PART}"
```

> **注意：** 对于 Linux To Go，引导分区和 ZFS 池通常在同一个 USB 设备上，因此 `BOOT_DISK` 和 `POOL_DISK` 应该相同。


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
      -m none zroot-ltg "$POOL_DEVICE"
```

执行此命令后，系统会提示您输入加密密码。

> **注意：** 主要选项说明：
>
> -   `compression=zstd` - 使用 zstd 压缩算法，提供更好的压缩率
> -   `encryption=aes-256-gcm` - AES-256-GCM 加密算法
> -   `keylocation=prompt` - 启动时提示输入密码
> -   `keyformat=passphrase` - 密钥格式为密码短语
> -   `autotrim=on` - 启用 TRIM，对 SSD 类型的 USB 设备有益


### 创建初始文件系统 {#创建初始文件系统}

```shell
zfs create -o mountpoint=none zroot-ltg/ROOT
zfs create -o mountpoint=/ -o canmount=noauto zroot-ltg/ROOT/${ID}
zfs create -o mountpoint=/home zroot-ltg/home

zpool set bootfs=zroot-ltg/ROOT/${ID} zroot-ltg
```

> **注意：** 重要的是在任何 `mountpoint=/` 的文件系统上设置属性 `canmount=noauto=（即在您创建的任何其他引导环境上）。如果没有此属性，操作系统将尝试自动挂载所有 ZFS 文件系统，当多个文件系统尝试挂载到 =/` 时会失败；这将阻止系统启动。不需要自动挂载 `/` 因为根文件系统在引导过程中会被显式挂载。
>
> 还要注意，与许多 ZFS 属性不同，=canmount= 不可继承。因此，在 `zroot-ltg/ROOT` 上设置 `canmount=noauto` 是不够的，因为您随后创建的任何引导环境都将默认为 =canmount=on=。必须在您创建的每个引导环境上显式设置 =canmount=noauto=。


### 导出，然后使用临时挂载点 `/mnt` 重新导入 {#导出-然后使用临时挂载点-mnt-重新导入}

```shell
zpool export zroot-ltg
zpool import -N -R /mnt zroot-ltg
zfs load-key -L prompt zroot-ltg
```

```shell
zfs mount zroot-ltg/ROOT/${ID}
zfs mount zroot-ltg/home
```


### 验证所有内容是否正确挂载 {#验证所有内容是否正确挂载}

```shell
# mount | grep mnt
zroot-ltg/ROOT/debian on /mnt type zfs (rw,relatime,xattr,posixacl)
zroot-ltg/home on /mnt/home type zfs (rw,relatime,xattr,posixacl)
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
deb http://deb.debian.org/debian/ trixie main non-free-firmware contrib non-free
deb-src http://deb.debian.org/debian/ trixie main non-free-firmware contrib non-free

deb http://deb.debian.org/debian-security trixie-security main non-free-firmware contrib non-free
deb-src http://deb.debian.org/debian-security/ trixie-security main non-free-firmware contrib non-free

# trixie-updates, to get updates before a point release is made;
deb http://deb.debian.org/debian trixie-updates main non-free-firmware contrib non-free
deb-src http://deb.debian.org/debian trixie-updates main non-free-firmware contrib non-free
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

> **提示：** 将 `<Region>/<City>` 替换为您的时区，例如 `Asia/Shanghai` 或 =America/New_York=。


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


### 安装硬件固件和驱动 {#安装硬件固件和驱动}

为确保在不同机器上的兼容性，安装常见的硬件固件和驱动程序：

```shell
# 基础固件包
apt install firmware-linux firmware-linux-nonfree firmware-misc-nonfree

# Wi-Fi 和音频固件
apt install firmware-iwlwifi firmware-realtek firmware-atheros
apt install firmware-sof-signed

# Intel 显卡驱动
apt install intel-media-va-driver i965-va-driver mesa-vulkan-drivers

# AMD 显卡驱动
apt install firmware-amd-graphics mesa-vulkan-drivers libdrm-amdgpu1

# NVIDIA 显卡驱动（可选，如果需要）
apt install nvidia-driver firmware-nvidia-gsp
```

> **注意：**
>
> -   NVIDIA 驱动体积较大，如果不需要可以跳过
> -   这些包确保系统在大多数硬件上都能正常工作


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


## 安装桌面环境（可选） {#安装桌面环境-可选}


### 安装 Xfce 桌面环境 {#安装-xfce-桌面环境}

```bash
apt install xfce4 xfce4-goodies lightdm
```


### 安装网络管理器 {#安装网络管理器}

如果不安装完整桌面环境，至少安装 NetworkManager 以便管理网络连接：

```bash
apt install network-manager
systemctl enable NetworkManager
```

> **注意：** 如果安装了桌面环境，NetworkManager 通常会自动安装。如果只需要命令行系统，可以只安装 NetworkManager 而不安装桌面环境。


## 安装和配置 ZFSBootMenu {#安装和配置-zfsbootmenu}


### 在数据集上设置 ZFSBootMenu 属性 {#在数据集上设置-zfsbootmenu-属性}

分配引导最终内核时使用的命令行参数。因为 ZFS 属性是可继承的，所以将通用属性分配给 `ROOT` 数据集，以便所有子数据集默认继承通用参数。

```shell
zfs set org.zfsbootmenu:commandline="quiet" zroot-ltg/ROOT
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


### 配置 Portable EFI 启动 {#配置-portable-efi-启动}

为确保 USB 设备在不同机器上都能启动，需要将 ZFSBootMenu 复制到标准 EFI 启动位置：

```shell
mkdir -p /boot/efi/EFI/BOOT
cp /boot/efi/EFI/ZBM/VMLINUZ.EFI /boot/efi/EFI/BOOT/BOOTX64.EFI
```

> **重要：** 这一步对于 Linux To Go 至关重要。=/EFI/BOOT/BOOTX64.EFI= 是 UEFI 固件的默认启动路径，确保在任何支持 UEFI 的机器上都能识别和启动此 USB 设备，即使该机器的 NVRAM 中没有专门的启动条目。


### 更新 ZFSBootMenu {#更新-zfsbootmenu}

当需要更新 ZFSBootMenu 到最新版本时，执行以下步骤：

```shell
# 备份当前版本
cp /boot/efi/EFI/ZBM/VMLINUZ.EFI /boot/efi/EFI/ZBM/VMLINUZ-BACKUP.EFI

# 下载最新版本
curl -o /boot/efi/EFI/ZBM/VMLINUZ.EFI -L https://get.zfsbootmenu.org/efi

# 更新 Portable EFI 启动文件
cp /boot/efi/EFI/ZBM/VMLINUZ.EFI /boot/efi/EFI/BOOT/BOOTX64.EFI
```

> **注意：** 更新后记得同时更新 `/EFI/BOOT/BOOTX64.EFI` 文件。


### 配置 EFI 引导条目（可选） {#配置-efi-引导条目-可选}

对于支持的机器，可以创建专门的 EFI 启动条目以获得更好的启动体验：

```shell
mount | grep efivarfs || mount -t efivarfs efivarfs /sys/firmware/efi/efivars
```

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

> **注意：** 这一步是可选的。由于已配置 Portable EFI，即使不创建这些启动条目，系统也能在任何机器上启动。这些条目只会保存在当前机器的 NVRAM 中。


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
zpool export zroot-ltg
reboot
```

> **使用提示：**
>
> -   在其他机器上使用时，在 BIOS/UEFI 设置中选择从 USB 设备启动
> -   首次在新机器上启动时，需要输入 ZFS 加密密码
> -   某些机器可能需要在 BIOS 中禁用 Secure Boot


## 跨机器使用注意事项 {#跨机器使用注意事项}


### 网络配置 {#网络配置}

在新机器上首次启动时，网络配置会自动适配。如果使用 NetworkManager，它会自动检测和配置可用的网络接口。


### 显卡驱动 {#显卡驱动}

系统已安装多种显卡驱动，大多数情况下会自动选择合适的驱动。如果遇到显示问题：

-   Intel/AMD 集成显卡通常开箱即用
-   NVIDIA 独立显卡需要确保已安装 `nvidia-driver`


### 硬件时钟 {#硬件时钟}

在不同机器间切换时，可能需要手动同步时间：

```bash
timedatectl set-ntp true
```
