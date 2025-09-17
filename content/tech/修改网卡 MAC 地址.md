+++
title = "修改网卡 MAC 地址"
author = ["wang1zhen"]
description = "记录在 Windows 注册表启用 Network Address 修改 MAC。"
date = 2024-10-16T00:00:00+09:00
draft = false
+++

From <https://superuser.com/questions/1011721/how-do-i-change-wifi-adapter-mac-address-for-win7-8-10-network-adapter-advance>


## Edit Registry {#edit-registry}

`win+r` regedit
`HKLM\SYSTEM\CurrentControlSet\Control\Class\{4D36E972-E325-11CE-BFC1-08002BE10318}\00xx\NDI\params`

Replace `00xx` with the numerical key associated with your network adapter by checking the `DriverDesc` string value.

Under `params`, create a new key named `NetworkAddress`.

Add the following string values under the `NetworkAddress` key:

```text
"optional"="1"
"type"="edit"
"uppercase"="1"
"limittext"="12"
"paramdesc"="Network Address"
```

After completing the steps, check the "Advanced" tab in your network adapter's properties. The "Network Address" field should appear.


## .reg file for reference: {#dot-reg-file-for-reference}

`MAC.reg`

```text
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Class\{4D36E972-E325-11CE-BFC1-08002BE10318}\0009\NDI\params\NetworkAddress]
"optional"="1"
"type"="edit"
"uppercase"="1"
"limittext"="12"
"paramdesc"="Network Address"
```

In this example, the Wi-Fi adapter is located under \`0009\`. Modify the key to match the specific adapter on your machine.
