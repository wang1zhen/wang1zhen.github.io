+++
title = "用 podman 部署 file browser (免登录)"
author = ["wang1zhen"]
description = "利用 podman 来部署 file browser，无需登录账号，直接查看文件。有图片查看、视频播放、文件下载功能。"
date = 2025-09-25T00:00:00+09:00
draft = false
+++

## Quadlet 配置 {#quadlet-配置}

路径：~/.config/containers/systemd/filebrowser.container

```ini
[Unit]
Description=FileBrowser
After=network-online.target

[Container]
Image=docker.io/filebrowser/filebrowser:latest
ContainerName=filebrowser
UserNS=keep-id
PublishPort=8080:8080
Volume=/share/web:/srv:ro
Volume=/share/filebrowser/database:/database
Volume=/share/filebrowser/config:/config
Exec=--noauth --port 8080
AutoUpdate=registry

[Service]
Restart=always
TimeoutStartSec=900

[Install]
WantedBy=default.target
```


## 配置要点 {#配置要点}

-   UserNS=keep-id：容器使用当前用户 UID/GID，避免权限错乱
-   Volume 映射
    -   /share/web:/srv:ro：网站根目录，只读
    -   /share/filebrowser/database:/database：数据库持久化
    -   /share/filebrowser/config:/config：配置持久化
-   Exec=--noauth --port 8080：关闭登录认证，直通访问
-   Restart=always：异常退出自动重启
-   AutoUpdate=registry：容器会根据镜像仓库版本自动更新


## 自动升级机制 {#自动升级机制}

-   AutoUpdate=registry 启用后，Podman 的 systemd 集成会在 `systemd.timer` 定期触发时检查镜像更新
-   默认检查周期：每日（受 `podman-auto-update.timer` 控制）
-   升级时机：
    -   如果仓库有新版本镜像 → 下载新镜像
    -   自动重启受影响的服务，使用新镜像运行容器
-   可手动触发：

<!--listend-->

```bash
systemctl --user start podman-auto-update.service
```


## 启动流程 {#启动流程}

```bash
mkdir -p /share/web \
      /share/filebrowser/database \
      /share/filebrowser/config

loginctl enable-linger $USER
systemctl --user daemon-reload
systemctl --user enable --now filebrowser.service
```


## 验证 {#验证}

```bash
systemctl --user status filebrowser
journalctl --user -u filebrowser.service -f
podman ps | grep filebrowser
```

访问：<http://<宿主机ip>:8080>
