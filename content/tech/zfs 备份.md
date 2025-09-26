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
    sudo wipefs -a /dev/sda
    ```

3.  使用 gdisk 分区
    ```bash
    sudo gdisk /dev/sda
    ```

    -   输入 `o` → 新建空 GPT 表
    -   输入 `n` → 新建分区（整盘）
    -   类型代码：=BF00= (Solaris / ZFS)
    -   输入 `w` → 写入

    完成后得到 `/dev/sda1`

4.  创建 ZFS 池
    ```bash
    sudo zpool create -f backup /dev/sda1
    ```

5.  优化设置
    ```bash
    sudo zfs set mountpoint=/mnt/backup backup
    sudo zfs set compression=zstd backup
    ```

6.  卸载/挂载池
    ```bash
    sudo zpool export backup                    # 卸载
    sudo zpool import                            # 查看可导入的池
    sudo zpool import -d /dev/disk/by-id backup # 重新挂载（指定设备路径）
    ```

7.  检查池状态
    ```bash
    sudo zpool status backup
    sudo zfs list backup
    ```


## 本地移动硬盘备份 {#本地移动硬盘备份}

1.  创建快照
    ```bash
    sudo zfs snapshot zroot/home@2025-09-25
    ```

2.  传输前验证快照存在
    ```bash
    sudo zfs list -t snapshot | grep hourly
    ```

3.  直接传输到 ZFS 移动硬盘（带进度显示）
    ```bash
    sudo zfs send zroot/home@2025-09-25 | pv | sudo zfs recv -F backup/home
    ```

4.  增量备份（带进度显示）
    ```bash
    sudo zfs send -i zroot/home@2025-09-01 zroot/home@2025-09-25 | pv | sudo zfs recv -F backup/home
    ```

5.  存成文件（目标盘非 ZFS 时）
    ```bash
    sudo zfs send zroot/home@2025-09-25 | pv > /mnt/usb/zroot_home_20250925.zfs
    ```
    恢复：
    ```bash
    pv /mnt/usb/zroot_home_20250925.zfs | sudo zfs recv backup/home
    ```

6.  优化选项
    -   压缩存档：
        ```bash
        sudo zfs send zroot/home@2025-09-25 | pv | gzip > /mnt/usb/home_20250925.zfs.gz
        ```
        恢复：
        ```bash
        pv /mnt/usb/home_20250925.zfs.gz | gunzip | sudo zfs recv backup/home
        ```
    -   大文件传输稳定性（=mbuffer= + 进度）：
        ```bash
        sudo zfs send zroot/home@2025-09-25 | pv | mbuffer -s 1M -m 4G | sudo zfs recv -F backup/home
        ```


## 远程 SSH 备份 {#远程-ssh-备份}

1.  完整快照传输（带进度）
    ```bash
    sudo zfs send zroot/home@2025-09-25 | pv | ssh user@remote "sudo zfs recv -F backup/home"
    ```

2.  增量快照传输（带进度）
    ```bash
    sudo zfs send -i zroot/home@2025-09-01 zroot/home@2025-09-25 | pv | ssh user@remote "sudo zfs recv -F backup/home"
    ```

3.  传输为文件（带进度）
    ```bash
    sudo zfs send zroot/home@2025-09-25 | pv | ssh user@remote "cat > /mnt/usb/home_20250925.zfs"
    ```
    恢复：
    ```bash
    ssh user@remote "pv /mnt/usb/home_20250925.zfs" | sudo zfs recv -F backup/home
    ```

4.  网络优化传输（带进度和缓冲）
    ```bash
    sudo zfs send zroot/home@2025-09-25 | pv | mbuffer -s 1M -m 4G | ssh user@remote "mbuffer -s 1M -m 4G | sudo zfs recv -F backup/home"
    ```

5.  预估传输大小
    ```bash
    sudo zfs send -n zroot/home@2025-09-25           # 完整快照大小
    sudo zfs send -n -i zroot/home@2025-09-01 zroot/home@2025-09-25  # 增量大小
    ```


## 快照管理 {#快照管理}

1.  列出所有快照
    ```bash
    sudo zfs list -t snapshot
    ```

2.  列出特定数据集的快照
    ```bash
    sudo zfs list -t snapshot -r zroot/home
    ```

3.  删除指定快照
    ```bash
    sudo zfs destroy zroot/home@2025-09-01
    ```

4.  批量删除旧快照（保留最近7个）
    ```bash
    zfs list -t snapshot -o name -s creation | grep "zroot/home@hourly" | head -n -7 | xargs -I {} sudo zfs destroy {}
    ```


## 定期维护 {#定期维护}

1.  数据校验（每月执行）
    ```bash
    sudo zpool scrub backup
    ```

2.  检查校验进度
    ```bash
    sudo zpool status backup
    ```

3.  查看池健康状态
    ```bash
    sudo zpool list backup
    sudo zfs get used,avail,compressratio backup
    ```

4.  检查快照空间占用
    ```bash
    sudo zfs list -t snapshot -o name,used,refer zroot/home
    ```


## 故障排除 {#故障排除}

1.  池无法导入
    ```bash
    sudo zpool import -f backup          # 强制导入
    sudo zpool import -D backup          # 忽略设备ID变化
    sudo zpool import -d /dev/sda1 backup # 指定设备路径导入
    ```

2.  检查池错误
    ```bash
    sudo zpool status -v backup          # 详细错误信息
    sudo zpool clear backup              # 清除错误状态
    ```

3.  恢复中断的传输
    -   删除未完成的接收：
        ```bash
        sudo zfs destroy backup/home
        ```
    -   重新开始传输

4.  检查传输失败原因
    ```bash
    dmesg | tail -20                     # 查看系统日志
    sudo zpool events backup             # 查看ZFS事件
    ```


## 性能调优 {#性能调优}

1.  针对备份池的优化设置
    ```bash
    sudo zfs set recordsize=1M backup          # 大文件优化
    sudo zfs set primarycache=metadata backup  # 节省内存
    sudo zfs set secondarycache=none backup    # 禁用L2ARC
    sudo zfs set compression=zstd backup       # 使用zstd压缩
    ```

2.  网络传输优化
    ```bash
    # SSH连接复用配置 (~/.ssh/config)
    Host backup-server
    HostName your-backup-server.com
    ControlMaster auto
    ControlPath ~/.ssh/master-%r@%h:%p
    ControlPersist 10m
    Compression yes
    ```

3.  进度显示增强
    ```bash
    # 显示传输速度和剩余时间
    sudo zfs send zroot/home@2025-09-25 | pv -p -t -e -r -b | sudo zfs recv -F backup/home
    ```
    参数说明：

    -   `-p` 显示百分比
    -   `-t` 显示计时器
    -   `-e` 显示预计时间
    -   `-r` 显示传输速度
    -   `-b` 显示字节数


## 监控和验证 {#监控和验证}

1.  监控传输过程
    ```bash
    # 另开终端监控
    watch -n 5 'sudo zfs list backup/home'
    watch -n 5 'sudo zpool iostat backup 1 1'
    ```

2.  验证备份完整性
    ```bash
    # 比较快照属性
    sudo zfs get -r checksum,creation,used zroot/home@2025-09-25
    sudo zfs get -r checksum,creation,used backup/home@2025-09-25

    # 检查数据集大小是否匹配
    sudo zfs list zroot/home backup/home
    ```

3.  测试恢复
    ```bash
    # 创建测试恢复到临时位置
    sudo zfs send backup/home@2025-09-25 | sudo zfs recv backup/test-restore

    # 验证后清理
    sudo zfs destroy -r backup/test-restore
    ```


## 注意事项 {#注意事项}

-   **-f** 会覆盖设备上原有数据，谨慎使用
-   快照命名建议带时间戳，便于管理：=@2025-09-25= 或 `@hourly-2025-09-25-14`
-   始终使用 **zstd** 压缩，提供最佳的压缩率和性能平衡
-   远程备份建议设置 SSH 免密登录，方便操作
-   定期执行 `scrub` 检查数据完整性
-   使用 `pv` 命令显示传输进度，便于监控大文件传输
-   重要数据建议保持 3-2-1 备份策略（3个副本，2种介质，1个异地）
-   传输前用 `-n` 参数预估数据大小，合理安排时间
