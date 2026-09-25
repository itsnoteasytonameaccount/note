---
title: opengl(9):高级光照，gamma校正，阴影
date: 2021/03/30
tags:
  - opengl
---

为学懂webgl，从opengl入手。本文内容主要来自[LearnOpenGL CN](https://learnopengl-cn.github.io/)文档，加上个人理解，以及一些细节补充、经验记录。本人学习过程中的详细代码参见[此处](https://github.com/itsnoteasytonameaccount/OpenGL-learning)

要学习推荐看原文[LearnOpenGL CN](https://learnopengl-cn.github.io/)，有评论区，加载评论区需要翻墙

另可在[此处](https://www.khronos.org/opengl/wiki/Getting_Started)找到其他opengl官方推荐的资料、教程网站
<!-- more -->

# 前言

总体来说学习已经渐入佳境了，不像之前，想要跑起一个窗口都需要花上半天排除各种错误。但是接下来才是重头戏，提前预览了下后面的内容，相较起来前面的都算简单的，毕竟只要有个初高中的文化水平大概就能理解了。后面的东西能学完的话才算真正有点样子了，再接再励。

# 高级光照

冯氏光照有个问题是由于使用反射向量和视线方向向量的反向量点乘来计算镜面光，当反射方向和视线方向夹角大于 90 度时，镜面光会变为 0，小于 90 度，镜面光就不为 0。

但是这不符合现实情况，会出现如下两种问题

1. 当光源位于平面背面，视点位于平面正面时，理论上说，光线是被遮挡的，但按照冯氏光照的计算方法，只要反射光与视线方向形成锐角，镜面分量就不为 0，如下，由于设置了光照半径（shininesss），大部分情况这产生的影响微乎其微，将光照半径（shininess）调小就可看出影响。

<div style='text-align:center' id='img1-1'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/phong_shading_problem.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图1-1：如图将shininess设为1后可以看到，即使背面的光源也产生了镜面光</span></p>
</div>

<div style='text-align:center' id='img1-2'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/problem_cause1.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图1-2: 引发原因</span></p>
</div>

2. 某些情况下会造成如下的光线产生明显的分割线现象产生。

<div style='text-align:center' id='img1-3'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/advanced_lighting_phong_limit.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图1-3: 产生分割线</span></p>
</div>

<div style='text-align:center' id='img1-4'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/problem_cause2.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图1-4: 上图问题引发原因</span></p>
</div>

漫反射用的是光线反方向向量与平面法线相乘，故不会出现上述两种问题。

## blinn-phong

blinn-phong 则采用半程向量（沿视点-顶点-光点组成的角的角平分线方向上的向量）与平面法线的点积来计算镜面分量，这可以有效解决上面提到的第二个问题。

<div style='text-align:center' id='img1-5'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/advanced_lighting_halfway_vector.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图1-5: 半程向量</span></p>
</div>

半程方向向量计算方式很简单即可得到如下：

$$
\vec{H} = \frac{\vec{L} + \vec{V}}{|\vec{L} + \vec{V}|}
$$

```glsl
float blinn_phong_spec(vec3 lightDir)
{
    // 视线反方向向量，毕竟是从顶点指向视点
    vec3 viewDir = normalize(-fs_in.FragPos);
    // 光线方向向量从光点指向顶点
    vec3 half = normalize(-lightDir + viewDir);
    float spec = pow(max(dot(half, fs_in.Normal), 0.0f), material.shininess);
    return spec;
}
```

> 除此之外，冯氏模型与 Blinn-Phong 模型也有一些细微的差别：半程向量与表面法线的夹角通常会小于观察与反射向量的夹角。所以，如果你想获得和冯氏着色类似的效果，就必须在使用 Blinn-Phong 模型时将镜面反光度设置更高一点。通常我们会选择冯氏着色时反光度分量的 2 到 4 倍。

差别就这么多，很简单

# gamma 校正

对于有（<sup>[[1]](#comment1)</sup>）这类标记的，可点击查看我的个人理解，或许能帮助理解。

当计算出像素的最终颜色后，我们需要立刻将它绘制在显示器上。在过去，电子图像的显示器大多都是阴极射线管显示器（cathode-ray tube (CRT)）。这种显示器有一种物理特性：两倍的电压输入并不会产生两倍的亮度。显示器的单个显像管输出的亮度<sup>[[1]](#comment1)</sup>约相当于输入电压<sup>[[2]](#comment2)</sup>的 2.2 次幂，这个指数 2.2 被称作显示器的 gamma 值。这种幂指数变化碰巧接近人类对亮度的感知方式。

关系**大约**如下：

设输出实际物理亮度和输入电压的关系为

$$
out = f_{crt}(x) = x^{2.2}，x \in [0, 1]  \tag{1}
$$

而人眼感知到的光线亮度和实际进入眼睛的光线物理亮度对应关系也是非线性的，如下

$$
feel = f_{eye}(x) = x^{2.2^{-1}}，x \in [0, 1]  \tag{2}
$$

这种巧合造成了我们对输入电压数值的调整带来的视觉感受变化正好符合我们的预期
即：

$$
f_{eye}(f_{crt}(\frac{b * x}{a})) = \frac{b * f_{eye}(f_{crt}(x))}{a}
$$

<div style='text-align:center' id='img2-1'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/gamma_correction_brightness.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-1: 感知亮度和物理亮度</span></p>
</div>

上图中，对人眼而言似乎第一行是正确的亮度值变化，两倍的亮度值确实对应着（相对一倍亮度值的亮度）两倍亮度。然而，如果讨论物理亮度（例如光源发射的光子数），第二行才是实际的物理意义上亮度与亮度值的对应关系。在第二行中，两倍的亮度值对应两倍的物理亮度，但是由于人眼的感知亮度是不同的，导致第二行看起来有点奇怪。

由于人眼感知亮度的方式更倾向于上图第一行，显示器至今仍在使用上文提到的输出功率与显示颜色的转化关系，以将亮度数值映射到上图第一行的亮度。

显示器的这种非线性映射确实输出了肉眼看起来更加满意的颜色（这个满意是指照着屏幕调整颜色时，把颜色的亮度值减半，你眼睛里感觉到亮度变化也是减半的，即亮度的数值配置变化导致的视觉变化符合预期），但是对于渲染图象而言，若要把设置的颜色按照物理规律进行运算，就会得到不正确的结果。

<div style='text-align:center' id='img2-2'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/gamma_correction_gamma_curves.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-2: 三条线分别代表：gamma校正的输入输出映射关系、理想的输入输出映射关系、显示器显像管的输入电压和输出亮度的映射关系</span></p>
</div>

如上图，仅观察红色凹曲线，横坐标可理解为输入电压强度<sup>[[1]](#comment1)</sup>或者颜色数值，纵坐标理解为输出光线实际物理亮度。

黑色虚线为线性的关系，红色实线为 CRT 屏幕实际上根据颜色数值变化的输出变化关系，在 gamma 值为 2.2 的屏幕上，以暗红色为例，当我们设置颜色为`vec3(0.5, 0, 0)`时，实际上屏幕输出的是`vec3(0.218, 0, 0)`，将设置的颜色值进行运算，乘上二，此时我们设置的颜色值变成了`vec3(1.0, 0, 0)`，屏幕上输出的颜色值也变成了`vec3(1.0, 0, 0)`，也就是说将通过运算将颜色值乘以 2，实际上输出的颜色值却成了原来的 4.5 倍。

到目前为止我们都以为我们在线性的空间<sup>[[3]](#comment3)</sup>中工作，但实际上，我们是在屏幕输出空间中工作，因此我们配置的所有光照变量，在物理上都是不正确的，只不过在屏幕上看起来不错。<sup>[[4]](#comment3)</sup>

由于我们设置的颜色对照了屏幕输出来决定，因此当使用这些颜色进行线性空间计算的时候得到的结果在物理上都是不正确的。使用越复杂的算法这一问题就变得越明显。<sup>[[5]](#comment5)</sup>

<div style='text-align:center' id='img2-3'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/gamma_correction_example.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-3: 有无gamma校正对比</span></p>
</div>

## 校正

校正办法是在显示到屏幕之前，对最终计算得到的颜色做屏幕映射的逆向操作。

$$
\begin{cases}
f_{correction}(x) = x^{2.2^{-1}}\\
feel_{corrected} = f_{eye}(f_{crt}(f_{correction}(x))) = x^{-2.2}
\end{cases} \tag{3}
$$

经过校正后，只要输入值和光线实际物理亮度一致，输出值也会和光线实际物理亮度一致。

> 2.2 是初略估计的大多数显示设备的平均 gamma 值。gamma 值对应的颜色空间被称作 sRGB 颜色空间（不完全相同，近似）。每个显示器有它们自己 gamma 曲线，但是 gamma 值设为 2.2 时在大多数显示器中效果最好。因为这个原因，游戏经常允许玩家改变游戏的 gamma 设置（因为每个显示器的 gamma 值略微有别）

有如下两个方法来在你的场景中应用 gamma 校正：

- 启用 opengl 内建的 sRGB 帧缓冲支持
- 在片段着色器中有我们自己设置 gamma 校正

第一种方法最简单，但是可操作性较差。调用代码就一行

```glsl
glEnable(GL_FRAMEBUFFER_SRGB);
```

通过启用`GL_FRAMEBUFFER_SRGB`，告诉 opengl 之后每个绘制命令在将颜色写入颜色缓冲前都应对颜色（来自 sRGB 空间的颜色）进行 gamma 校正。sRGB 空间指的是符合大多数屏幕标准的 gamma 值为 2.2 的颜色空间。启用`GL_FRAMEBUFFER_SRGB`之后，opengl 自动在随后每次帧缓冲的片段着色器运行之后进行 gamma 校正，包括默认帧缓冲。

> 注意：gamma 校正改变了颜色空间，所以你只应该在颜色要输出到缓冲时才执行 gamma 校正，否则会导致你之后计算得到不正确的结果。例如：如果你使用多个帧缓冲，你应当在帧缓冲间传递线性空间的颜色计算结果，并且只在最后一个帧缓冲中颜色要发送到显示器前使用 gamma 校正。

第二种方法较复杂但可控制性强。
我们可以在片段着色器最后进行 gamma 校正，以确保颜色发送到显示器前都是 gamma 校正过的。

```glsl
void main()
{
    // do super fancy lighting in linear space
    [...]
    // apply gamma correction
    float gamma = 2.2;
    FragColor.rgb = pow(fragColor.rgb, vec3(1.0/gamma));
}
```

这样做的一个问题在于，假如你的程序有多个片段着色器，你可能需要给每一个片段着色器都添加这样的代码。一个更简单的解决方案是在渲染循环中引入一个后处理阶段，在后处理阶段执行 gamma 校正作为最后一步（比如先使用自定义帧缓冲绘制图像，然后从自定义帧缓冲绘制到默认帧缓冲时添加后期效果，以及 gamma 校正）。

## sRGB 纹理

用电脑绘图编辑颜色等，所取的颜色都是根据你在显示器上看到的颜色决定的。这意味着你在电脑上创建或编辑的图片颜色（对显示器输出的颜色而言）不是线性空间中的颜色，而是 sRGB 空间中的颜色<sup>[[4]](#comment4)</sup> 。（这时的颜色值就是需要显示在显示器上的颜色值，而不用再进行 gamma 校正）

在引入 gamma 校正前这不是问题，因为 sRGB 空间的纹理颜色不进行 gamma 校正才是对的。然而引入 gamma 校正后，对 sRGB 空间的纹理进行 gamma，会导致其看起来不合适。

<div style='text-align:center' id='img2-4'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/gamma_correction_srgbtextures.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-4: 有无gamma校正对比</span></p>
</div>

一个解决办法是在纹理颜色计算前，将其转化成非 sRGB 空间的颜色。

在代码中这样做是比较麻烦的，要对每一个纹理进行一次幂运算。幸运的是 opengl 通过提供`GL_SRGB`和`GL_SRGB_ALPHA`这两种内部纹理格式为我们提供了解决问题的办法。

如果我们用这两种格式中的任意一种创建纹理，只要我们使用这些纹理 opengl 会自动修正整颜色，允许我们在线性空间工作。我们可以通过如下代码指定一个 sRGB 空间的纹理。

```glsl
glTexImage2D(GL_TEXTURE_2D, 0, GL_SRGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
```

如果你想在你的纹理中包括透明度分量，你需要指定纹理格式为 GL_SRGB_ALPHA。

需要注意的是并非所有纹理都是 sRGB 纹理，漫反射纹理通常是 sRGB 空间的纹理，用于获取照明参数的纹理通常是线性空间的纹理。

## 衰减

使用了 gamma 校正后另一个不同是衰减方程不同，在真实物理世界中，光线的衰减参数与距离成反比，如下

```glsl
float attenuation = 1.0 / (distance * distance);
```

但不引入 gamma 校正前，如果使用上述方程，效果往往会过于强烈，原因可以参考[注释 4](comment4)，所以一般采用我们之前提到的更高级的方程：

```glsl
float attenuation = 1.0f / (light.constant + light.linear * distance + light.quadratic * distance * distance);
```

或直接使用如下方程

```glsl
float attenuation = 1.0 / distance;
```

用了 gamma 校正后，我们可以直接使用第一个距离平方的反比方程（当然也可以使用更高级的那个，那个方程可以更好地控制和调整衰减参数）

## 注释

<a id="comment1"></a>
[1]. 任何颜色的光线都由三原色光线通过加色法合成，这里的亮度指的应该是各个三原色分量的亮度。后面提到亮度的地方大概都是对颜色单个分量进行讨论，所以有时候说的是颜色，有时候又说的是亮度，看起来有点混乱。

<a id="comment2"></a>
[2]. 这里的电压的数值指的应该是将可取电压区间线性映射到[0,1]区间的对应值。也就是说若 10v 时产生最大亮度，那么 10v 时亮度是$1^{2.2}$，1v 时亮度是$(\frac{1}{10})^{2.2}$；或者理解为电压的强度，最亮时的电压强度视为 1。

<a id="comment3"></a>
[3]. 这个线性空间概念我觉得比较令人费解，指的应该是颜色亮度数值和实际输出颜色物理亮度的一一对应关系是否线性，线性的话就是线性颜色空间，非线性就是非线性颜色空间。不同颜色空间中运算规则应该是不同的。如在线性空间中将 0.5 乘 2 得到的物理亮度结果变成原来两倍，但是在 crt 的空间将 0.5 乘 2 得到的物理亮度结果变成原来的四倍。

<a id="comment4"></a>
[4]. 假设对照物理世界中的某个物体，在电脑上绘制，若要使屏幕上显示的颜色与现实中物体的颜色相同，即要使$out_{crt} = c$，其中$out_{crt}$为屏幕输出，$c$为物体反射的光线物理亮度。因为$out_{crt} = x_1^{2.2}, x_1 \in [0,1]$，所以我们最终在程序中配置的颜色亮度$x_1$与实际的物理亮度$c$是不相等的，但是满足关系$c^{2.2^{-1}} = x_1$

<a id="comment5"></a>
[5]. 假设实际物理世界中光线照在某个物体上会被吸收 50%，反射 50%，光源原本亮度值为$c$。那么当我们在程序中模拟这个光源和物体时，首先对照现实设置光源 sRGB 空间的亮度值$x_1$，有$c^{2.2^{-1}} = x_1$，并且将物理规律运用到程序中得到物体反射光线亮度为$x_1/2$，最终运行程序时我们眼中看到的光线亮度会是：

$$
feel_{crt} = f_{eye}(f_{crt}(x_1/2)) = \frac{c^{2.2^{-1}}}{2}, n \in [0,1] \tag{4}
$$

但现实中我们看到的光线亮度是：

$$
feel_{real} = f_{eye}(c/2) = (\frac{c}{2})^{2.2^{-1}}, n \in [0,1] \tag{5}
$$

$$
\begin{aligned}
&\because (4)(5) \\
&\therefore feel_{crt} < feel_{real}
\end{aligned}
$$

所以采用的算法越高级，越贴近现实，屏幕上看到的光反而更容易显暗

# 阴影

多数游戏用到的能得到一个不错的结果并且也不难实现的阴影技术是阴影映射。阴影映射理解起来不难、不会消耗太多性能，并且非常容易扩展成更加高级的算法（像 Omnidirectional Shadow Maps 和 Cascaded Shadow Maps）

## 小窗口

为方便查找问题，确认阴影贴图是否成功生成，可以在右上角绘制一个小窗口，显示阴影贴图等

__示例1：直接绘制一个矩形__

```c++
    static float vertices[] = {
    0.6f, 1.0f, -0.1f, -0.1f,
    0.6f, 0.7f, -0.1f, 1.1f,
    1.0f, 1.0f, 1.1f, 0.0f,

    1.0f, 0.7f, 1.1f, 1.1f,
    1.0f, 1.0f, 1.1f, -0.1f,
    0.6f, 0.7f, -0.1f, 1.1f};

    // 绑定缓冲等......

    // texture是阴影贴图，这个函数在所有渲染完成后调用
    void draw(unsigned int texture)
    {
        glUseProgram(program);
        glActiveTexture(GL_TEXTURE0);
        glBindTexture(GL_TEXTURE_2D, texture);

        glBindVertexArray(vao);
        glDrawArrays(GL_TRIANGLES, 0, 6);
        glBindVertexArray(0);
        glUseProgram(0);
    };
```

__示例2：使用glViewport__

```glsl
    // 其他渲染都结束后调用
    void draw(int width, int height, float scale = 0.2f)
    {
        glDisable(GL_DEPTH_TEST);

        // 右上角显示
        glViewport((int)((1.0f - scale) * (float)width), (int)((1.0f - scale) * (float)height), width, height);

        // 渲染代码...

        glUseProgram(0);
        glViewport(0, 0, width, height);

        glEnable(GL_DEPTH_TEST);
    };
```

## 阴影映射

此节中提到的光源都指的是平行光，平行光概念上来说是无穷远处的光，没法取位置坐标，因此此节提到的光源位置指的是沿平行光光线方向任取的某点。

阴影映射背后的思想很简单，将视点与光源位置重合，能被看到的点就是能被光照到的点，不能被看到的点就是有阴影的点。

实现这种思路利用的是深度缓冲。

思路如下：
1. 先在光源的视角下绘制一次，将深度缓冲保存至阴影贴图中。
2. 再在视点的位置绘制一次，绘制时进行如下操作：
   + 将顶点A的位置P转换为光源视角下的顶点A的位置T。
   + 根据光源视角下的顶点A的位置T获取获取相应阴影贴图中的深度。
   + 若阴影贴图中的深度小于位置T的深度，那么说明在光源的视角下，顶点A是没有通过深度测试的，也就是说顶点A的光线被其他点遮挡了。
   + 此时就不计算顶点A的光线效果，只保留环境光

达成上述思路需要使用一个用来绘制阴影贴图的额外帧缓冲

```c++
// 创建帧缓冲
unsigned int depthMapFBO;
glGenFramebuffers(1, &depthMapFBO);  
// 创建阴影贴图
const unsigned int SHADOW_WIDTH = 1024, SHADOW_HEIGHT = 1024;

unsigned int depthMap;
glGenTextures(1, &depthMap);
glBindTexture(GL_TEXTURE_2D, depthMap);
// 此时的纹理类型是GL_DEPTH_COMPONENT
glTexImage2D(GL_TEXTURE_2D, 0, GL_DEPTH_COMPONENT, 
             SHADOW_WIDTH, SHADOW_HEIGHT, 0, GL_DEPTH_COMPONENT, GL_FLOAT, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT); 
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);  
// 绑定阴影贴图到帧缓冲的深度缓冲
glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_TEXTURE_2D, depthMap, 0);
// 前面提到过没有颜色附件的帧缓冲是不完整的，因此我们需要告诉opengl我们不需要渲染任何颜色数据
// 通过调用如下两行来达到此目的
glDrawBuffer(GL_NONE);
glReadBuffer(GL_NONE);
glBindFramebuffer(GL_FRAMEBUFFER, 0);  
```

然后就是着色器

顶点着色器没什么改变

```glsl
#version 330 core
layout (location = 0) in vec3 aPos;

// lightSpaceMatrix是光源位置的观察矩阵与投影矩阵相乘，也就是projection * view
// 不过这个投影矩阵不是透视投影矩阵而是正交投影矩阵，代表平行光
uniform mat4 lightSpaceMatrix;
uniform mat4 model;

void main()
{
    gl_Position = lightSpaceMatrix * model * vec4(aPos, 1.0);
}  
```

因为不需要绘制颜色，片段着色器写一个空main即可

```glsl
#version 330 core

void main()
{             
    // gl_FragDepth = gl_FragCoord.z;
}  
```

可尝试将阴影贴图绘制出来，片段着色器代码如下

```glsl
#version 330 core
out vec4 FragColor;
  
in vec2 TexCoords;

uniform sampler2D depthMap;

void main()
{             
    float depthValue = texture(depthMap, TexCoords).r;
    FragColor = vec4(vec3(depthValue), 1.0);
}  
```

开始绘制阴影时很简单，只是顶点着色器中多了些输入输出，以及片段着色其中多了些判断


顶点着色器改变如下

```glsl
// ...

out VS_OUT {
    // ...

    vec4 FragPosLightSpace;
} vs_out;

// ...

uniform mat4 lightSpaceMatrix;

void main()
{   
    // ...
    vs_out.FragPosLightSpace = lightSpaceMatrix * vec4(vs_out.FragPos, 1.0);
    // ...
}
```

片段着色器中添加的判断

```glsl
float ShadowCalculation(vec4 fragPosLightSpace)
{
    // 透视除法
    vec3 projCoords = fragPosLightSpace.xyz / fragPosLightSpace.w;
    // 映射到[0,1]区间
    projCoords = projCoords * 0.5 + 0.5;
    // 获取光线视角的最小深度
    float closestDepth = texture(shadowMap, projCoords.xy).r; 
    // 当前顶点的深度
    float currentDepth = projCoords.z;
    // 比较深度判断是否有阴影
    float shadow = currentDepth > closestDepth  ? 1.0 : 0.0;

    return shadow;
}  
```

做完如上步骤就成功了不过会存在一些问题

### 阴影失真

第一个问题是阴影失真。如下图

<div style='text-align:center' id='img3-1'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/shadow_acne.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图3-1: 阴影失真，纹理过滤设为GL_NEAREST时</span></p>
</div>

<div style='text-align:center' id='img3-2'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/shadow_acne_linear.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图3-2: 阴影失真，纹理过滤为GL_LINEAR时</span></p>
</div>

产生这个问题的原因是阴影贴图是一个个像素组成的，并不能准确取到某一坐标处的离光源最近点的准确深度，而只能取邻近点的值。

也就是说如下代码取到的只是在阴影贴图上离点$(x, y)$最近的像素$(x + a, y + b)$所保存的深度（或者根据四周像素取的线性值）

```glsl
float depth = texture(shadow_texel, light_space_frag_pos.xy).r;
```

当取到的depth比该点处实际的最小深度小时，就会导致该点对应的离光源最近的顶点处显示阴影。

解决这个问题的办法是设置一个偏差值bias，将点$(x, y)$的实际深度稍微变小，这样就能消除depth取值偏差带来的影响。

bias的取值方法可以根据阴影失真产生的原因来推断，点$(x + a, y + b)$上深度取值depth与$(x, y)$的深度偏差取决于两者所对应的平面A相对光线方向的倾斜程度，当光线方向和平面A垂直时，$(x + a, y + b)$与$(x, y)$的深度取值是相同的（平面A前没其他遮挡物）

因此bias的取值大小根据光线方向与顶点处的法向量夹角决定，夹角越小，bias越小。

```glsl
float bias = max(0.05 * (1.0 - dot(normal, lightDir)), 0.005);
float shadow = currentDepth - bias > closestDepth  ? 1.0 : 0.0;
```

### 悬浮

引入bias后又会存在第二个问题，悬浮

<div style='text-align:center' id='img3-3'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/peter_panning.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图3-3: bias设为0.1时产生的悬浮问题</span></p>
</div>

这个引发原因也好理解，由于我们设置了bias。对于同一光线路径上的两个点A，B，A为更靠近光源的顶点，B为被A遮挡应当存在阴影的点，A点深度为$d_A$，b点深度为$d_B$，在将B点深度与最小深度比较时，若$d_B - d_A < bias$，就可能照成$depth > d_B - bias$

解决这个问题的办法是不用bias，用面剔除也可以解决阴影失真，这样就不需要Bias了。

具体方法是生成阴影贴图时剔除物体的正面。由于实心物体面向光的一面与背光一面深度有一定距离，阴影贴图取的是背面深度，因此些许的depth取值偏差不会影响深度比较，也就不会产生条纹。而对于地板这样的平面，若其前方没有遮挡物，而正面又被剔除了，此时深度缓冲取值为最小值1，因此地板上也不会产生条纹。（不过这种方法有限制）

另外这个方法会引发的一个问题是，blinn光照有个问题是背光面在特定情况下会有一定光照，使用正面剔除后可能导致背面阴影失真

<div style='text-align:center' id='img3-4'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/shadow_acne_back.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图3-4: 背面的阴影失真</span></p>
</div>

### 超出采样范围

阴影贴图只能保存一定裁剪空间内的深度，超出裁剪空间的点，转化到光源的空间下时，深度存在大于1（比远平面还远）和小于0（比近平面还近）以及超出左右两边等等情况，为了直观感受这一点，我们做修改如下。

缩小裁剪空间范围

```c++
    glm::mat4 light_projection = glm::ortho(-4.0f, 4.0f, -10.0f, 10.0f, adjust->near, adjust->far);
```

修改着色器

```glsl
float ShadowCalculation(vec4 fragPosLightSpace)
{
    // ...
    if(currentDepth < 0)
    {
        // 深度小于0加亮
        return 10;
    }
    // 事实上这一处判断不需要，因为最后一行的返回语句也会导致currentDepth大于1时直接显示阴影
    else if(currentDepth > 1)
    {
        // 深度大于1直接取阴影
        return 0;
    }
    return closestDepth < currentDepth - bias ? 0.0f : 1.0f;
}
```

<div style='text-align:center' id='img3-5'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/depth_value.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图3-5: 超出采样范围会产生的情况</span></p>
</div>

由于阴影贴图中最大深度值为1，因此即使不做上述修改，比远平面还远的点也总是会显示阴影，如果不想这样，我们就要做些修改。

修改纹理重复方式

```c++
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_BORDER);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_BORDER);
GLfloat borderColor[] = { 1.0, 1.0, 1.0, 1.0 };
// 设置边框颜色
glTexParameterfv(GL_TEXTURE_2D, GL_TEXTURE_BORDER_COLOR, borderColor);
```

然后在着色器中判断

```glsl
float ShadowCalculation(vec4 fragPosLightSpace)
{
    // ...
    if(projCoords.z > 1.0)
        shadow = 0.0;

    return shadow;
}
```

这样超出裁剪范围的都没有阴影

### PCF

目前实现的阴影锯齿比较严重，这也是由于阴影贴图分辨率优先。可以通过提高阴影贴图的分辨率这一方法来减小锯齿。

还有一个方案叫PCF（percentage-closer filtering），这是一种多个不同过滤方式的组合，它产生柔和阴影，使它们出现更少的锯齿块和硬边。核心思想是从阴影贴图中多次采样，每一次采样的纹理坐标都稍有不同。每个独立的样本可能在也可能不再阴影中。所有的次生结果接着结合在一起，进行平均化，我们就得到了柔和阴影。
他的一种简单实现如下：

```glsl
float shadow = 0.0;
vec2 texelSize = 1.0 / textureSize(shadowMap, 0);
for(int x = -1; x <= 1; ++x)
{
    for(int y = -1; y <= 1; ++y)
    {
        float pcfDepth = texture(shadowMap, projCoords.xy + vec2(x, y) * texelSize).r; 
        shadow += currentDepth - bias > pcfDepth ? 1.0 : 0.0;        
    }    
}
shadow /= 9.0;
```

### 投影矩阵

根据不同光线特点，正交适合平行光，透视适合点光源

> 另一个细微差别是，透视投影矩阵，将深度缓冲可视化经常会得到一个几乎全白的结果。发生这个是因为透视投影下，深度变成了非线性的深度值，它的大多数可辨范围都位于近平面附近。为了可以像使用正交投影一样合适地观察深度值，你必须先将非线性深度值转变为线性的，如我们在深度测试教程中已经讨论过的那样。

# 参考

1. [LearnOpenGL CN](https://learnopengl-cn.github.io/)
2. [LearnOpenGL](https://learnopengl.com/)
3. [制作shadow map时，为什么剔除正面可以解决物体悬浮问题（Peter panning问题）？](https://www.zhihu.com/question/321779117)
