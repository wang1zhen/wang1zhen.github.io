+++
title = "podman 部署 frp 和 caddy"
author = ["wang1zhen"]
description = "利用 podman 来部署 frp 服务，并用 caddy 代理实现 HTTPS 访问"
date = 2025-10-28T00:00:00+09:00
draft = false
+++

## 系统级目录布局 {#系统级目录布局}

应用配置放在 /opt；Quadlet 单元放在 /etc/containers/systemd。

目录布局（系统级）
_opt/podman/frp/frps.toml
/opt/podman/caddy/Caddyfile
/opt/podman/caddy/data_        ← Caddy 证书与 ACME 存储（挂到容器 _data）
/opt/podman/caddy/config_      ← Caddy 运行期配置（挂到容器 /config）


## frps 配置 {#frps-配置}

`/opt/podman/frp/frps.toml`

```toml
bindPort = 7000

auth.method = "token"
auth.token = "asdfasdfasdf"

webServer.addr = "0.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "CHANGE_ME_STRONG"
```


## Caddy 配置 {#caddy-配置}

`/opt/podman/caddy/Caddyfile`

```json
:80 {
  root * /usr/share/caddy
  file_server
}

:443 {
  root * /usr/share/caddy
  file_server
}

# 仅 7501 走反代 + HTTPS（自动证书）
frp.example.com:7501 {
  encode gzip
  reverse_proxy frps:7500
}
```


## Quadlet 网络 {#quadlet-网络}

`/etc/containers/systemd/frpnet.network`

```text
[Network]
Driver=bridge
```


## frps Quadlet {#frps-quadlet}

`/etc/containers/systemd/frps.container`

```text
[Unit]
Description=FRP server

[Container]
Image=fatedier/frps:latest
ContainerName=frps
Network=frpnet
Volume=/opt/podman/frp/frps.toml:/etc/frp/frps.toml
PublishPort=7000:7000
Command=-c /etc/frp/frps.toml

[Service]
Restart=always
```


## Caddy Quadlet {#caddy-quadlet}

`/etc/containers/systemd/caddy-frps.container`

```text
[Unit]
Description=Caddy

[Container]
Image=caddy:alpine
ContainerName=caddy
Network=frpnet
PublishPort=80:80
PublishPort=443:443
PublishPort=7501:7501
Volume=/opt/podman/caddy/Caddyfile:/etc/caddy/Caddyfile
Volume=/opt/podman/caddy/data:/data
Volume=/opt/podman/caddy/config:/config

[Service]
Restart=always
```


## 启动 {#启动}

sudo systemctl daemon-reload
sudo systemctl start frpnet.service frps.service caddy.service

验证
curl -I <http://frp.example.com>        # 应 301 到 https
curl -I <https://frp.example.com>
curl -I <https://frp.example.com:7501>
podman logs --since=10m frps
podman logs --since=10m caddy-frps

防火墙

-   放行 80/tcp, 443/tcp, 7000/tcp, 7501/tcp。
-   7500 不暴露宿主，仅桥接网络内可见。
