---
title: opengl(12):hdr渲染，泛光
date: 2021/06/15
tags:
  - opengl
---

为学懂 webgl，从 opengl 入手。本文内容主要来自[LearnOpenGL CN](https://learnopengl-cn.github.io/)文档，加上个人理解，以及一些细节补充、经验记录。本人学习过程中的详细代码参见[此处](https://github.com/itsnoteasytonameaccount/OpenGL-learning)

要学习推荐看原文[LearnOpenGL CN](https://learnopengl-cn.github.io/)，有评论区，加载评论区需要翻墙

另可在[此处](https://www.khronos.org/opengl/wiki/Getting_Started)找到其他 opengl 官方推荐的资料、教程网站

<!-- more -->

hdr和泛光这两部分没什么难点，了解一下直接照着教程代码写就可以了

# hdr(High Dynamic Range, 高动态范围)渲染

在做到opengl光照的时候，想必多少会有点疑问，我们的光照计算是叠加的

```glsl
	for(int i = 0; i < NR_POINT_LGIHTS; i++)
    {
        result += CalcLight(lights[i], Normal, viewDir, diffuseTexture, specularTexture);
    }
```

那么如果场景中有非常多的光源，颜色分量范围是$[0.0, 1.0]$，叠加结果超过白色$vec3(1.0f, 1.0f, 1.0f, 1.0f)$时，很多地方不就成纯白了。hdr就是用来解决这个问题。

## 介绍

> HDR原本只是被运用在摄影上，摄影师对同一个场景采取不同曝光拍多张照片，捕捉大范围的色彩值。这些图片被合成为HDR图片，从而综合不同的曝光等级使得大范围的细节可见。看下面这个例子，左边这张图片在被光照亮的区域充满细节，但是在黑暗的区域就什么都看不见了；但是右边这张图的高曝光却可以让之前看不出来的黑暗区域显现出来。

> 这与我们眼睛工作的原理非常相似，也是HDR渲染的基础。当光线很弱的啥时候，人眼会自动调整从而使过暗和过亮的部分变得更清晰，就像人眼有一个能自动根据场景亮度调整的自动曝光滑块。

借用hdr思想，hdr渲染的原理则是先允许颜色计算结果超过$[0.0, 1.0]$这个范围，保留所有颜色细节，再通过一定的算法将颜色重新映射到范围内。

## 步骤

1. 先将绘制出场景颜色信息存放再特定的允许颜色范围超过$[0.0, 1.0]$的缓冲中（渲染hdr场景）。
2. 绘制占满窗口的矩形，将步骤一中的场景颜色映射到$[0.0, 1.0]$，并绘制到矩形上（hdr场景映射到ldr（低动态范围）场景）。

## 浮点帧缓冲

浮点帧缓冲用来允许颜色范围超过$[0.0, 1.0]$

> 当一个帧缓冲的颜色缓冲的内部格式被设定成了GL_RGB16F, GL_RGBA16F, GL_RGB32F 或者GL_RGBA32F时，这些帧缓冲被叫做浮点帧缓冲(Floating Point Framebuffer)，浮点帧缓冲可以存储超过0.0到1.0范围的浮点值，所以非常适合HDR渲染。

浮点帧缓冲的颜色附件创建如下

```glsl
glBindTexture(GL_TEXTURE_2D, colorBuffer);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGB, GL_FLOAT, NULL
```

>默认的帧缓冲默认一个颜色分量只占用8位(bits)。当使用一个使用32位每颜色分量的浮点帧缓冲时(使用GL_RGB32F 或者GL_RGBA32F)，我们需要四倍的内存来存储这些颜色。所以除非你需要一个非常高的精确度，32位不是必须的，使用GLRGB16F就足够了。

## 色调映射

简单的色调映射算法，Reinhard色调映射算法实现如下

```glsl
void main()
{             
    const float gamma = 2.2;
    vec3 hdrColor = texture(hdrBuffer, TexCoords).rgb;

    // Reinhard色调映射
    vec3 mapped = hdrColor / (hdrColor + vec3(1.0));
    // Gamma校正
    mapped = pow(mapped, vec3(1.0 / gamma));

    color = vec4(mapped, 1.0);
}   
```

可以在色调映射算法里设置一个曝光参数，通过这个参数决定映射表现，一个简单的曝光色调映射算法会像这样：

```glsl
uniform float exposure;

void main()
{             
    const float gamma = 2.2;
    vec3 hdrColor = texture(hdrBuffer, TexCoords).rgb;

    // 曝光色调映射
    vec3 mapped = vec3(1.0) - exp(-hdrColor * exposure);
    // Gamma校正 
    mapped = pow(mapped, vec3(1.0 / gamma));

    color = vec4(mapped, 1.0);
}  
```

## 最终成果

最终成果如下，不知道是不是代码有问题，我渲染出的效果比learn OpenGL教程上的亮一些，曝光系数设为0.2才有learn OpenGL曝光系数1.0的效果

<div style='text-align:center' id='img1-1'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/hdr.gif' alt='图片加载失败'/></p>
<p style='margin-bottom: 20px; line-height:1.5'><span style='font-size:12px'>图1-1: 最终成果</span></p>
</div>

# 泛光

> 明亮的光源和区域经常很难向观察者表达出来，因为监视器的亮度范围是有限的。一种区分明亮光源的方式是使它们在监视器上发出光芒，光源的光芒向四周发散。这样观察者就会产生光源或亮区的确是强光区的错觉。（译注：这个问题的提出简单来说是为了解决这样的问题：例如有一张在阳光下的白纸，白纸在监视器上显示出是出白色，而前方的太阳也是纯白色的，所以基本上白纸和太阳就是一样的了，给太阳加一个光晕，这样太阳看起来似乎就比白纸更亮了）

泛光就是用来产生光晕效果的，比较简单，没有特别难理解的。

泛光实现步骤如下：
1. 渲染hdr场景
2. 取出hdr场景超过一定亮度的片段组成纹理
3. 对步骤2中得到的新的纹理进行模糊处理，模糊处理对泛光效果的好坏起决定作用
4. 将hdr场景和模糊处理后的纹理组合，再映射到ldr

<div style='text-align:center' id='img2-1'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/bloom_steps.png' alt='图片加载失败'/></p>
<p style='margin-bottom: 20px; line-height:1.5'><span style='font-size:12px'>图2-1: 泛光实现步骤</span></p>
</div>

## 提取亮色

提取亮色可以使用一个叫多目标渲染的手段，一次渲染输出到两个颜色缓冲

此时着色器中需要设置一个像素着色器写入到哪个颜色缓冲

```glsl
#version 330 core
layout (location = 0) out vec4 FragColor;
layout (location = 1) out vec4 BrightColor;

[...]

void main()
{            
    [...] // first do normal lighting calculations and output results
    FragColor = vec4(lighting, 1.0f);
    // Check whether fragment output is higher than threshold, if so output as brightness color
    float brightness = dot(FragColor.rgb, vec3(0.2126, 0.7152, 0.0722));
    if(brightness > 1.0)
        BrightColor = vec4(FragColor.rgb, 1.0);
}
```

自定义帧缓冲应当绑定两个颜色缓冲

```glsl
// Set up floating point framebuffer to render scene to
GLuint hdrFBO;
glGenFramebuffers(1, &hdrFBO);
glBindFramebuffer(GL_FRAMEBUFFER, hdrFBO);
GLuint colorBuffers[2];
glGenTextures(2, colorBuffers);
for (GLuint i = 0; i < 2; i++)
{
    glBindTexture(GL_TEXTURE_2D, colorBuffers[i]);
    glTexImage2D(
        GL_TEXTURE_2D, 0, GL_RGB16F, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGB, GL_FLOAT, NULL
    );
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    // attach texture to framebuffer
    glFramebufferTexture2D(
        GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0 + i, GL_TEXTURE_2D, colorBuffers[i], 0
    );
}
```

我们需要显式告知OpenGL我们正在通过glDrawBuffers渲染到多个颜色缓冲，否则OpenGL只会渲染到帧缓冲的第一个颜色附件，而忽略所有其他的。我们可以通过传递多个颜色附件的枚举来做这件事，我们以下面的操作进行渲染：

```glsl
GLuint attachments[2] = { GL_COLOR_ATTACHMENT0, GL_COLOR_ATTACHMENT1 };
glDrawBuffers(2, attachments);
```

## 高斯模糊

接下来是进行模糊处理，由于模糊处理的效果很大程度上决定了最终的泛光效果，因此不采用之前提到的简单地周围颜色取均值的模糊效果，而是采用高斯模糊。

高斯模糊简化理解就是对模糊点为中心的所有像素点取加权均值作为模糊点的值，而加权的权重是由某个特定的正态分布函数决定的，中心点的权重最大。由于本人在校期间不学无术，理解止于此，再高深的东西就不懂了，不过理解到这就行了。

> 要实现高斯模糊过滤我们需要一个二维四方形作为权重，从这个二维高斯曲线方程中去获取它。然而这个过程有个问题，就是很快会消耗极大的性能。以一个32×32的模糊kernel为例，我们必须对每个fragment从一个纹理中采样1024次！

> 高斯方程有个非常巧妙的特性，它允许我们把二维方程分解为两个更小的方程：一个描述水平权重，另一个描述垂直权重。我们首先用水平权重在整个纹理上进行水平模糊，然后在经改变的纹理上进行垂直模糊。利用这个特性，结果是一样的，但是可以节省难以置信的性能，因为我们现在只需做32+32次采样，不再是1024了！这叫做两步高斯模糊。

每次应用模糊效果需要渲染一次。

着色器中代码如下，代码中权重取值的依据在[Efficient Gaussian blur with linear sampling](https://rastergrid.com/blog/2010/09/efficient-gaussian-blur-with-linear-sampling/)中给出，用二项式系数来计算。
可惜我看不太懂，现在后悔没学好大学数学已经晚了。正态分布最早是求二项分布的渐近公式时导出的，n越大，二项分布的柱状图顶部连线越接近正态分布，可以用二项分布近似正态分布。

```glsl
#version 330 core
out vec4 FragColor;
  
in vec2 TexCoords;

uniform sampler2D image;
  
uniform bool horizontal;
uniform float weight[5] = float[] (0.227027, 0.1945946, 0.1216216, 0.054054, 0.016216);

void main()
{             
    vec2 tex_offset = 1.0 / textureSize(image, 0); // gets size of single texel
    vec3 result = texture(image, TexCoords).rgb * weight[0]; // current fragment's contribution
    if(horizontal)
    {
        for(int i = 1; i < 5; ++i)
        {
            result += texture(image, TexCoords + vec2(tex_offset.x * i, 0.0)).rgb * weight[i];
            result += texture(image, TexCoords - vec2(tex_offset.x * i, 0.0)).rgb * weight[i];
        }
    }
    else
    {
        for(int i = 1; i < 5; ++i)
        {
            result += texture(image, TexCoords + vec2(0.0, tex_offset.y * i)).rgb * weight[i];
            result += texture(image, TexCoords - vec2(0.0, tex_offset.y * i)).rgb * weight[i];
        }
    }
    FragColor = vec4(result, 1.0);
}
```

不管要渲染多少次，创建两个帧缓冲就够了，一个帧缓从另一个帧缓冲的颜色缓冲纹理中读取场景数据，渲染到自己的颜色缓冲纹理，交替执行即可。参考代码如下，共渲染了10次，5次水平5次竖直高斯模糊。

```glsl
bool horizontal = true, first_iteration = true;
int amount = 10;
shaderBlur.use();
for (unsigned int i = 0; i < amount; i++)
{
    glBindFramebuffer(GL_FRAMEBUFFER, pingpongFBO[horizontal]); 
    shaderBlur.setInt("horizontal", horizontal);
    glBindTexture(
        GL_TEXTURE_2D, first_iteration ? colorBuffers[1] : pingpongBuffers[!horizontal]
    ); 
    RenderQuad();
    horizontal = !horizontal;
    if (first_iteration)
        first_iteration = false;
}
glBindFramebuffer(GL_FRAMEBUFFER, 0);
```

## 合并

最后的步骤就是组合模糊后的纹理和hdr场景纹理

```glsl
#version 330 core
out vec4 FragColor;
  
in vec2 TexCoords;

uniform sampler2D scene;
uniform sampler2D bloomBlur;
uniform float exposure;

void main()
{             
    const float gamma = 2.2;
    vec3 hdrColor = texture(scene, TexCoords).rgb;      
    vec3 bloomColor = texture(bloomBlur, TexCoords).rgb;
    hdrColor += bloomColor; // additive blending
    // tone mapping
    vec3 result = vec3(1.0) - exp(-hdrColor * exposure);
    // also gamma correct while we're at it       
    result = pow(result, vec3(1.0 / gamma));
    FragColor = vec4(result, 1.0);
}  
```

最终效果如下，好像要比learn OpenGL CN的要亮一些，一直不知道原因，找不到问题在哪

<div style='text-align:center' id='img2-2'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/bloom.png' alt='图片加载失败'/></p>
<p style='margin-bottom: 20px; line-height:1.5'><span style='font-size:12px'>图2-2: 泛光效果</span></p>
</div>

# 参考

1. [Learn OpenGL](http://learnopengl.com/#!Advanced-Lighting/Bloom)
1. [Learn OpenGL CN](https://learnopengl-cn.github.io/05%20Advanced%20Lighting/07%20Bloom/)
