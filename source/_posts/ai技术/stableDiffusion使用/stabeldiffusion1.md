---
title: stablediffusion使用(1) -- 安装试用
date: 2026/05/24
tags:
- stablediffusion
---

# StableDiffusion安装

github地址[GitHub - CompVis/stable-diffusion: A latent text-to-image diffusion model · GitHub](https://github.com/CompVis/stable-diffusion)

本文以stablediffusion 0.1.5为例

## 安装

需要配合conda使用，可以很容易地创建stablediffusion依赖的python环境。

### 创建虚拟环境

如果`stable-diffusion`目录下已经有了之前创建的虚拟环境，先删除，之前创建的环境配置可能失效了。

可以先试一下

```bash
# 此命令列出有没有已经创建的环境
conda env list
# 发现有一个已经存在的ldm，可以启用
conda activate ldm
# 发现没用，先exit退出当前环境，然后直接删了
# 注意不是删除stablediffusion下的ldm目录，那个是依赖的库，不是虚拟环境目录
conda env remove --name ldm
```

如果没用，删除，重新创建，stablediffusion工程目录下有共享的conda环境配置`environment.yaml`

使用此文件可以直接在stablediffusion目录下执行如下命令，创建需要的虚拟环境

```bash
conda env create -f environment.yaml
```

如果报错，应该是安装超时了，修改仓库的源或者配置一下代理可以解决

```bash
set HTTP_PROXY=xxxxxxx
set HTTPS_PROXY=xxxxxxx
```

创建好环境后使用如下命令启用刚创建的环境

```bash
conda activate ldm
```

启用环境时，可能报如下错误

```bash
Traceback (most recent call last):
  File "G:\tools\miniconda\Scripts\conda-script.py", line 11, in <module>
    from conda.cli import main
ImportError: cannot import name 'main' from 'conda.cli' (unknown location)
```

这是由于本地已经配置了`PYTHON_HOME`环境变量，需要在命令行将其临时置空，否则conda使用的python会从错误的地方搜索依赖库。

```bash
set PYTHONHOME=
set PYTHONPATH=
conda activate ldm
```



### 验证运行

接下来通过如下命令进行验证

```bash
python scripts/txt2img.py --prompt "a professional photograph of an astronaut riding a horse" --ckpt <path/to/768model.ckpt/> --config configs/stable-diffusion/v2-inference-v.yaml --H 768 --W 768
```

第一次运行时会下载一些模型数据之类的，需要访问huggingface网站，国内一般连接不上，此时可以通过国内镜像网站下载

网上搜索到的镜像网站有如下这些

<table><thead><tr><th>镜像源</th><th>地址</th><th>特点</th><th>适用场景</th><th>稳定性</th></tr></thead><tbody><tr><td><strong>hf-mirror.com</strong></td><td>https://hf-mirror.com</td><td>社区驱动官方镜像，支持全API，高速，断点续传</td><td>所有场景（优先推荐）</td><td>★★★★★</td></tr><tr><td>ModelScope（阿里）</td><td>https://huggingface.modelscope.cn</td><td>国内大厂维护，镜像同步快，提供在线GPU环境</td><td>国产模型/在线开发</td><td>★★★★☆</td></tr><tr><td>清华镜像</td><td>https://mirrors.tuna.tsinghua.edu.cn/huggingface</td><td>学术镜像，适合小文件，部分大模型可能同步延迟</td><td>学术研究/教育场景</td><td>★★★☆☆</td></tr><tr><td>GitCode AI</td><td>https://ai.gitcode.com/models</td><td>基于GitCode的镜像，支持git操作</td><td>开源项目协作</td><td>★★★☆☆</td></tr></tbody></table>

通过设置环境变量指定镜像网站以及下载位置，如下

```bash
set HF_ENDPOINT=https://hf-mirror.com
set HF_HOME=E:\stablediffusion_cache
```

设置好后继续运行测试，可能报如下错误

> size mismatch for model.diffusion_model.output_blocks.11.1.transformer_blocks.0.attn2.to_v.weight: copying a param with shape torch.Size([320, 768]) from checkpoint, the shape in current model is torch.Size([320, 1024])

这是由于使用的模型的版本和配置的版本对应不上导致的，经验证如下名人模型和配置的版本是可以对应上的

```bash
call python scripts/txt2img.py --prompt "1girl" --config ../stable-diffusion-webui/configs/v1-inference.yaml --H 512 --W 512 --ckpt ChilloutMix-ni-fp16.ckpt --device cpu
```

这时候还可能报如下错误

```bash
OSError: Can't load tokenizer for 'openai/clip-vit-large-patch14'. If you were trying to load it from 'https://huggingface.co/models', make sure you don't have a local directory with the same name. Otherwise, make sure 'openai/clip-vit-large-patch14' is the correct path to a directory containing all relevant files for a CLIPTokenizer tokenizer.
```

解决方案是在前文列出的镜像网站里找到`openai/clip-vit-large-patch14`通过git或其他方法直接下载到本地stabeldiffusion的`open/clip-vit-large-patch14`目录下

```bash
git clone https://atomgit.com/hf_mirrors/openai/clip-vit-large-patch14.git
```

之后继续运行，又出现如下错误

```bash
RuntimeError: expected scalar type BFloat16 but found Float
```

这是由于数据类型不匹配导致的，需要在命令行后添加`--precision full`

最终生成成功

### 使用显卡运行

前文的命令，生成图片时，使用的是CPU， 效率相对较低，想要使用显卡生成图片，需要使用参数`--device cuda`，前提是你有英伟达的显卡，以及你本地已经安装好了cuda版本的torch。

以windows系统，rtx5070ti举例

先在命令行使用如下命令查询cuda版本

```bash
nvidia-smi
```

可以看到cuda version是`13.2`

之后需要在[pytorch网站](https://pytorch.org/get-started/locally)上找到你的cuda对应版本的torch包。

使用如下命令重新安装

```bash
pip3 install torch==2.12.0+cu132 torchvision==0.27.0+cu132 torchaudio --index-url https://download.pytorch.org/whl
```

安装的时候发现装不了，这时候才发现`pytorch 2.12.0`需要python 3.10以上，之前创建的虚拟环境是3.8的，以及之前`environment.yaml`配置的依赖版本和3.10有很多冲突，只能完全自己手动重新安装依赖，需要执行如下步骤

```bash
# 创建python 3.10的环境
conda create -n ldm python=3.10
# 启用环境
conda activate ldm
# 安装依赖的numpy
conda install numpy
# 安装所有需要的依赖，其中pytorch-lightnin和torchmetrics必须使用指定的版本，否则代码会报错
pip install albumentations opencv-python imageio imageio-ffmpeg pytorch-lightning==1.4.2 omegaconf test-tube streamlit einops transformers webdataset kornia open_clip_torch streamlit-drawable-canvas torchmetrics==0.6.0 invisible-watermark -e .
# 安装torch gpu版本，并且使用阿里云镜像
pip3 install torch==2.12.0+cu132 torchvision==0.27.0+cu132 torchaudio -f https://mirrors.aliyun.com/pytorch-wheels/cu132
# 不安装这个会报错triton not found; flop counting will not work for triton kernels
pip install -U "triton-windows<3.7"
```

最终执行下面的命令，就能正常生成了

```bash
python scripts/txt2img.py --prompt "1girl" --config ../stable-diffusion-webui/configs/v1-inference.yaml --H 512 --W 512 --ckpt ChilloutMix-ni-fp16.ckpt --device cuda
```

