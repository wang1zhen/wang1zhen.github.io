+++
title = "为 Linux 系统启用 zram"
author = ["wang1zhen"]
date = 2024-07-22T14:08:00+09:00
draft = false
+++

在 linux 系统中使用 zram 可以对内存进行压缩（现代 CPU 下几乎即时），提升表现。


## zram-generator {#zram-generator}


### 安装 {#安装}

```shell
sudo pacman -S zram-generator
```


### 配置 systemd service {#配置-systemd-service}

`/etc/systemd/zram-generator.conf`

```text
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
```


### systemd {#systemd}

```shell
sudo systemctl daemon-reload
sudo systemctl start systemd-zram-setup@zram0.service
```


### 优化 {#优化}

`/etc/sysctl.d/99-vm-zram-parameters.conf`

```text
vm.swappiness = 200
vm.watermark_boost_factor = 0
vm.watermark_scale_factor = 125
vm.page-cluster = 0
```
