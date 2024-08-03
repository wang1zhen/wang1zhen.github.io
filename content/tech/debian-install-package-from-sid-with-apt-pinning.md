+++
title = "在 Debian 稳定版中通过 APT Pinning 安装特定的 Sid 软件包"
author = ["wang1zhen"]
date = 2024-08-03T19:45:00+09:00
draft = false
+++

为了在 Debian 稳定版中安装特定来自 Sid 的软件包，同时保持系统的整体稳定性，可以通过 APT Pinning 来实现精细化的版本控制。


## 配置 APT Pinning {#配置-apt-pinning}

在 `/etc/apt/preferences.d/` 中设置优先级。

```text
# Stable has the highest priority
Package: *
Pin: release a=stable
Pin-Priority: 900

# Backports have higher priority than sid, but lower than stable
Package: *
Pin: release a=stable-backports
Pin-Priority: 800

# Sid has the lowest priority
Package: *
Pin: release a=unstable
Pin-Priority: 100
```

-   \`a=stable\`: 优先级 900 确保来自当前稳定版（例如 \`bookworm\`）的包具有最高优先级。
-   \`a=stable-backports\`: 优先级 800 确保 Backports 中的包在没有其他更高优先级（如 \`stable\`）的情况下可以被选中。
-   \`a=unstable\`: 优先级 100 确保 Sid 的包只有在明确指定时才会被安装。


## 只对特定包从 sid 更新 {#只对特定包从-sid-更新}

建立 `/etc/apt/preferences.d/sid-packages` 文件

```text
Package: emacs-pgtk
Pin: release a=unstable
Pin-Priority: 1001

Package: hugo
Pin: release a=unstable
Pin-Priority: 1001
```


## 应用 {#应用}

```bash
sudo apt update
apt list --upgradable
apt policy <some-package>
```
