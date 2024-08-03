+++
title = "在 Debian 稳定版中通过 APT Pinning 安装特定的 Sid 软件包"
author = ["wang1zhen"]
date = 2024-08-03T19:45:00+09:00
draft = false
+++

为了在 Debian 稳定版中安装特定来自 Sid 的软件包，同时保持系统的整体稳定性，可以通过 APT Pinning 来实现精细化的版本控制。


## 配置 APT Pinning {#配置-apt-pinning}

在 `/etc/apt/preferences.d/sid` 中设置优先级。

```text
Package: *
Pin: release a=stable
Pin-Priority: 900

Package: *
Pin: release a=stable-updates
Pin-Priority: 900

Package: *
Pin: release a=stable-security
Pin-Priority: 900

Package: *
Pin: release a=stable-backports
Pin-Priority: 800

Package: *
Pin: release a=unstable
Pin-Priority: 100
```


## 只对特定包从 sid 更新 {#只对特定包从-sid-更新}

`/etc/apt/preferences.d/sid`

```text
Package: emacs-pgtk hugo eza
Pin: release a=unstable
Pin-Priority: 1001
```


## 应用 {#应用}

```bash
sudo apt update
apt list --upgradable
apt policy <some-package>
```
