+++
title = "zfs 备份"
author = ["wang1zhen"]
description = "把 zfs 备份到本地的硬盘或是 ssh 远程服务器"
date = 2025-09-26T00:00:00+09:00
draft = false
+++

## 硬盘初始化 {#硬盘初始化}

1.  确认设备
    ```bash
    lsblk
    ```

2.  清除旧分区表（可选）
    ```bash
    wipefs -a /dev/sdb
    ```

3.  使用 gdisk 分区
    ```bash
    gdisk /dev/sdb
    ```

    -   输入 `o` → 新建空 GPT 表
    -   输入 `n` → 新建分区（整盘）
    -   类型代码：=BF00= (Solaris / ZFS)
    -   输入 `w` → 写入

    完成后得到 `/dev/sdb1`

4.  创建 ZFS 池
    ```bash
    zpool create -f backup /dev/sdb1
    ```

5.  优化设置
    ```bash
    zfs set mountpoint=/mnt/backup backup
    zfs set compression=zstd backup
    ```

6.  卸载/挂载池
    ```bash
    zpool export backup        # 卸载
    zpool import backup        # 重新挂载
    ```


## 本地移动硬盘备份 {#本地移动硬盘备份}

1.  创建快照
    ```bash
    zfs snapshot zroot/home@2025-09-25
    ```

2.  直接传输到 ZFS 移动硬盘
    ```bash
    zfs send zroot/home@2025-09-25 | zfs recv -F backup/home
    ```

3.  增量备份
    ```bash
    zfs send -i zroot/home@2025-09-01 zroot/home@2025-09-25 | zfs recv -F backup/home
    ```

4.  存成文件（目标盘非 ZFS 时）
    ```bash
    zfs send zroot/home@2025-09-25 > /mnt/usb/zroot_home_20250925.zfs
    ```
    恢复：
    ```bash
    zfs recv backup/home < /mnt/usb/zroot_home_20250925.zfs
    ```

5.  优化选项
    -   压缩存档：
        ```bash
        zfs send zroot/home@2025-09-25 | gzip > /mnt/usb/home_20250925.zfs.gz
        ```
    -   大文件传输稳定性（=mbuffer=）：
        ```bash
        zfs send zroot/home@2025-09-25 | mbuffer -s 128k -m 1G | zfs recv -F backup/home
        ```


## 远程 SSH 备份 {#远程-ssh-备份}

1.  完整快照传输
    ```bash
    zfs send zroot/home@2025-09-25 | ssh user@remote "zfs recv -F backup/home"
    ```

2.  增量快照传输
    ```bash
    zfs send -i zroot/home@2025-09-01 zroot/home@2025-09-25 | ssh user@remote "zfs recv -F backup/home"
    ```

3.  传输为文件
    ```bash
    zfs send zroot/home@2025-09-25 | ssh user@remote "cat > /mnt/usb/home_20250925.zfs"
    ```
    恢复：
    ```bash
    ssh user@remote "cat /mnt/usb/home_20250925.zfs" | zfs recv -F backup/home
    ```

4.  稳定传输（推荐 mbuffer）
    ```bash
    zfs send zroot/home@2025-09-25 | mbuffer -s 128k -m 1G | ssh user@remote "mbuffer -s 128k -m 1G | zfs recv -F backup/home"
    ```


## 注意事项 {#注意事项}

-   **-f** 会覆盖设备上原有数据，谨慎使用
-   快照命名建议带日期，例如 `@2025-09-25`
-   远程备份建议设置 SSH 免密，方便脚本化
