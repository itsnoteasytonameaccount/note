---
title: stable diffusion使用
tags:
- python
- stable-diffusion
date: 2023/03/26
---

# 安装conda

stable diffusion依赖这个工具来安装环境

## 用pip安装

第一次报错` Building wheel for conda (setup.py) ... error...`

+ 解决方案

需要更新pip

```python
python -m pip install --upgrade pip 
```

再次报错

`no module name auxlib`

+ 解决方案

安装`python install auxlib`

最后`python install conda`完成

最后发现根本没有`conda env create命令`，因为根本不是最新版，原因是通过`python install conda`只能安装到`4.6`版本，`4.10`才有`conda env create`

## 只能用下载安装包的方式安装

刚开始一直报错

要么是`conda env`不存在

要么是`from conda.cli import main`提示找不到`conda.cli`，后来发现是因为环境变量中的`pythonhome`影响

**运行conda前必须确保环境中没有`pythonhome`环境变量；或者确保`pythonhome`环境变量以及`path`环境变量中的python位置一致.**

> 注意这个又会有一个问题。ida运行前需要确保有这个环境变量，这完全冲突了

如果conda命令执行时报错，实在不知什么原因，大概类似如下脚本配置一下环境变量即可，如下脚本缺点是会影响当前命令行环境变量

```bash
# conda_preset.bat
@echo off
set PYTHONHOME=D:\tools\miniconda
set CONDABIN_PATH=%PYTHONHOME%\condabin
set PATH=%PYTHONHOME%;%PYTHONHOME%\Scripts;%PYTHONHOME%\Lib;%PATH%
set PYTHONHOME=
```

最后需要执行的是condabin文件夹下的`conda.bat`文件

# stable-diffusion

## 安装

从git下载仓库

[1.0版本](https://github.com/CompVis/stable-diffusion)

[2.0版本]()

后文都是基于2.0版本进行，如果是1.0可能不同

## 启动

1. 创建虚拟环境

   ```bash
   # env这个命令不知道怎么回事，官方文档里没有，但是能用，其他教程里也都有
   conda create env -f environment.yaml
   # 如果中途报错可以修正错误后通过update更新环境
   conda update env -f environment.yaml
   # 如果想删除环境可以执行如下命令,ldm是你创建的环境名
   conda env remove -n ldm
   ```

2. 启动虚拟环境

   ```bash
   conda activate ldm
   ```
   
3. 下载模型

   常用的下载模型的网站：[huggingface](https://huggingface.co/models)、[civitai(需翻墙)](civitai.com)

   具体的模型说明可以查看[模型特点](#模型特点)一节

   目前常见的模型都变成了safetensors格式的，但stable-diffusion默认只能读取ckpt文件，因此想要使用的话需要了解safetensors文件如何读取的，如下是safetensors和ckpt互相转化的代码

   ```python
   import torch
   import sys
   import os
   import getopt
   from safetensors.torch import load_file, save_file
   
   MODE_TRANS2_CKPT = 0
   MODE_TRANS2_SAFETENSORS = 1
   
   checkpoint_dict_replacements = {
       'cond_stage_model.transformer.embeddings.': 'cond_stage_model.transformer.text_model.embeddings.',
       'cond_stage_model.transformer.encoder.': 'cond_stage_model.transformer.text_model.encoder.',
       'cond_stage_model.transformer.final_layer_norm.': 'cond_stage_model.transformer.text_model.final_layer_norm.',
   }
   
   def transform_checkpoint_dict_key(k):
       for text, replacement in checkpoint_dict_replacements.items():
           if k.startswith(text):
               k = replacement + k[len(text):]
   
       return k
   
   def safetensors2ckpt(path, outdir='./'):
       names = path.split(os.sep).pop().split('.')
       weights = {}
       pl_sd = load_file(path, device='cuda')
       sd = {}
       for k, v in pl_sd.items():
           new_key = transform_checkpoint_dict_key(k)
   
           if new_key is not None:
               sd[new_key] = v
       
       pl_sd.clear()
       pl_sd.update(sd)
       weights['state_dict'] = pl_sd
       torch.save(weights, outdir + names[0] + '.ckpt')
       pass
   
   
   def ckpt2safetensors(path, outdir='./'):
       names = path.split(os.sep).pop().split('.')
       pl_sd = torch.load(path, device='cuda')['state_dict']
       save_file(pl_sd, outdir + names[0] + '.safetensors')
       pass
   
   
   def readCkptInfo(path):
       pl_sd = torch.load(path, device='cuda')
       print(pl_sd.keys())
   
   
   def readSafetensorsInfo(path):
       pl_sd = load_file(path, device='cuda')
       print(pl_sd.keys())
   
   
   if __name__ == '__main__':
       opts, args = getopt.getopt(
           sys.argv[1:], '-p:-o:-m:-i', ['mode=', 'path=', 'outdir=', 'info'])
       path = ''
       mode = MODE_TRANS2_CKPT
       outdir = './'
       infoFlag = False
       for name, value in opts:
           if name in ('-i', 'info'):
               infoFlag = True
           elif name in ('-p', 'path'):
               path = value
           elif name in ('-m', 'mode') and int(value) in (MODE_TRANS2_CKPT, MODE_TRANS2_SAFETENSORS):
               mode = int(value)
           elif name in ('-o', 'outdir'):
               outdir = value
       if mode == MODE_TRANS2_CKPT:
           if infoFlag:
               readSafetensorsInfo(path)
           else:
               safetensors2ckpt(path, outdir)
       elif mode == MODE_TRANS2_SAFETENSORS:
           if infoFlag:
               readCkptInfo(path)
           else:
               ckpt2safetensors(path, outdir)
   
   ```

4. 测试生成图片

   上述内容全部准备完后就可以尝试通过命令行生成一次图片了,  github上给出的生成命令示例是

   ```bash
   python scripts/txt2img.py --prompt "a professional photograph of an astronaut riding a horse" --ckpt <path/to/768model.ckpt/> --config configs/stable-diffusion/v2-inference-v.yaml --H 768 --W 768
   ```

   可以通过`python scripts/txt2img,py --help`查看更多相关选项

   > 执行过程中会根据你通过`--config`指定的配置文件下载一个比较大的模型文件，会用你通过`--ckpt`指定的checkpoint和下载的模型对比，如果报错提示"从checkpoint读取到xxx，但是从本地模型读取到xxx"，一般是你的config文件和模型的版本不匹配

# 安装webui

为了简单体验stable-diffusion的功能，通过webui试试

[webui github链接](https://github.com/AUTOMATIC1111/stable-diffusion-webui)

将stable-diffusion-ai放入webui的repositories目录后

按照webui github readme指示执行webui-user.bat

## virtualenv

由于webui依赖python 3.10版本，所以需要用virtualenv生成一个虚拟环境

```bash
pip install virtualenv
# venv是你要给虚拟环境取得名
virtualenv venv --python=你的3.10版本的python位置
# 启动虚拟环境
venv/Scripts/activate
```

之后的命令都应当在这个环境执行

虚拟环境创建好后最好修改一下huggingface的缓存路径，否则你的c盘会被写入很多的缓存文件，鉴于C盘一般容量较小，很可能被占满

进入`venv\Scripts\activate.bat`添加如下行配置环境变量

```bat
set HUGGINGFACE_HUB_CACHE=你的缓存存放位置

@REM 或者直接设置一个总的
set XDG_CACHE_HOME=torch,huggingface等的缓存根目录
```



## 第一次启动

第一次执行会安装很多东西，同时也会创建一个名为venv的虚拟环境，之后若是为webui安装依赖都需要在这个虚拟环境下

安装过程中可能会因为网络问题中断，一般分为如下情况，通过对应手段解决后重新执行webui-user.bat即可

1. 访问不到源仓库地址

   解决方案是翻墙或换源，命令如下

   ```bash
   # 切换为清华源
   pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
   # 重置为默认源
   pip config set global.index-url https://pypi.Python.org/simple
   ```

2. 从git下载文件失败

​       解决方案是翻墙或者手动下载后放置在特定的目录，下载过程中报错会提示**从xxxx地址下载到xxxxx路径失败**，此时可以考虑手动从对应地址将对应文件下载后存放在对应的路径下

3. ` Could not find a version that satisfies the requirement xxxx`

​       一般是由于网络问题导致读取源仓库里对应依赖可选版本失败、或者你使用的源确实没有需要的版本的依赖，那么就只能将源设置为默认源再将翻墙试试了

4. 需要升级pip版本

   ```bash
   python -m pip install --upgrade pip
   ```

# 安装xformers

选择性安装（注意如果是为了给webui安装请启用正确的虚拟环境）

>启用xformers对大图及以上有极为巨大的性能提升，极大幅度降低显存占用，并且有很大可能修复黑图。代价是部分细节可能有些许改变（有可能有巨大改变）。如果不是非常吹毛求疵，强烈推荐使用xformers生成高分辨率的图片。

>  即使同一种子、同一参数，使用xFormers后每次出的图都有点变化，主要是物件的边缘。但是和显卡型号的改变一样，只是细节上一些变化，不对比看不出来。请自行衡量能不能接受这些改变。
> 引自: [xFormers安装使用](https://blog.csdn.net/watson2017/article/details/129358022)

stable-diffusion的readme让自行编译安装，实际上不必要，而且更容易出错，如下通过编译安装的xformers0.0.18版本，生成图片时直接报错

![image-20230330201232798](.\assets\image-20230330201232798.png)

实际上直接通过pip命令安装即可

```bash
# 只能是0.0.16及之前版本，其他版本会导致torch版本冲突
pip install -v xformers==0.0.16
```

注意如果安装xformers时未指定版本导致正确版本的torch被卸载，需要手动重新安装

可以参照[参考](#参考)中给出的文章的指示安装(nvidia显卡情况下)

1. nvidia-smi，查看 cuda 信息
2. [此处](https://download.pytorch.org/whl/torch_stable.html)下载对应版本的torch
3. 执行pip install 安装whl文件

# webui相关概念介绍

## 模型特点

以下仅为本人理解，

模型分为很多类别，常见的是lora,checkpoint,vae等，这些模型是无法通过后缀区分的(后缀如ckpt和safetensors只代表数据存储的格式，不代表模型类型)，想要区分类型的话需要你在下载模型时留意,一般模型提供者都会指出是什么模型的，或者也可以根据尺寸判断

1. lora：这个一般体积较小，一般是用来修改要输出的图片内容的，比如生成的人物有特定样貌之类的。大小几十兆到几百兆不等
2. checkpoint: 这个就是模型本体，决定你能生成什么样的图片，大小1G以上
3. vae这个类似于滤镜，大小几百兆左右

webui中可选择的模型

![image-20230330225815625](.\assets\image-20230330225815625.png)

![image-20230330225924673](.\assets\image-20230330225924673.png)

# 错误处理

1. RuntimeError: Expected all tensors to be on the same device, but found at least two devices,

   这个错误可能的情况是你的prompt中有lora,但是对应的lora没有加载到gpu里而是在内存中，将prompt中的lora去掉，或者重新启动webui。导致这种现象的原因可能是在启用lora插件的情况下，点击了apply and restart ui.

   ![image-20230405010824305](.\assets\image-20230405010824305.png)

# 参考

1. [Cuda 和 GPU版torch安装最全攻略，以及在GPU 上运行 torch代码](https://blog.csdn.net/weixin_46833029/article/details/127946022)

2. [StableDiffusion模型资源探索食用指南](https://zhuanlan.zhihu.com/p/597504900)

3. [【AI绘画】全部模型种类总结 / 使用方法 / 简易训练指导——魔导绪论](https://www.bilibili.com/read/cv21362202)

4. [conda 相关命令，conda环境创建，跑通代码](https://blog.csdn.net/m0_63160525/article/details/127818163)