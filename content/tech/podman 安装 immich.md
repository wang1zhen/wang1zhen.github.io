+++
title = "podman 安装 immich"
author = ["wang1zhen"]
description = "用 podman 来安装 immich 并启用硬件加速。"
date = 2025-09-24T00:00:00+09:00
draft = false
+++

## 目录与环境变量 {#目录与环境变量}

-   建议使用绝对路径。
-   样例目录
    -   /share/immich/library
    -   /share/immich/postgres
-   环境文件路径
    -   ~/.config/containers/systemd/immich/.env

<!--listend-->

```bash
mkdir -p /share/immich/{library,postgres}
mkdir -p ~/.config/containers/systemd/immich
```


## .env 样例 {#dot-env-样例}

```bash
# 必填 - 改为绝对路径
UPLOAD_LOCATION=/share/immich/library
DB_DATA_LOCATION=/share/immich/postgres

# 版本标签
IMMICH_VERSION=release

# 时区
TZ=Asia/Tokyo

# 数据库
DB_PASSWORD=postgres
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```


## Quadlet 网络与命名卷 {#quadlet-网络与命名卷}

```cfg
# ~/.config/containers/systemd/immich-network.network
[Network]
NetworkName=immich

[Install]
WantedBy=default.target
```

```cfg
# ~/.config/containers/systemd/model-cache.volume
[Volume]
VolumeName=model-cache

[Install]
WantedBy=default.target
```


## Valkey {#valkey}

```cfg
# ~/.config/containers/systemd/immich-redis.container
[Unit]
Description=Immich Redis (Valkey)

[Container]
Image=docker.io/valkey/valkey:8-bookworm
ContainerName=immich_redis
Network=immich
RestartPolicy=always

[Install]
WantedBy=default.target
```


## Postgres {#postgres}

```cfg
# ~/.config/containers/systemd/immich-postgres.container
[Unit]
Description=Immich Postgres (vectorchord + pgvector)
After=network-online.target
Wants=network-online.target

[Container]
EnvironmentFile=%h/.config/containers/systemd/immich/.env
Environment=POSTGRES_PASSWORD=%E{DB_PASSWORD}
Environment=POSTGRES_USER=%E{DB_USERNAME}
Environment=POSTGRES_DB=%E{DB_DATABASE_NAME}
Environment=POSTGRES_INITDB_ARGS=--data-checksums
Image=ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0
ContainerName=immich_postgres
Volume=%E{DB_DATA_LOCATION}:/var/lib/postgresql/data
ShmSize=128m
Network=immich
RestartPolicy=always

[Install]
WantedBy=default.target
```


## Machine Learning - CPU 基础版 {#machine-learning-cpu-基础版}

```cfg
# ~/.config/containers/systemd/immich-machine-learning.container
[Unit]
Description=Immich Machine Learning
After=immich-redis.service immich-postgres.service
Wants=immich-redis.service immich-postgres.service

[Container]
EnvironmentFile=%h/.config/containers/systemd/immich/.env
Image=ghcr.io/immich-app/immich-machine-learning:%E{IMMICH_VERSION}
ContainerName=immich_machine_learning
Volume=model-cache:/cache
Network=immich
RestartPolicy=always

[Install]
WantedBy=default.target
```


## Server - CPU 基础版 {#server-cpu-基础版}

```cfg
# ~/.config/containers/systemd/immich-server.container
[Unit]
Description=Immich Server
After=immich-redis.service immich-postgres.service
Wants=immich-redis.service immich-postgres.service

[Container]
EnvironmentFile=%h/.config/containers/systemd/immich/.env
Image=ghcr.io/immich-app/immich-server:%E{IMMICH_VERSION}
ContainerName=immich_server
PublishPort=2283:2283
Volume=%E{UPLOAD_LOCATION}:/data
Volume=/etc/localtime:/etc/localtime:ro
Network=immich
RestartPolicy=always

[Install]
WantedBy=default.target
```


## 启动与管理 {#启动与管理}

```bash
systemctl --user daemon-reload
systemctl --user enable --now immich-network.network model-cache.volume
systemctl --user enable --now immich-postgres.service immich-redis.service
systemctl --user enable --now immich-machine-learning.service immich-server.service

# 查看日志
journalctl --user -u immich-server.service -f
journalctl --user -u immich-machine-learning.service -f
```


## GPU 加速 - 总览 {#gpu-加速-总览}

-   只改两处：immich-machine-learning.container 与 immich-server.container。


## NVIDIA - CUDA 推理与 NVENC 转码 {#nvidia-cuda-推理与-nvenc-转码}

-   主机准备
    -   安装专有驱动与 NVIDIA Container Toolkit（启用 OCI Hook）。
    -   确认 _usr/share/containers/oci/hooks.d_ 存在 nvidia hook。
-   immich-machine-learning.container

<!--listend-->

```cfg
[Container]
EnvironmentFile=%h/.config/containers/systemd/immich/.env
Image=ghcr.io/immich-app/immich-machine-learning:%E{IMMICH_VERSION}-cuda
ContainerName=immich_machine_learning
Volume=model-cache:/cache
Network=immich
Environment=NVIDIA_VISIBLE_DEVICES=all
Environment=NVIDIA_DRIVER_CAPABILITIES=compute,video,utility
Device=/dev/nvidiactl
Device=/dev/nvidia0
Device=/dev/nvidia-uvm
RestartPolicy=always
```

-   immich-server.container

<!--listend-->

```cfg
[Container]
EnvironmentFile=%h/.config/containers/systemd/immich/.env
Image=ghcr.io/immich-app/immich-server:%E{IMMICH_VERSION}
ContainerName=immich_server
PublishPort=2283:2283
Volume=%E{UPLOAD_LOCATION}:/data
Volume=/etc/localtime:/etc/localtime:ro
Network=immich
Environment=NVIDIA_VISIBLE_DEVICES=all
Environment=NVIDIA_DRIVER_CAPABILITIES=video,utility
Device=/dev/nvidiactl
Device=/dev/nvidia0
Device=/dev/nvidia-uvm
RestartPolicy=always
```


## AMD - ROCm 推理与 VA-API 转码 {#amd-rocm-推理与-va-api-转码}

-   主机准备
    -   安装 ROCm 运行时。
    -   将用户加入 render 与 video 组。

<!--listend-->

```bash
sudo usermod -aG render,video "$USER"
newgrp render
```

-   immich-machine-learning.container

<!--listend-->

```cfg
[Container]
EnvironmentFile=%h/.config/containers/systemd/immich/.env
Image=ghcr.io/immich-app/immich-machine-learning:%E{IMMICH_VERSION}-rocm
ContainerName=immich_machine_learning
Volume=model-cache:/cache
Network=immich
Device=/dev/kfd
Device=/dev/dri
RestartPolicy=always
```

-   immich-server.container

<!--listend-->

```cfg
[Container]
EnvironmentFile=%h/.config/containers/systemd/immich/.env
Image=ghcr.io/immich-app/immich-server:%E{IMMICH_VERSION}
ContainerName=immich_server
PublishPort=2283:2283
Volume=%E{UPLOAD_LOCATION}:/data
Volume=/etc/localtime:/etc/localtime:ro
Network=immich
Device=/dev/dri
RestartPolicy=always
```


## 改动后重载 {#改动后重载}

```bash
systemctl --user daemon-reload
systemctl --user restart immich-machine-learning.service immich-server.service
```


## 映射目录与设备说明 {#映射目录与设备说明}

-   Volume=%E{UPLOAD_LOCATION}:/data
    -   Immich 媒体与派生数据的根。
    -   典型结构
        -   original/ 原始媒体
        -   thumbs/ 缩略图与预览
        -   encoded-video/ 转码缓存
        -   backup/ 应用内部备份
        -   faces/ clips/ index/ 特征与向量索引缓存
-   Volume=%E{DB_DATA_LOCATION}:/var/lib/postgresql/data
    -   Postgres 数据目录。
    -   结构
        -   base/ 表文件
        -   pg_wal/ 预写式日志
        -   global/ 全局元数据
        -   其他系统目录
-   Volume=model-cache:/cache
    -   ML 模型与推理缓存，避免重复下载或编译。
-   Volume=/etc/localtime:/etc/localtime:ro
    -   容器时间与宿主同步，日志时间正确。
-   Device=/dev/dri
    -   暴露渲染节点给容器，开启 VA-API 或 QuickSync。
-   Device=/dev/nvidiactl /dev/nvidia0 /dev/nvidia-uvm
    -   直通 NVIDIA 设备，配合 OCI Hook 启用 CUDA 与 NVENC。


## 故障排查 {#故障排查}

-   权限
    -   rootless 需加入 render、video 组。检查 ls -l /dev/dri。
-   编码失败
    -   查看 ffmpeg 日志

<!--listend-->

```bash
journalctl --user -u immich-server.service -f
```

-   Intel 可尝试设置 LIBVA_DRIVER_NAME=iHD。
-   CUDA 不可见
    -   检查 NVIDIA OCI Hook 是否存在。
    -   确认容器内可运行 nvidia-smi。
-   模型重复下载
    -   确认 /cache 已挂载命名卷 model-cache。
