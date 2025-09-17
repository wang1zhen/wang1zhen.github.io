+++
title = "sing-box webui 配置"
author = ["wang1zhen"]
description = "说明安装 sing-box、导入订阅并使用可视化控制面板。"
date = 2025-09-15T00:00:00+09:00
draft = false
+++

## 安装 sing-box {#安装-sing-box}

```bash
yay -S sing-box
```


## 获取订阅配置 {#获取订阅配置}

从 prprcloud 后台复制订阅链接，然后下载到配置目录：

```bash
curl -o ~/.config/sing-box/config.json '你的订阅链接'
```

如果在海外使用，选择直连/direct 节点。


## 运行 sing-box {#运行-sing-box}

```bash
sudo sing-box run -c ~/.config/sing-box/config.json
```


## 使用 Dashboard {#使用-dashboard}

-   如果需要 Zashboard，直接访问：
    <http://board.zash.run.place/>
    在设置页无需填写任何内容。
-   如果不使用 Zashboard，也可以直接访问：
    <http://localhost:9090>
    搭配 yacd 使用。


## Aff 链接 {#aff-链接}

[prprcloud 订阅链接](https://prpr.96110.cn.com/aff.php?aff=6492)
