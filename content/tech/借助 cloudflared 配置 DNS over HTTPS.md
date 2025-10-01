+++
title = "借助 cloudflared 配置 DNS over HTTPS"
author = ["wang1zhen"]
description = "借助 cloudflared 在 Arch linux 下配置 DNS over HTTPS 并验证。"
date = 2025-10-01T00:00:00+09:00
draft = false
+++

## 安装配置步骤 {#安装配置步骤}


### 安装 cloudflared {#安装-cloudflared}

```bash
sudo pacman -S cloudflared
```


### 创建 systemd 服务文件 {#创建-systemd-服务文件}

```bash
sudo vim /etc/systemd/system/cloudflared.service
```

```cfg
[Unit]
Description=DNS over HTTPS proxy client
Wants=network-online.target nss-lookup.target
Before=nss-lookup.target

[Service]
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
DynamicUser=yes
ExecStart=/usr/bin/cloudflared proxy-dns --port 5053 --upstream https://1.1.1.1/dns-query --upstream https://1.0.0.1/dns-query

[Install]
WantedBy=multi-user.target
```


### 启动 cloudflared 服务 {#启动-cloudflared-服务}

```bash
# 重载 systemd 配置
sudo systemctl daemon-reload

# 启用开机自启
sudo systemctl enable cloudflared

# 立即启动服务
sudo systemctl start cloudflared

# 检查服务状态
sudo systemctl status cloudflared
```

确认服务显示为 `active (running)` 。


### 配置 systemd-resolved {#配置-systemd-resolved}


#### 创建 drop-in 配置目录 {#创建-drop-in-配置目录}

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
```


#### 创建配置文件 {#创建配置文件}

```bash
sudo vim /etc/systemd/resolved.conf.d/cloudflare-doh.conf
```

```cfg
[Resolve]
DNS=127.0.0.1:5053
FallbackDNS=1.1.1.1 1.0.0.1
DNSOverTLS=no
Domains=~.
```


### 重启 systemd-resolved {#重启-systemd-resolved}

```bash
sudo systemctl restart systemd-resolved
```


### 验证配置 {#验证配置}


#### 检查 cloudflared 端口监听 {#检查-cloudflared-端口监听}

```bash
sudo ss -tulpn | grep 5053
```

应该看到类似输出：

```text
udp   UNCONN 0      0          127.0.0.1:5053       0.0.0.0:*
tcp   LISTEN 0      4096       127.0.0.1:5053       0.0.0.0:*
```


#### 查看 DNS 配置状态 {#查看-dns-配置状态}

```bash
resolvectl status
```

应该看到：

```text
Global
       DNS Servers: 127.0.0.1:5053
Fallback DNS Servers: 1.1.1.1
                      1.0.0.1
          DNS Domain: ~.
```


#### 测试 DNS 解析 {#测试-dns-解析}

```bash
# 测试普通域名
resolvectl query google.com

# 测试 Cloudflare
resolvectl query cloudflare.com

# 测试 Tailscale 域名
resolvectl query something.tail4ef8f.ts.net
```


#### 查看合并后的完整配置 {#查看合并后的完整配置}

```bash
systemd-analyze cat-config systemd/resolved.conf
```


#### 在浏览器中验证 DoH {#在浏览器中验证-doh}

访问：<https://1.1.1.1/help>

查看页面显示：

-   **Using DNS over HTTPS (DoH)** : Yes ✓
-   **Connected to** : 1.1.1.1


### 查看服务日志（可选） {#查看服务日志-可选}


#### 查看 cloudflared 日志 {#查看-cloudflared-日志}

```bash
# 查看最近 50 条日志
sudo journalctl -u cloudflared -n 50

# 实时查看日志
sudo journalctl -u cloudflared -f
```


#### 查看 systemd-resolved 日志 {#查看-systemd-resolved-日志}

```bash
sudo journalctl -u systemd-resolved -n 50
```


## 故障排查 {#故障排查}


### cloudflared 服务无法启动 {#cloudflared-服务无法启动}


#### 排查步骤 {#排查步骤}

```bash
# 查看详细错误信息
sudo journalctl -u cloudflared -n 50 --no-pager

# 检查端口占用
sudo ss -tulpn | grep 5053

# 手动测试 cloudflared
sudo cloudflared proxy-dns --port 5053 --upstream https://1.1.1.1/dns-query
```


#### 常见原因 {#常见原因}

-   端口 5053 被占用
-   网络连接问题


### DNS 解析不工作 {#dns-解析不工作}


#### 排查步骤 {#排查步骤}

```bash
# 检查两个服务状态
systemctl status cloudflared
systemctl status systemd-resolved

# 检查端口监听
sudo ss -tulpn | grep -E ':(53|5053)'

# 测试本地 DNS
resolvectl query example.com
```


#### 解决方法 {#解决方法}

```bash
# 重启服务
sudo systemctl restart cloudflared
sudo systemctl restart systemd-resolved

# 清除 DNS 缓存
resolvectl flush-caches
```


### Tailscale 域名无法解析 {#tailscale-域名无法解析}


#### 检查 Tailscale 接口 {#检查-tailscale-接口}

```bash
resolvectl status tailscale0
```

Tailscale 应该自动配置其 DNS，如果有问题：

```bash
# 重启 Tailscale
sudo systemctl restart tailscaled

# 重启 systemd-resolved
sudo systemctl restart systemd-resolved
```


## 性能优化（可选） {#性能优化-可选}


### 调整 DNS 缓存时间 {#调整-dns-缓存时间}

编辑配置：

```bash
sudo vim /etc/systemd/resolved.conf.d/cloudflare-doh.conf
```

添加缓存设置：

```cfg
[Resolve]
DNS=127.0.0.1:5053
FallbackDNS=1.1.1.1 1.0.0.1
DNSOverTLS=no
Domains=~.
Cache=yes
CacheFromLocalhost=yes
```

重启服务：

```bash
sudo systemctl restart systemd-resolved
```


## 卸载配置 {#卸载配置}

如需恢复默认配置：

```bash
# 停止并禁用 cloudflared
sudo systemctl stop cloudflared
sudo systemctl disable cloudflared

# 删除服务文件
sudo rm /etc/systemd/system/cloudflared.service

# 删除 resolved 配置
sudo rm /etc/systemd/resolved.conf.d/cloudflare-doh.conf

# 重载 systemd
sudo systemctl daemon-reload

# 重启 systemd-resolved
sudo systemctl restart systemd-resolved

# 验证恢复
resolvectl status
```


## 配置文件位置总结 {#配置文件位置总结}

| 文件/目录                                          | 作用                    |
|------------------------------------------------|-----------------------|
| `/etc/systemd/system/cloudflared.service`          | cloudflared 服务配置    |
| `/etc/systemd/resolved.conf.d/cloudflare-doh.conf` | systemd-resolved DNS 配置 |
| `/etc/systemd/resolved.conf`                       | 主配置文件（未修改）    |
| `/etc/resolv.conf`                                 | 符号链接（无需修改）    |


## 工作原理示意图 {#工作原理示意图}

```text
应用程序
    ↓
systemd-resolved (127.0.0.53:53)
    ↓
cloudflared (127.0.0.1:5053)
    ↓
Cloudflare DoH (HTTPS 加密)
    ↓
1.1.1.1 / 1.0.0.1
```


## 相关资源 {#相关资源}

-   [Arch Wiki - Cloudflared](https://wiki.archlinux.org/title/Cloudflared)
-   [Cloudflare 1.1.1.1 Documentation](https://developers.cloudflare.com/1.1.1.1/)
-   [systemd-resolved.conf Manual](https://www.freedesktop.org/software/systemd/man/resolved.conf.html)
-   [Cloudflare DNS 测试页面](https://1.1.1.1/help)
