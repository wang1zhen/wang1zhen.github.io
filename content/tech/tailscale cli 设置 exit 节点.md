+++
title = "tailscale cli 设置 exit 节点"
author = ["wang1zhen"]
description = "为 tailscale cli 设置 exit 节点"
date = 2025-09-18T00:00:00+09:00
draft = false
+++

## 查看可用 Exit 节点 {#查看可用-exit-节点}

```bash
tailscale status
```


## 指定 Exit 节点 {#指定-exit-节点}

```bash
# 节点名或 Tailscale 内网 IP
tailscale up --exit-node=mynode
```


## 保留本地局域网访问 {#保留本地局域网访问}

```bash
tailscale up --exit-node=mynode --exit-node-allow-lan-access=true
```


## 取消 Exit 节点 {#取消-exit-节点}

```bash
tailscale up --exit-node=
```


## 参数说明 {#参数说明}

-   \`--exit-node=&lt;节点名或IP&gt;\` 指定出口节点
-   \`--exit-node-allow-lan-access\` 允许访问本地局域网，否则所有流量都走 exit 节点
