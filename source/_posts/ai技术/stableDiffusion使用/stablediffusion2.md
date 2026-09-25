---
title: stablediffusion使用(1) -- webui
date: 2026/05/27
tags:
- stablediffusion
---

# stablediffusion webui

使用命令行生成图片可能不太熟悉，可以通过webui使用

[GitHub - AUTOMATIC1111/stable-diffusion-webui: Stable Diffusion web UI · GitHub](https://github.com/AUTOMATIC1111/stable-diffusion-webui)

## 自动安装

按照github上readme的指示，执行如下步骤

```bash
# 源码克隆下来
git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git
# 创建独立的python环境
conda create -n webui python=3.10.8
# 启用环境
conda activate webui
# 设置下载的临时文件存放的位置，不舍的话默认是往C盘下，占空间还蛮多的
set XDG_CACHE_HOME=E:\stablediffusion_cache\xdg-cache-home
# 执行webui-user.bat，这个脚本在第一次运行时会自动创建需要的环境
webui-user.bat
```

这个安装步骤有几个可以修改的点是

1. 第一次执行时，可以直接创建虚拟环境后，自行根据`requirements.txt`中的内容安装依赖，然后指定`VENV_DIR`环境变量后，再直接执行`webui.bat`这样，会自动替你跳过环境准备的步骤。
2. 可以参考`modules\launch_utils.py`自行准备好需要的环境后，再尝试执行`webui.bat`

### 安装过程中出现的问题

1.  ModuleNotFoundError: No module named 'pkg_resources'

   这个错实在launch_utils.py中执行`pip3 install https://github.com/openai/CLIP/archive/d50d76daa670286dd6cacf3bcd80b5e4823fc8e1.zip --prefer-binary `命令时报的错。

   找了半天没找到原因，网上一直说是setuptools版本不对，但是试了改setuptools版本照样报错，查看错误堆栈发现运行的时候用的不是venv下的site-packages中的setuptools而是在用户目录下创建了一个临时目录，于是手动执行`pip3 install https://github.com/openai/CLIP/archive/d50d76daa670286dd6cacf3bcd80b5e4823fc8e1.zip --prefer-binary `并且添加参数`--no-build-isolation`，竟然成功了。

2. RuntimeError: Couldn't clone Stable Diffusion.和RuntimeError: Couldn't clone Stable Diffusion XL

   对应的git仓似乎都不存在了，直接在launch_utils.py中修改仓地址，改成其他人fork的，如下图两个地址对应的仓库都没了
   
   ![image-20260528221248269](.\assets\image-20260528221248269.png)

## 修改启动参数

可以按自己的需要修改webui-user.bat中的一些参数，来决定启动表现

一些具体的参数说明参考此文档[命令行参数 - Stable Diffusion WebUI使用手冊 - Stable Diffusion文档中心](https://docs.stablediffusion.cn/article/1/593258220479315968.html)

```bash
@echo off
# 避免环境冲突
set PYTHON=
set GIT=
set VENV_DIR=
set COMMANDLINE_ARGS=

# 不启用这个加载模型时会报错
set TORCH_FORCE_NO_WEIGHTS_ONLY_LOAD=1

# 选择是否启用xformers
@REM set XFORMERS=--xformers

# 使用的配置文件
set CONFIG_FILE="%cd%\repositories\stable-diffusion-stability-ai\configs\stable-diffusion\v2-inference-v.yaml"

# --no-half --no-half-vae --disable-nan-check三个参数用来检查显卡精度不足导致的图片生成问题
if not '%NO_HALF_FLAG%'=='' (
    set NO_HALF=--precision full --no-half --no-half-vae --disable-nan-check 
) else (
    set NO_HALF=
)

# skip-install跳过一些安装步骤
set COMMANDLINE_ARGS=--medvram --skip-install %NO_HALF%

call webui.bat --config %CONFIG_FILE%

```

## 编译安装xfromers

因为显卡比较新，xformers不支持，使用xformers启动时会报错，因此需要手动编译xformers

使用如下命令下载源码后安装

```bash
git clone https://github.com/facebookresearch/xformers.git --recursive
cd xformers
# 如果前面clone xformers时未添加--recursive参数时，可以再次执行如下命令下载子模块
git submodule update --init --recursive
python setup.py install
```

或者直接一键下载编译安装

```bash
pip install -v --no-build-isolation -U git+https://github.com/facebookresearch/xformers.git@main#egg=xformers
```

编译前需要先准备编译工具链，下载vs community安装如下组件

![image-20260530013826592](.\assets\image-20260530013826592.png)

后续再次执行`webui-user.bat`还是会报错`xformers.ops`上找不到`memory_efficient_attention`

查看源码发现，虽然安装了mslk，但是`importlib.util.find_spec("mslk")`查不到

```bash
if _HAS_MSLK := importlib.util.find_spec("mslk") is not None:
    print('has mslk')
    from .fmha import (
        AttentionBias,
        AttentionBias as AttentionMask,
        AttentionOp,
        AttentionOpBase,
        LowerTriangularMask,
        memory_efficient_attention,
        memory_efficient_attention_backward,
        memory_efficient_attention_forward,
        memory_efficient_attention_forward_requires_grad,
        MemoryEfficientAttentionCkOp,
        MemoryEfficientAttentionCutlassFwdFlashBwOp,
        MemoryEfficientAttentionCutlassOp,
        MemoryEfficientAttentionFlashAttentionOp,
        MemoryEfficientAttentionSplitKCkOp,
    )
else:
    print('has no mslk')
```

通过pip卸载mslk在手动编译安装一份即可

```bash
git clone https://github.com/meta-pytorch/MSLK.git
# 不设置会报错
set MSLK_PYTHON_ONLY=1
python setup.py install
```

再次执行又报如下错误

> triton.runtime.errors.OutOfResources: out of resource: shared memory, Required: 147456, Hardware limit: 101376. Reducing block sizes or `num_stages` may help

询问AI得知如下结论

> 需要144kb共享内存，但是5070ti最大只能有128kb共享内存，安全范围是100kb以内，内存超了

因此只能修改`mslk\attention\fmha\triton_splitk.py`源码重新安装

![image-20260530050423041](C:\Home\WorkSpace\Document\Markdown\source\_posts\ai技术\stableDiffusion使用\assets\image-20260530050423041.png)

另外ai还建议

> **移除 `mslk` 加速（最推荐）：**
> 在 `webui-user.bat` 中，添加强行切换到 PyTorch 2.x 原生高效注意力机制的命令，它几乎完全不输 Triton，且针对消费级显卡做了完美兼容：
>
> ```
> set COMMANDLINE_ARGS=--opt-sdp-attention --disable-safe-unpickle
> ```
>
> **全局禁用 Triton 缓存（针对此 WebUI 环境）：**
> 如果你之前在 `webui-user.bat` 中加入了 `--no-half-vae` 或者开启了额外的编译优化，请暂时删掉它们，只保留 `--xformers` 或 `--opt-sdp-attention`。
