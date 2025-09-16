+++
title = "kdenlive 制作切片"
author = ["wang1zhen"]
date = 2025-09-16T00:00:00+09:00
draft = false
+++

## 工具与素材准备 {#工具与素材准备}

-   剪辑：Kdenlive
-   字幕生成：Whisper（large-v3 模型）
-   字幕美化：Aegisub
-   AI 分离：Demucs（--two-stems=vocals）

音频统一规格：48 kHz，双声道，WAV。

```bash
# 从录播提取音频
ffmpeg -i record.mp4 -vn -ac 2 -ar 48000 record.wav
```


## 人声与伴奏分离 {#人声与伴奏分离}

使用 Demucs two-stems：

```bash
demucs --two-stems=vocals record.wav
```

输出目录：./separated/htdemucs/record/

-   vocals.wav → 人声轨
-   no_vocals.wav → 伴奏轨


## 自动字幕生成（Whisper large-v3） {#自动字幕生成-whisper-large-v3}

仅对人声生成字幕：

```bash
whisper vocals.wav --model large-v3 --language ja --task transcribe --output_format srt
```

-   vocals.wav → 输入音频文件（人声轨）
-   --model large-v3 → 使用精度最高的模型
-   --language ja → 指定语言（日语，中文 zh，英文 en）
-   --task transcribe → 转写模式（保持原语言）
-   --output_format srt → 输出带时间戳的 .srt

输出：vocals.srt


## 字幕美化 {#字幕美化}

1.  在 Aegisub 打开 vocals.srt。
2.  创建样式：

<!--listend-->

```ass
Style: SongStyle,Noto Sans CJK JP,48,&H00FFFFFF,&H000000FF,&H00FACE87,&H64000000,0,0,0,0,100,100,0,0,1,3,1,2,10,10,30,1
```

-   白字 &amp;H00FFFFFF
-   浅蓝描边 &amp;H00FACE87
-   Outline=3，Shadow=1
-   Alignment=2（底部居中）

-   应用样式到全部字幕行。
-   导出 .ass 文件。


## 时间轴混音（Kdenlive） {#时间轴混音-kdenlive}

轨道布局：

-   A1：人声 (vocals.wav)
-   A2：伴奏 (no_vocals.wav)

混音规则：

-   唱歌时：A1 + A2（A2 音量比 A1 低 3–6 dB）
-   聊天时：A2 保留，A1 静音

静音方法：

-   在需要静音的片段，选中 A1 → Timeline → Add effect → Volume and Dynamics → Mute。


## 导入字幕 {#导入字幕}

1.  在 Kdenlive 添加字幕轨。
2.  导入 .ass 文件。
3.  确认预览，导出时会用 libass 渲染特效。


## 导出成品 {#导出成品}

1.  在 Kdenlive 渲染时，选择内嵌字幕（硬字幕）。
2.  导出设置：
    -   视频：H.264，保持原分辨率和帧率
    -   音频：AAC 320 kbps
