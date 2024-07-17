+++
title = "Installation of Arch Linux with btrfs and snapper 基于 btrfs 与 snapper 安装 Arch Linux"
author = ["wang1zhen"]
date = 2024-07-17T09:39:00+09:00
draft = false
+++

## 启动 Live ISO 并连接 Wi-Fi {#启动-live-iso-并连接-wi-fi}

1.  启动到 Arch Linux Live 环境。
2.  设置 root 密码：
    ```sh
    passwd
    ```

3.  使用 nmtui 连接 Wi-Fi：
    ```sh
    nmtui
    ```

4.  查看 IP 地址：
    ```sh
    ip addr show
    ```


## 安装并启用 OpenSSH 服务器 {#安装并启用-openssh-服务器}

1.  在 Live 环境中安装 OpenSSH 服务器：
    ```sh
    pacman -Sy
    pacman -S openssh
    ```

2.  启用并启动 SSH 服务：
    ```sh
    systemctl start sshd
    ```

3.  确认 SSH 服务正在运行：
    ```sh
    systemctl status sshd
    ```

4.  从远程机器 ssh 连接到 Arch Live ISO


## 分区硬盘 {#分区硬盘}

使用 sgdisk 进行分区:

假设硬盘是 /dev/nvme0n1：

```sh
sgdisk -Z /dev/nvme0n1  # 清除所有分区
sgdisk -n 1:0:+512M -t 1:ef00 /dev/nvme0n1  # 创建EFI分区
sgdisk -n 2:0:0 -t 2:8300 /dev/nvme0n1  # 创建剩余空间的btrfs分区
```


## 格式化分区 {#格式化分区}

1.  格式化 EFI 分区为 FAT32：
    ```sh
    mkfs.vfat -F32 /dev/nvme0n1p1
    ```

2.  格式化 btrfs 分区：
    ```sh
    mkfs.btrfs /dev/nvme0n1p2
    ```


## 创建并挂载 subvolume {#创建并挂载-subvolume}

1.  挂载 btrfs 分区：
    ```sh
    mount /dev/nvme0n1p2 /mnt
    ```

2.  创建 subvolume：
    ```sh
    btrfs subvolume create /mnt/@
    btrfs subvolume create /mnt/@home
    btrfs subvolume create /mnt/@snapshots
    btrfs subvolume create /mnt/@var_log
    btrfs subvolume create /mnt/@var_cache
    btrfs subvolume create /mnt/@var_tmp
    btrfs subvolume create /mnt/@swap
    ```

3.  卸载 btrfs 分区：
    ```sh
    umount /mnt
    ```

4.  重新挂载 subvolume：
    ```sh
    mount -o subvol=@,noatime,compress=zstd /dev/nvme0n1p2 /mnt
    mkdir -p /mnt/{home,.snapshots,var/log,var/cache,var/tmp,boot/efi,swap}
    mount -o subvol=@home,noatime,compress=zstd /dev/nvme0n1p2 /mnt/home
    mount -o subvol=@snapshots,noatime,compress=zstd /dev/nvme0n1p2 /mnt/.snapshots
    mount -o subvol=@var_log,noatime,compress=zstd /dev/nvme0n1p2 /mnt/var/log
    mount -o subvol=@var_cache,noatime,compress=zstd /dev/nvme0n1p2 /mnt/var/cache
    mount -o subvol=@var_tmp,noatime,compress=zstd /dev/nvme0n1p2 /mnt/var/tmp
    mount -o subvol=@swap,noatime /dev/nvme0n1p2 /mnt/swap
    mount /dev/nvme0n1p1 /mnt/boot/efi
    ```


## 创建 swapfile {#创建-swapfile}

1.  创建 swapfile：
    ```sh
    btrfs filesystem mkswapfile --size 32g --uuid clear /mnt/swap/swapfile
    ```

2.  启用 swapfile：
    ```sh
    chmod 600 /mnt/swap/swapfile
    mkswap /mnt/swap/swapfile
    swapon /mnt/swap/swapfile
    ```


## 安装基础系统 {#安装基础系统}

1.  安装基础系统：
    ```sh
    pacstrap /mnt base linux linux-firmware
    ```


## 配置系统 {#配置系统}

1.  生成 fstab：
    ```sh
    genfstab -U /mnt >> /mnt/etc/fstab
    ```

2.  检查 fstab：
    ```sh
    vim /mnt/etc/fstab
    ```

3.  切换到新系统的 chroot 环境：
    ```sh
    arch-chroot /mnt
    ```

4.  设置基本配置：
    ```sh
    echo "hostname" > /etc/hostname
    echo "127.0.0.1 localhost" > /etc/hosts
    echo "127.0.1.1 hostname.localdomain hostname" >> /etc/hosts

    ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
    hwclock --systohc

    pacman -S networkmanager grub efibootmgr btrfs-progs snapper vim
    ```

5.  配置 locale：
    ```sh
    vim /etc/locale.gen
    ```
    在文件中取消注释行：
    ```text
    en_US.UTF-8 UTF-8
    ```
    生成 locale：
    ```sh
    locale-gen
    ```
    设置系统的默认语言环境：
    ```sh
    echo "LANG=en_US.UTF-8" > /etc/locale.conf
    ```

6.  安装 ssh server
    ```sh
    pacman -S openssh
    systemctl enable sshd
    ```

7.  安装 KDE Plasma 桌面环境和 Wayland 支持：
    ```sh
    pacman -S plasma-meta sudo
    systemctl enable NetworkManager
    ```

8.  配置 grub：
    ```sh
    grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

9.  创建新用户并加入 sudo 组：
    ```sh
    useradd -m -G wheel username
    passwd username
    ```

10. 允许 sudo 组成员使用 sudo 命令：
    ```sh
    EDITOR=vim visudo
    ```
    在文件中添加以下行：
    ```text
    %wheel ALL=(ALL) NOPASSWD: ALL
    ```

11. 退出 chroot 环境并卸载临时文件系统：
    ```sh
    exit
    umount -R /mnt
    ```


## 完成安装并重启 {#完成安装并重启}

1.  重启系统以验证配置是否正确。
    ```sh
    reboot
    ```


## 配置 Snapper {#配置-snapper}

1.  sudo
    ```shell
    sudo -i
    ```

2.  启动进入新安装的 Arch 系统后，安装 Snapper 和相关包：
    ```sh
    pacman -S snapper snap-pac grub-btrfs snap-pac-grub
    ```

3.  初始化 Snapper 配置：
    ```sh
    umount /.snapshots
    rm -r /.snapshots
    snapper -c root create-config /
    btrfs subvolume delete /.snapshots
    mkdir /.snapshots
    mount -a
    chmod 750 /.snapshots
    snapper -c home create-config /home
    ```

4.  编辑 Snapper 配置文件 /etc/snapper/configs/root 和 /etc/snapper/configs/home，设置快照策略：
    ```sh
    vim /etc/snapper/configs/root
    ```
    在配置文件中，设置以下参数：
    ```text
    TIMELINE_CREATE="yes"
    TIMELINE_CLEANUP="yes"
    TIMELINE_MIN_AGE="1800"
    TIMELINE_LIMIT_HOURLY="24"
    TIMELINE_LIMIT_DAILY="7"
    TIMELINE_LIMIT_WEEKLY="4"
    TIMELINE_LIMIT_MONTHLY="3"
    ```
    类似地，编辑 /etc/snapper/configs/home：
    ```sh
    vim /etc/snapper/configs/home
    ```
    设置相同的快照策略：
    ```text
    TIMELINE_CREATE="yes"
    TIMELINE_CLEANUP="yes"
    TIMELINE_MIN_AGE="1800"
    TIMELINE_LIMIT_HOURLY="24"
    TIMELINE_LIMIT_DAILY="7"
    TIMELINE_LIMIT_WEEKLY="4"
    TIMELINE_LIMIT_MONTHLY="3"
    ```

5.  允许所有 sudo 组的成员管理 Snapper 快照：
    ```sh
    vim /etc/snapper/configs/root
    ```
    找到 ALLOW_USERS 和 ALLOW_GROUPS，修改为：
    ```text
    ALLOW_USERS=""
    ALLOW_GROUPS="sudo"
    ```
    类似地，编辑 /etc/snapper/configs/home：
    ```sh
    vim /etc/snapper/configs/home
    ```
    修改为：
    ```text
    ALLOW_USERS=""
    ALLOW_GROUPS="sudo"
    ```

6.  配置 grub-btrfs
    ```shell
    vim /etc/default/grub-btrfs/config
    ```
    修改：
    ```text
    GRUB_BTRFS_LIMIT="5000"
    ```

    1.  启用并启动 snapper-timeline 和 snapper-cleanup 定时任务：

    <!--listend-->

    ```sh
    systemctl enable snapper-timeline.timer
    systemctl start snapper-timeline.timer
    systemctl enable snapper-cleanup.timer
    systemctl start snapper-cleanup.timer
    ```

7.  手动生成一个 Snapper 快照：
    ```sh
    snapper -c root create --description "Initial snapshot"
    ```
