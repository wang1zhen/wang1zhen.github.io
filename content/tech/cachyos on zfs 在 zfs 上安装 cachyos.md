+++
title = "cachyos on zfs 在 zfs 上安装 cachyos"
author = ["wang1zhen"]
description = "基于 zfs 安装 cachyos"
date = 2026-03-08T00:00:00+09:00
draft = false
+++

本指南专为在单块 1TB 固态硬盘、32GB 内存环境下安装基于 ZFS 的 CachyOS 编写。旨在平衡 ZFS 的强大快照保护功能与 CachyOS 极致的游戏/办公性能。

在单块固态硬盘上使用 ZFS 的核心目的和权衡：

-   \*时光机快照（Snapshot）\*：瞬间备份与回滚，完美抵御系统更新滚挂或误删文件
-   \*透明压缩（Compression）\*：变相扩容 1TB 固态硬盘，延长闪存写入寿命
-   \*数据完整性校验（Checksum）\*：实时发现静默数据损坏（Bit-rot）
-   \*限制\*：单盘无法自我修复数据，轻微的性能损耗，建议保持 80% 容量红线


## 第一部分：准备安装介质 {#第一部分-准备安装介质}


### 下载 CachyOS ISO {#下载-cachyos-iso}

从 CachyOS 官方网站下载最新 ISO 镜像：

```sh
# 使用 wget 下载（替换为最新版本链接）
wget https://mirror.cachyos.org/ISO/cachyos-desktop-linux-250202.iso
```


### 创建启动U盘 {#创建启动u盘}

```sh
# 使用 dd 命令写入U盘（将 /dev/sdX 替换为实际的U盘设备）
sudo dd if=cachyos-desktop-linux-250202.iso of=/dev/sdX bs=4M status=progress

# 或使用 Ventoy 制作多系统启动盘
```


## 第二部分：使用 Calamares 图形界面分区 {#第二部分-使用-calamares-图形界面分区}


### 启动 CachyOS 安装程序 {#启动-cachyos-安装程序}


### 引导器选择 {#引导器选择}

在引导器选择页面：

-   选择 **Limine** 作为引导器
-   Limine 原生支持 ZFS，具备自动探测能力，可跨硬盘扫描 Windows Boot Manager


### 图形化分区配置 {#图形化分区配置}

在 Calamares 的分区步骤，选择 \*"手动分区（Manual Partitioning）"\*：

-   创建 Boot 分区：
    -   大小：\*8GB\*
    -   文件系统：=fat32=
    -   挂载点：=/boot=
    -   flag: boot
    -   说明：内核文件独立于 ZFS 存储池，便于后期回滚时对齐内核版本

-   创建 Swap 分区：
    -   大小：\*32GB\*（建议与内存大小相同或略大）
    -   文件系统：=linuxswap=
    -   说明：休眠功能需要物理 swap 分区，ZFS 官方不推荐休眠到 ZFS 卷

-   创建 ZFS 根分区：
    -   大小：\*剩余全部空间\*
    -   文件系统：=zfs=
    -   挂载点：=/=
    -   说明：CachyOS 会自动创建 `zpcachyos/ROOT/cos` 这种 Boot Environment 规范布局


### 完成安装 {#完成安装}

继续完成安装程序的其他步骤（用户创建、软件包选择等），等待安装完成。
安装完成后系统会自动重启。


## 第三部分：装机后的 ZFS 终极调优 {#第三部分-装机后的-zfs-终极调优}

系统安装完成后，必须进行以下关键调优，以释放性能并保护固态硬盘。


### 验证 ZFS 存储池状态 {#验证-zfs-存储池状态}

```sh
zpool status
zfs list
```


### 强制开启 ZSTD 透明压缩 {#强制开启-zstd-透明压缩}

对系统安装程序生成的核心数据集逐一强制设定 zstd：

```sh
sudo zfs set compression=zstd zpcachyos/ROOT/cos/root
sudo zfs set compression=zstd zpcachyos/ROOT/cos/home
sudo zfs set compression=zstd zpcachyos/ROOT/cos/varcache
sudo sudo zfs set compression=zstd zpcachyos/ROOT/cos/varlog
```

说明：

-   =compression=zstd=：使用 zstd 压缩算法，提供优秀的压缩比和性能
-   透明压缩可变相扩容 1TB 固态硬盘，并延长闪存写入寿命


### 为游戏库和下载目录设定专属 Recordsize {#为游戏库和下载目录设定专属-recordsize}

**千万不要修改全局或系统根目录的 recordsize！** 仅针对存放大型连续文件的数据集进行优化。

```sh
# 1. 创建独立的数据集，精准挂载到根目录下
sudo zfs create -o mountpoint=/games zpcachyos/ROOT/cos/games
sudo zfs create -o mountpoint=/downloads zpcachyos/ROOT/cos/downloads

# 2. 移交所有权（请将 ${user_new} 替换为实际用户名）
user_new="YOUR_USERNAME"
sudo chown ${user_new}:${user_new} /games
sudo chown ${user_new}:${user_new} /downloads

# 3. 设定 1MB 块大小并开启压缩
sudo zfs set recordsize=1M zpcachyos/ROOT/cos/games
sudo zfs set recordsize=1M zpcachyos/ROOT/cos/downloads
sudo zfs set compression=zstd zpcachyos/ROOT/cos/games
sudo zfs set compression=zstd zpcachyos/ROOT/cos/downloads
```

参数说明：

-   =recordsize=1M=：1MB 块大小，适合大型连续文件（游戏、下载）


### 隔离高频缓存目录 {#隔离高频缓存目录}

将用户缓存目录隔离到独立数据集，保护固态硬盘并精简快照。

```sh
user_new="YOUR_USERNAME"

# 备份现有缓存
mv /home/${user_new}/.cache /home/${user_new}/.cache_backup

# 创建缓存数据集
sudo zfs create -o mountpoint=/home/${user_new}/.cache zpcachyos/ROOT/cos/${user_new}_cache
sudo chown ${user_new}:${user_new} /home/${user_new}/.cache
sudo zfs set compression=zstd zpcachyos/ROOT/cos/${user_new}_cache

# 恢复缓存数据
mv /home/${user_new}/.cache_backup/* /home/${user_new}/.cache/
rm -rf /home/${user_new}/.cache_backup
```


### 全面隔离容器生态（Podman/Docker） {#全面隔离容器生态-podman-docker}

```sh
user_new="YOUR_USERNAME"

# Rootful 容器
sudo zfs create -o mountpoint=/var/lib/containers zpcachyos/ROOT/cos/rootful_containers
sudo zfs set compression=zstd zpcachyos/ROOT/cos/rootful_containers

# Rootless 容器
sudo zfs create -o mountpoint=/home/${user_new}/.local/share/containers zpcachyos/ROOT/cos/rootless_containers
sudo chown ${user_new}:${user_new} /home/${user_new}/.local/share/containers
sudo zfs set compression=zstd zpcachyos/ROOT/cos/rootless_containers
```


### 开启固态硬盘护航功能 {#开启固态硬盘护航功能}

```sh
# 启用自动 TRIM (一般已开启)
sudo zpool set autotrim=on zpcachyos

# 启用每月 scrub 定时器
sudo systemctl enable --now zfs-scrub-monthly@zpcachyos.timer
```

说明：

-   =autotrim=on=：自动启用 TRIM，保持 SSD 性能
-   =zfs-scrub-monthly=：每月数据完整性检查


### 验证底层黄金参数 {#验证底层黄金参数}

```sh
zpool get ashift zpcachyos
zfs get xattr,dnodesize,atime,relatime zpcachyos
```

理想的输出应显示：

-   =ashift=12=：针对 4K 扇区磁盘优化
-   =xattr=sa=：扩展属性存入系统属性，大幅提升权限读取性能
-   =dnodesize=auto=：自动调整 dnode 大小
-   `atime=off` 或 =relatime=on=：减少读文件时产生的无用写入


## 第四部分：系统配置与服务 {#第四部分-系统配置与服务}


### 验证 32GB Swap 的休眠配置 {#验证-32gb-swap-的休眠配置}

确认安装器已自动配置好以下两项：

```sh
# 检查 Limine 配置中的 resume 参数
grep resume /boot/limine.conf

# 检查 mkinitcpio.conf 中的 resume 钩子位置
grep "^HOOKS" /etc/mkinitcpio.conf
```


## 第五部分：配置自动化快照保护 {#第五部分-配置自动化快照保护}


### 安装 Sanoid {#安装-sanoid}

```sh
paru -S sanoid
```


### 配置 Sanoid {#配置-sanoid}

```sh
sudo tee /etc/sanoid/sanoid.conf << 'EOF'
[zpcachyos/ROOT/cos]
    use_template = production
    recursive = yes

[zpcachyos/ROOT/cos/varcache]
    use_template = ignore
[zpcachyos/ROOT/cos/varlog]
    use_template = ignore
[zpcachyos/ROOT/cos/games]
    use_template = ignore
[zpcachyos/ROOT/cos/downloads]
    use_template = ignore
[zpcachyos/ROOT/cos/wang1zhen_cache]
    use_template = ignore
[zpcachyos/ROOT/cos/rootless_containers]
    use_template = ignore
[zpcachyos/ROOT/cos/rootful_containers]
    use_template = ignore
[zpcachyos/ROOT/cos/share]
    use_template = ignore

[template_production]
    autosnap = yes
    autoprune = yes
    frequently = 0
    hourly = 24
    daily = 7
    weekly = 4
    monthly = 0
    yearly = 0

[template_ignore]
    autosnap = no
    autoprune = no
    monitor = no
EOF
```

说明：

-   =hourly = 24=：保留最近 24 小时的每小时快照
-   =daily = 7=：保留最近 7 天的每日快照
-   =weekly = 4=：保留最近 4 周的每周快照
-   缓存、游戏、下载、容器目录被排除在自动快照之外


### 启动 Sanoid 服务 {#启动-sanoid-服务}

```sh
sudo systemctl enable --now sanoid.timer
```


### 验证 Sanoid 配置 {#验证-sanoid-配置}

```sh
# 查看配置文件
sudo cat /etc/sanoid/sanoid.conf

# 检查定时器状态
systemctl status sanoid.timer
systemctl list-timers | grep sanoid

# 查看已创建的快照
zfs list -t snapshot
```


## 第六部分：游戏生态配置 {#第六部分-游戏生态配置}


### 安装游戏核心套件 {#安装游戏核心套件}

```sh
sudo pacman -S cachyos-gaming-applications
```


### 配置防火墙 {#配置防火墙}


## 第七部分：灾难恢复与回滚避坑 {#第七部分-灾难恢复与回滚避坑}


### /boot 隔离问题说明 {#boot-隔离问题说明}

由于 `/boot` 位于独立的 FAT32 分区（不受 ZFS 快照保护），而内核模块（=/usr/lib/modules=）存在于 ZFS 中。当你通过 ZFS 回滚系统时，FAT32 分区里的内核镜像仍然是更新后的版本，会导致\*"内核镜像"与"内核模块"版本不匹配\*，开机报 =Failed to load kernel modules=。


### 回滚救机指南 {#回滚救机指南}


#### 使用 Live USB 恢复 {#使用-live-usb-恢复}

前提步骤：使用 Live USB 原生 ZFS 挂载池并回滚快照。

```sh
# 1. 扫描并强制导入存储池，同时将所有的挂载点优雅重定向到 /mnt
zpool import -f -R /mnt zpcachyos

# 2. 强制回滚 root 数据集到健康的旧快照
# (-r: 连带销毁比此快照更新的坏快照，-f: 强制短暂卸载目标以便回滚)
zfs rollback -r -f zpcachyos/ROOT/cos/root@你的健康快照名

# 3. 单独挂载你的 FAT32 EFI 分区
mount /dev/nvme0n1p1 /mnt/boot

# 4. 进入 chroot 环境
arch-chroot /mnt
```


#### 情景 A：配置/软件问题导致滚挂（非内核问题） {#情景-a-配置-软件问题导致滚挂-非内核问题}

此时最新的内核本身是健康的，只需要让 `/boot` 里的内核与 ZFS 里的模块重新同步：

```sh
# 直接重新拉取并安装最新内核
pacman -S linux-cachyos linux-cachyos-headers
```


#### 情景 B：最新版内核存在 Bug 导致滚挂（硬核降级） {#情景-b-最新版内核存在-bug-导致滚挂-硬核降级}

此时\*绝对不能\*使用 `pacman -S=（否则又会装上那个坏内核）。由于 Sanoid 豁免了 =varcache` 数据集，本地缓存中依然保留着健康的旧版内核！

```sh
# 1. 查看本地缓存里的内核包，找到版本号匹配回滚日期的那个
ls -lh /var/cache/pacman/pkg/linux-cachyos*

# 2. 使用 pacman -U 直接从本地缓存强行安装旧版内核
pacman -U /var/cache/pacman/pkg/linux-cachyos-<版本号>.pkg.tar.zst \
       /var/cache/pacman/pkg/linux-cachyos-headers-<版本号>.pkg.tar.zst
```

完成上述操作后，执行 `exit` 退出 chroot，重启电脑。
