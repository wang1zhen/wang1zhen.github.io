+++
title = "Linux 下 fio 测试硬盘读写性能"
author = ["wang1zhen"]
description = "在 Linux 下 用 fio 来测试硬盘的实际读写性能（替代 DCrystal Disk Mark）"
date = 2025-09-18T00:00:00+09:00
draft = false
+++

## 核心选项说明 {#核心选项说明}

-   --name: 任务名
-   --rw: I/O 模式
    -   read: 顺序读
    -   write: 顺序写
    -   rw: 顺序读写交替
    -   randread: 随机读
    -   randwrite: 随机写
    -   randrw: 随机读写交替
-   --bs: 块大小（block size），如 4k, 1M
-   --size: 测试文件大小
-   --filename: 目标文件或设备路径
-   --runtime: 测试时长
-   --time_based: 基于时间运行，而不是直到写满 size
-   --group_reporting: 汇总结果
-   --numjobs: job 数量，即多少线程/进程同时运行相同任务
-   --iodepth: 每个 job 可挂起的并发 I/O 请求数（队列深度）
    -   numjobs × iodepth ≈ 总并发请求数


## 准备测试文件 {#准备测试文件}

```sh
fio --name=prepare --rw=write --bs=1M --size=1G --filename=testfile --numjobs=1 --group_reporting
```


## 顺序吞吐量测试（CDM Seq QD=8 T=1） {#顺序吞吐量测试-cdm-seq-qd-8-t-1}


### 顺序读 {#顺序读}

```sh
fio --name=seqread --rw=read --bs=1M --size=1G --filename=testfile --numjobs=1 --iodepth=8 --runtime=30 --time_based --group_reporting
```


### 顺序写 {#顺序写}

```sh
fio --name=seqwrite --rw=write --bs=1M --size=1G --filename=testfile --numjobs=1 --iodepth=8 --runtime=30 --time_based --group_reporting
```


## 随机延迟测试（CDM Rnd4K QD=1 T=1） {#随机延迟测试-cdm-rnd4k-qd-1-t-1}


### 随机读 {#随机读}

```sh
fio --name=latread --rw=randread --bs=4k --size=1G --filename=testfile --numjobs=1 --iodepth=1 --runtime=60 --time_based --group_reporting
```


### 随机写 {#随机写}

```sh
fio --name=latwrite --rw=randwrite --bs=4k --size=1G --filename=testfile --numjobs=1 --iodepth=1 --runtime=60 --time_based --group_reporting
```


## 随机 IOPS 高并发（CDM Rnd4K QD=32 T=1） {#随机-iops-高并发-cdm-rnd4k-qd-32-t-1}


### 随机读 {#随机读}

```sh
fio --name=randread --rw=randread --bs=4k --size=1G --filename=testfile --numjobs=1 --iodepth=32 --runtime=60 --time_based --group_reporting
```


### 随机写 {#随机写}

```sh
fio --name=randwrite --rw=randwrite --bs=4k --size=1G --filename=testfile --numjobs=1 --iodepth=32 --runtime=60 --time_based --group_reporting
```


## 随机 IOPS 超高并发（CDM Rnd4K QD=32 T=16） {#随机-iops-超高并发-cdm-rnd4k-qd-32-t-16}


### 随机读 {#随机读}

```sh
fio --name=randread_mt --rw=randread --bs=4k --size=1G --filename=testfile --numjobs=16 --iodepth=32 --runtime=60 --time_based --group_reporting
```


### 随机写 {#随机写}

```sh
fio --name=randwrite_mt --rw=randwrite --bs=4k --size=1G --filename=testfile --numjobs=16 --iodepth=32 --runtime=60 --time_based --group_reporting
```


## 混合负载（模拟真实业务） {#混合负载-模拟真实业务}

```sh
fio --name=randrw --rw=randrw --rwmixread=70 --bs=4k --size=1G --filename=testfile --numjobs=4 --iodepth=32 --runtime=60 --time_based --group_reporting
```


## 清理 {#清理}

```sh
rm testfile
```
