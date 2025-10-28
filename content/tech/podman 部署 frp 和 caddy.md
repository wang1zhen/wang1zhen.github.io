+++
title = "podman 部署 frp 和 caddy"
author = ["wang1zhen"]
description = "利用 podman 来部署 frp 服务，并用 caddy 代理实现 HTTPS 访问"
date = 2025-10-28T00:00:00+09:00
draft = false
+++

## 系统级目录布局 {#系统级目录布局}

应用配置放在 /opt；Quadlet 单元放在 /etc/containers/systemd。

```text
/opt/podman/
├─ frp/
│  └─ frps.toml
└─ caddy/
   └─ Caddyfile

/etc/containers/systemd/
├─ frps.container
└─ caddy.container
```


## frps 配置 {#frps-配置}

`/opt/podman/frp/frps.toml`

```toml
bindPort = 7000

auth.method = "token"
auth.token = "REPLACE_WITH_STRONG_TOKEN"

webServer.addr = "0.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "REPLACE_WITH_STRONG_PASSWORD"
```


## Caddy 配置 {#caddy-配置}

`/opt/podman/caddy/Caddyfile`

```cfg
vps.wang1zhen.com:7500 {
  reverse_proxy http://127.0.0.1:7750
}

:80 {
  root * /usr/share/caddy
  file_server
}
:443 {
  root * /usr/share/caddy
  file_server
  tls internal
}
```


## frps Quadlet {#frps-quadlet}

`/etc/containers/systemd/frps.container`

```ini
[Unit]
Description=frps

[Container]
Image=docker.io/snowdreamtech/frps:alpine
ContainerName=frps
PublishPort=7000:7000/tcp
PublishPort=127.0.0.1:7750:7500/tcp
Volume=/opt/podman/frp/frps.toml:/etc/frp/frps.toml:ro

[Service]
Restart=always

[Install]
WantedBy=multi-user.target
```


## Caddy Quadlet {#caddy-quadlet}

`/etc/containers/systemd/caddy.container`

```ini
[Unit]
Description=Caddy

[Container]
Image=docker.io/caddy:2
ContainerName=caddy
Network=host
Volume=/opt/podman/caddy/Caddyfile:/etc/caddy/Caddyfile:ro
Volume=/opt/podman/caddy:/data
Pull=always

[Service]
Restart=always

[Install]
WantedBy=multi-user.target
```


## 启动 {#启动}

sudo systemctl daemon-reload
sudo systemctl start frpnet-network.service frps.service caddy.service

验证
curl -I <http://frp.example.com>        # 应 301 到 https
curl -I <https://frp.example.com>
curl -I <https://frp.example.com:7500>
podman logs --since=10m frps
podman logs --since=10m caddy-frps
