---
title: 本地部署katago
date: 2026/08/22
tags:
- ai
- katago
---

# 先下载引擎

[katago release]([Releases · lightvector/KataGo](https://github.com/lightvector/KataGo/releases))

英伟达显卡，所以选的[katago-v1.17.2-trt10.16.1-cuda13.2-windows-x64.zip](https://github.com/lightvector/KataGo/releases/download/v1.17.2/katago-v1.17.2-trt10.16.1-cuda13.2-windows-x64.zip)版本。

使用tensorRT 10.16.1和cuda 13.2

需要在[TensorRT Download | NVIDIA Developer](https://developer.nvidia.com/tensorrt/download)下载TensorRT SDK，得先创建开发者账号

然后下载对应的[TensorRT 10.16.1 GA for Windows 10, 11, Server 2022 and CUDA 13.0 to 13.2 ZIP Package](https://developer.nvidia.com/downloads/compute/machine-learning/tensorrt/10.16.1/zip/TensorRT-10.16.1.11.Windows.amd64.cuda-13.2.zip)

> 版本列表还有个带bs50的包，那代表添加了对50*50大棋盘的支持

# 再下载模型

[Release OpenCL, Windows, TRT Bugfixes · lightvector/KataGo](https://github.com/lightvector/KataGo/releases/tag/v1.17.1)

按照自己的主机性能找合适的模型



# 配置环境变量

本地解压TensorRT后，将TensorRT的bin路径添加到Path环境变量，这样Katago可以搜索到所需dll库



# 执行

之后直接命令行执行`katago.exe`，可以查看到用法

之后使用

```bash
katago.exe benchmark -model models\b11c768h12nbt3tflrs-fson-silu.bin.gz
```

命令，可以测试建议的线程

执行后会输出一些建议，如下

```bash
Based on some test data, each speed doubling gains perhaps ~250 Elo by searching deeper.
Based on some test data, each thread costs perhaps 7 Elo if using 800 visits, and 2 Elo if using 5000 visits (by making MCTS worse).
So APPROXIMATELY based on this benchmark, if you intend to do a 5 second search:
numSearchThreads =  5: (baseline)
numSearchThreads = 10:  +162 Elo
numSearchThreads = 12:  +189 Elo
numSearchThreads = 16:  +244 Elo
numSearchThreads = 20:  +278 Elo
numSearchThreads = 24:  +279 Elo (recommended)
numSearchThreads = 32:  +272 Elo
numSearchThreads = 40:  +249 Elo
numSearchThreads = 64:  +184 Elo
```

可以看到运行结果建议我使用24线程，但还有另外的建议

```bash
Testing using 800 visits.
  If you have a good GPU, you might increase this using "-visits N" to get more accurate results.
  If you have a weak GPU and this is taking forever, you can decrease it instead to finish the benchmark faster.
```

我的显卡还蛮不错，再跑一次

```bash
katago.exe benchmark -model models\b11c768h12nbt3tflrs-fson-silu.bin.gz -visits 1600
```

试了一下还是800的默认访问次数合适6s左右，1600访问需要10s

接着生成配置文件

```bash
katago.exe genconfig -model models\b11c768h12nbt3tflrs-fson-silu.bin.gz
```

除了一个使用的规则和访问次数需要设置之外，其余皆可以使用默认配置，访问次数咱们已经在前面用测试命令测试出来了

# 客户端

引擎是没有客户端界面的，所以需要找一个推荐的客户端配合使用，或者自己用接口开发一个简易客户端也行，接下来介绍用接口开发简易客户端的思路

# 使用gtp接口

[GTP Command Interface | lightvector/KataGo | DeepWiki](https://deepwiki.com/lightvector/KataGo/5.1-gtp-command-interface)

