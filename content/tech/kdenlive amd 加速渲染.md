+++
title = "kdenlive amd 加速渲染"
author = ["wang1zhen"]
description = "在 Arch Linux 下使用 kdenlive 时利用 amd 显卡加速渲染"
date = 2025-10-25T00:00:00+09:00
draft = false
+++

## 系统准备 {#系统准备}

```bash
sudo pacman -S kdenlive ffmpeg libva-utils libva-mesa-driver mesa-vdpau libvdpau-va-gl amdgpu_top
vainfo | grep -E "H264|HEVC|AV1"
sudo cat /sys/class/drm/renderD128/device/uevent | grep PCI_SLOT_NAME
```


## Kdenlive 自定义预设 {#kdenlive-自定义预设}

****H.264 固定质量****

```bash
f=mp4 vcodec=h264_vaapi vaapi_device=/dev/dri/renderD128 rc=constqp qp=22 g=240 bf=2 acodec=aac ab=192k channels=2 movflags=+faststart
```

****HEVC（H.265）版本****

```bash
f=mp4 vcodec=hevc_vaapi vaapi_device=/dev/dri/renderD128 rc=constqp qp=22 g=240 bf=2 acodec=aac ab=192k channels=2 movflags=+faststart
```


## 参数说明 {#参数说明}

| 参数                             | 作用           |
|--------------------------------|--------------|
| vaapi_device=/dev/dri/renderD128 | 指定 RX 7700 XT |
| rc=constqp                       | 固定量化模式   |
| qp=22                            | 控制画质，数值越小越清晰 |
| g=240                            | GOP 长度，约 4 秒 |
| bf=2                             | 启用双向预测帧，提升压缩效率 |
| acodec=aac ab=192k channels=2    | 音频配置       |
| movflags=+faststart              | 优化 MP4 播放头 |


## GPU 使用率监控 {#gpu-使用率监控}

```bash
amdgpu_top
# 或简化方式
watch -n 1 grep "VCN Encode" /sys/kernel/debug/dri/*/amdgpu_pm_info
```

****判断标准****

-   VCN Encode 活动上升 → GPU 硬件编码已启用。
-   20–60% 占用为正常范围（VCN 为固定功能单元）。
