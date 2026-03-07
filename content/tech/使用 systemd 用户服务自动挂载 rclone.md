+++
title = "使用 systemd 用户服务自动挂载 rclone"
author = ["wang1zhen"]
description = "创建用户 systemd 服务来自动挂载 rclone，以 onedrive 为例"
date = 2026-03-07T00:00:00+09:00
draft = false
+++

## 简介 {#简介}

使用 `systemd` 的 **用户服务 (User Service)** 来自动挂载 rclone 是最优雅且安全的做法。不需要 root 权限，且挂载点直接关联到当前用户。本文档记录了将名为 `onedrive` 的 rclone remote 挂载到 `~/onedrive` 的完整流程。


## 创建本地挂载目录 {#创建本地挂载目录}

在开始配置服务之前，必须确保目标挂载文件夹存在，否则 systemd 会拒绝启动服务。

```bash
mkdir -p ~/onedrive
```


## 创建 systemd 用户服务文件 {#创建-systemd-用户服务文件}

我们需要在用户的 systemd 配置目录下创建一个服务文件。

1.  创建存放服务文件的目录（如果尚不存在）：
    ```bash
    mkdir -p ~/.config/systemd/user/
    ```

2.  创建并编辑服务文件 `rclone-onedrive.service` ：
    ```bash
    vim ~/.config/systemd/user/rclone-onedrive.service
    ```

3.  填入以下配置内容（注意 `%h` 会自动解析为当前用户的家目录）：
    ```ini
    [Unit]
    Description=Rclone Mount Service for ~/onedrive
    AssertPathIsDirectory=%h/onedrive
    After=network-online.target

    [Service]
    Type=notify
    # 启动挂载
    # 将 onedrive: 替换为你的 rclone remote 名称
    ExecStart=/usr/bin/rclone mount onedrive: %h/onedrive \
        --config=%h/.config/rclone/rclone.conf \
        --vfs-cache-mode writes \
        --allow-non-empty \
        --dir-cache-time 1h \
        --log-level INFO

    # 停止挂载时卸载目录
    ExecStop=/usr/bin/fusermount -u %h/onedrive

    # 崩溃后自动重启
    Restart=on-failure
    RestartSec=10

    [Install]
    WantedBy=default.target
    ```


## 重载并启用服务 {#重载并启用服务}

通知 systemd 读取新创建的配置文件，并启动挂载。

```bash
# 1. 重新加载用户的 systemd 守护进程
systemctl --user daemon-reload

# 2. 启动服务测试是否成功
systemctl --user start rclone-onedrive.service

# 3. 检查服务状态 (需显示 active (running))
systemctl --user status rclone-onedrive.service

# 4. 设置为开机/登录时自动启动
systemctl --user enable rclone-onedrive.service
```
