+++
title = "kdenlive amd 加速渲染"
author = ["wang1zhen"]
description = "在 Arch Linux 下使用 kdenlive 时利用 amd 显卡加速渲染"
date = 2026-02-16T00:00:00+09:00
draft = false
+++

## 系统准备 {#系统准备}

```bash
sudo pacman -S kdenlive ffmpeg libva-utils libva-mesa-driver amdgpu_top
vainfo | grep -E "H264|HEVC|AV1"
sudo cat /sys/class/drm/renderD128/device/uevent | grep PCI_SLOT_NAME
lspci -nn | grep -E "VGA|3D|Display"
```


## 参数 {#参数}


### H264 {#h264}

**Windows (AMF)**

```shell
f=mp4 vcodec=h264_amf rc=cqp qp_i=20 qp_p=20 qp_b=20 g=75 bf=2 acodec=aac ab=320k usage=transcoding quality=quality vprofile=high movflags=+faststart
```

**Linux (VAAPI)**

```shell
f=mp4 vcodec=h264_vaapi vaapi_device=/dev/dri/renderD128 qp=20 g=75 bf=2 acodec=aac ab=320k vprofile=high movflags=+faststart
```


### H.265 / HEVC {#h-dot-265-hevc}

**Windows (AMF)**

```shell
f=mp4 vcodec=hevc_amf rc=cqp qp_i=20 qp_p=20 qp_b=20 g=75 bf=2 acodec=aac ab=320k usage=transcoding quality=quality vprofile=main movflags=+faststart
```

**Linux (VAAPI)**

```shell
f=mp4 vcodec=hevc_vaapi vaapi_device=/dev/dri/renderD128 qp=20 g=75 bf=2 acodec=aac ab=320k vprofile=main movflags=+faststart
```


### AV1 {#av1}

**Windows (AMF)**

```shell
f=mp4 vcodec=av1_amf vprofile=main rc=cqp qp=24 g=75 acodec=aac ab=320k usage=transcoding quality=quality movflags=+faststart
```

**Linux (VAAPI)**

```shell
f=mp4 vcodec=av1_vaapi vaapi_device=/dev/dri/renderD128 qp=24 g=75 acodec=aac ab=320k movflags=+faststart
```


## 参数解析 {#参数解析}


### 共通参数 (Windows &amp; Linux 通用) {#共通参数--windows-and-linux-通用}

无论使用哪种操作系统，这些参数定义了视频的基础结构、画质策略和音频标准。

**容器与音频**

-   f=mp4 (封装格式)

    MP4 是兼容性之王。虽然 MKV 更灵活，但 MP4 能确保在 Windows、Mac、Android 和 iOS 上直接播放。

-   acodec=aac (音频编码器)

    使用 AAC (Advanced Audio Coding)，MP4 的标准音频格式。

-   ab=320k (音频码率)

    设置为 320 kbps。这是 AAC 的透明音质标准，接近无损听感，且体积占用极小。

**帧结构与 GOP**

-   g=75 (关键帧间隔 / GOP Size)

    计算逻辑: 针对 25 fps 项目，设置 75 表示 3秒 一个关键帧 (\\(75 \div 25 = 3\\))。

    作用: 3秒的间隔保证了拖动进度条时的流畅度，同时比短 GOP (如 1秒) 有更高的压缩效率。

-   bf=2 (B 帧数量 - 仅 H.265/H.264)

    含义: 在参考帧之间插入 2 个双向预测帧。

    重要性: AMD RDNA3 架构完美支持 HEVC B 帧。开启后，同画质下体积减少约 10%-20%。

    注意: AV1 目前通常由驱动自动管理 B 帧，一般不手动指定。

-   movflags=+faststart (Web 优化)

    将 MP4 索引移动到文件头。实现微信、浏览器中的视频“秒开” (边下边播)。

**画质基准 (Rate Control)**

-   qp=24 / qp=20 / qp=20 (量化参数基准)

    AV1: 推荐 24。AV1 压缩率极高，24 已是视觉无损。

    H.265: 推荐 20。H.265 效率略低于 AV1，需降低 QP 值以维持同等画质。

    H.264: 推荐 20。H.264 压缩率较低，需要较低 QP 值以维持画质。

<!--listend-->

-   vprofile=main / vprofile=high (编码规格)

    H.265/H.264: 显式指定使用 Main Profile (8-bit) / High Profile (H.264)，确保最大兼容性。解决 Kdenlive 参数命名冲突的标准写法。


### Windows 专用参数 (AMF 引擎) {#windows-专用参数--amf-引擎}

这些参数专门调用 AMD 官方闭源驱动中的 Advanced Media Framework (AMF) 功能。

**核心编码器**

-   vcodec=av1_amf / vcodec=hevc_amf / vcodec=h264_amf

    调用 Windows 驱动内置的硬件编码接口。

**AMF 独占优化**

-   rc=cqp (码率控制模式)

    显式声明使用“恒定量化参数”模式。

-   qp_i= / qp_p= / qp_b= (精细化质控制 - 仅 H.265/H.264)

    作用: 在 AMF 中，分别指定 I帧、P帧、B帧的压缩量。

    建议: 全部设为 20，防止画面在动态变化时出现"呼吸效应"（清晰度波动）。

<!--listend-->

-   usage=transcoding (场景提示)

    告诉显卡当前任务是"转码/导出"。显卡会分配更多显存进行预读分析，优先保证画质而非低延迟。

-   quality=quality (算法预设)

    强制启用最高质量的编码算法 (相比 speed 档位)。


### Linux 专用参数 (VAAPI 引擎) {#linux-专用参数--vaapi-引擎}

这些参数调用 Linux 内核标准的 Video Acceleration API (VAAPI)，配合 Mesa 开源驱动使用。

**核心编码器**

-   vcodec=av1_vaapi / vcodec=hevc_vaapi / vcodec=h264_vaapi

    调用 Linux 原生硬件接口。不要在 Linux 下强行使用 \_amf，除非安装了专业版闭源驱动。

**VAAPI 独占配置**

-   vaapi_device=/dev/dri/renderD128 (设备路径)

    必须项。明确告诉 FFmpeg 显卡在哪里。对于 7840HS 核显，通常是 renderD128。

-   qp=20 (简化的画质控制)

    VAAPI 的 CQP 模式比 AMF 智能，通常只需要指定一个全局 qp 值，驱动会自动计算 I/P/B 帧的权重，无需手动拆分。
