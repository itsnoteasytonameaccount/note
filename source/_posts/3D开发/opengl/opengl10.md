---
title: opengl(10):万向光阴影，CSM
date: 2021/04/10
tags:
  - opengl
---

为学懂webgl，从opengl入手。本文内容主要来自[LearnOpenGL CN](https://learnopengl-cn.github.io/)文档，加上个人理解，以及一些细节补充、经验记录。本人学习过程中的详细代码参见[此处](https://github.com/itsnoteasytonameaccount/OpenGL-learning)

要学习推荐看原文[LearnOpenGL CN](https://learnopengl-cn.github.io/)，有评论区，加载评论区需要翻墙

另可在[此处](https://www.khronos.org/opengl/wiki/Getting_Started)找到其他opengl官方推荐的资料、教程网站
<!-- more -->

# 万向光阴影

万向光阴影的思路和阴影映射相同，不过万向光阴影用的阴影贴图是立方体贴图。通过立方体贴图就可以存放点光源在所有方向上距离最近的顶点深度值。因此阴影映射更适合平行光源，而万向光阴影更适合点光源。

将万向光阴影的实现步骤分为如下：

1. 准备步骤：初始化窗口、创建缓冲对象、创建帧缓冲、创建立方体贴图、准备数据等等
2. 第一轮渲染，渲染阴影贴图
3. 第二轮渲染，渲染场景并根据阴影贴图决定是否存在阴影

这一部分和上一章阴影贴图部分所差不多，较为简单，只有部分代码存在不同之处，如下文。

## 准备步骤

其余基本一致，准备帧缓冲时大致相同

```c++
void PointShadows::createFramebuffer()
{
    // 生成帧缓冲
    glGenFramebuffers(1, &fbo);

    // 生成立方体贴图
    glGenTextures(1, &depth_texture);
    glBindTexture(GL_TEXTURE_CUBE_MAP, depth_texture);
    for (unsigned int i = 0; i < 6; i++)
    {
        glTexImage2D(GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 0, GL_DEPTH_COMPONENT, SHADOW_WIDTH, SHADOW_HEIGHT, 0, GL_DEPTH_COMPONENT, GL_FLOAT, NULL);
    }
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);

    glBindFramebuffer(GL_FRAMEBUFFER, fbo);
    // 绑定阴影贴图到帧缓冲
    glFramebufferTexture(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, depth_texture, 0);

    // 由于没绑定颜色缓冲，设置不读写颜色缓冲
    glDrawBuffer(GL_NONE);
    glReadBuffer(GL_NONE);

    // 解绑
    glBindFramebuffer(GL_FRAMEBUFFER, 0);
    glBindTexture(GL_TEXTURE_CUBE_MAP, 0);
}
```

另外准备数据时需要为 6 个面都准备变换矩阵

```c++
float aspect = (float)SHADOW_WIDTH/(float)SHADOW_HEIGHT;
float near = 1.0f;
float far = 25.0f;
// 透视矩阵的视角设置为90度，以确保所有方向上的深度被保存
glm::mat4 shadowProj = glm::perspective(glm::radians(90.0f), aspect, near, far);
// 矩阵的存放顺序要和立方体贴图面的遍历顺序相同，以方便对应
std::vector<glm::mat4> shadowTransforms;
shadowTransforms.push_back(shadowProj *
                 glm::lookAt(lightPos, lightPos + glm::vec3( 1.0, 0.0, 0.0), glm::vec3(0.0,-1.0, 0.0));
shadowTransforms.push_back(shadowProj *
                 glm::lookAt(lightPos, lightPos + glm::vec3(-1.0, 0.0, 0.0), glm::vec3(0.0,-1.0, 0.0));
shadowTransforms.push_back(shadowProj *
                 glm::lookAt(lightPos, lightPos + glm::vec3( 0.0, 1.0, 0.0), glm::vec3(0.0, 0.0, 1.0));
shadowTransforms.push_back(shadowProj *
                 glm::lookAt(lightPos, lightPos + glm::vec3( 0.0,-1.0, 0.0), glm::vec3(0.0, 0.0,-1.0));
shadowTransforms.push_back(shadowProj *
                 glm::lookAt(lightPos, lightPos + glm::vec3( 0.0, 0.0, 1.0), glm::vec3(0.0,-1.0, 0.0));
shadowTransforms.push_back(shadowProj *
                 glm::lookAt(lightPos, lightPos + glm::vec3( 0.0, 0.0,-1.0), glm::vec3(0.0,-1.0, 0.0));
```

## 渲染阴影贴图

立方体贴图有 6 个面，相较阴影映射渲染麻烦些。

一种方法是渲染 6 次，每次将立方体贴图的一个面绑定到帧缓冲。如下：

```c++
for(unsigned int i = 0; i < 6; i++)
{
    GLenum face = GL_TEXTURE_CUBE_MAP_POSITIVE_X + i;
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, face, depthCubemap, 0);
    // lightViewMatrices是光源位置六个方向上的projection * view
    BindViewMatrix(lightViewMatrices[i]);
    RenderScene();
}
```

不过这种方法比较消耗性能。

另一种方法是利用几何着色器，每次渲染分别输出图像到 6 个面，需要用到几何着色器中的`gl_Layer`变量。这种方法相对第一种方法在大部分情况下更高效。

此时顶点着色器只用输出顶点的世界坐标

```glsl
#version 330 core

layout (location = 0) in vec3 aFragPos;

uniform mat4 model;

void main()
{
    gl_Position = model * vec4(aFragPos, 1.0f);
}
```

此时几何着色器中的代码如下

```glsl
#version 330 core

// 输入一个三角形
layout (triangles) in;
// 输出6个三角形，每个面一个
layout (triangle_strip, max_vertices = 18) out;

// 6个面方向上的 投影矩阵*观察矩阵
uniform mat4 shadow_matrices[6];

out vec4 FragPos;

void main()
{
    // 将三角形循环输出到6个顶点
    for(int face = 0; face < 6; face++)
    {
        // 设置输出到的面
        gl_Layer = face;
        for(int i = 0; i < 3; i++)
        {
            // 将顶点的世界坐标输出到几何着色器供下一步计算线性深度
            FragPos = gl_in[i].gl_Position;
            // 输出裁剪坐标
            gl_Position = shadow_matrices[face] * FragPos;
            // 提交点
            EmitVertex();
        }
        // 结束一个图元提交
        EndPrimitive();
    }
}
```

由于矩阵转化后的深度，是对应各个面的裁剪坐标系下的深度$z_c$，而不是实际点光源到顶点的距离，为了便于之后的计算需要修改一下最终输出的深度。

因此片段着色器中如下：

```glsl
#version 330 core

in vec4 FragPos;

uniform float far_plane;
uniform vec3 lightPos;

void main()
{
    // 计算距离
    float light_distance = length(FragPos.xyz - lightPos);
    // 线性映射到[0,1]
    light_distance = light_distance / far_plane;

    // 改变深度
    gl_FragDepth = light_distance;
}
```

## 渲染默认帧缓冲

渲染默认帧缓冲时操作没什么区别，只有片段着色器中，计算阴影的方式需要改变，不再需要将坐标转化到光源的坐标系下，而是直接计算距离。

```glsl
float ShadowCalculation(vec3 fragPos)
{
    // 顶点到光源的向量
    vec3 fragToLight = fragPos - lightPos;
    // 取立方体贴图中保存的深度
    float closestDepth = texture(depthMap, fragToLight).r;
    // 映射到[0,1]的逆操作
    closestDepth *= far_plane;
    // 计算顶点到光源距离
    float currentDepth = length(fragToLight);
    // 比较计算阴影值
    float bias = 0.05;
    float shadow = currentDepth -  bias > closestDepth ? 1.0 : 0.0;

    return shadow;
}
```

### PCF

同样用 PCF 抗锯齿，但有些不同

如果按照阴影映射中的方式采样，大概如下

```glsl
float shadow = 0.0;
float bias = 0.05;
float samples = 4.0;
float offset = 0.1;
for(float x = -offset; x < offset; x += offset / (samples * 0.5))
{
    for(float y = -offset; y < offset; y += offset / (samples * 0.5))
    {
        for(float z = -offset; z < offset; z += offset / (samples * 0.5))
        {
            float closestDepth = texture(depthMap, fragToLight + vec3(x, y, z)).r;
            closestDepth *= far_plane;   // Undo mapping [0;1]
            if(currentDepth - bias > closestDepth)
                shadow += 1.0;
        }
    }
}
shadow /= (samples * samples * samples);
```

这种方法的一个问题是采样数太多了，$4*4*4$就有 64 个样本，而且不少采样是多余的，举个例子来说向量$(1, 1, 1)$和$(2, 2, 2)$采到的样本其实是相同的，上面代码的采样方法也可能取到非常相近的样本。

因此最好规定一个采样偏移数组，来决定取哪些样本。

```glsl
vec3 sampleOffsetDirections[20] = vec3[]
(
   vec3( 1,  1,  1), vec3( 1, -1,  1), vec3(-1, -1,  1), vec3(-1,  1,  1),
   vec3( 1,  1, -1), vec3( 1, -1, -1), vec3(-1, -1, -1), vec3(-1,  1, -1),
   vec3( 1,  1,  0), vec3( 1, -1,  0), vec3(-1, -1,  0), vec3(-1,  1,  0),
   vec3( 1,  0,  1), vec3(-1,  0,  1), vec3( 1,  0, -1), vec3(-1,  0, -1),
   vec3( 0,  1,  1), vec3( 0, -1,  1), vec3( 0, -1, -1), vec3( 0,  1, -1)
);

float shadow = 0.0;
float bias = 0.15;
int samples = 20;
float viewDistance = length(viewPos - fragPos);
// 偏移半径，决定从附近多远的地方采样
float diskRadius = 0.05;
for(int i = 0; i < samples; ++i)
{
    float closestDepth = texture(depthMap, fragToLight + sampleOffsetDirections[i] * diskRadius).r;
    closestDepth *= far_plane;   // Undo mapping [0;1]
    if(currentDepth - bias > closestDepth)
        shadow += 1.0;
}
shadow /= float(samples);
```

最终效果

<div style='text-align:center' id='img1-1'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/point_shadows.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图1-1: 阴影效果</span></p>
</div>

# CSM（Cascaded Shadow Maps 级联阴影贴图）

阴影贴图是游戏引擎中使用非常广泛的用于实现实时阴影的技术。在较大的场景中使用这一技术时，阴影贴图会变得难以调整并且更容易产生失真和锯齿。

级联阴影贴图（CSM）是一种通过在近处提供（相对其包括的场景）更大分辨率的阴影贴图而远处出提供（相对其包括的场景）较小的分辨率的阴影贴图修复锯齿问题的技术。

CSM 通常被用来模拟大型区域中太阳投射出的阴影。在一张阴影贴图中捕获所有东西需要大到难以实现的分辨率，因此，需要使用多张阴影贴图 ———— 一个阴影贴图只包括近处物体以此来投射出更清晰的阴影，另一个则用模糊的分辨率来捕获远处所有物体。当然并非只能有两个阴影贴图，也可以使用两个以上的阴影贴图。

CSM 分为如下步骤：

1. 进行摄像头视锥的分割（求分割位置的深度）
2. 光锥的计算（求光线的正交投影矩阵）
2. 对得到的每一个光锥，渲染光线视角下场景中顶点的深度到阴影贴图
3. 在摄像头视角下渲染场景中的顶点，通过阴影贴图及光线视角下的深度决定是否存在阴影。

<div style='text-align:center' id='img2-1'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/cascaded_shadow_maps.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-1: 级联阴影贴图划分示例</span></p>
</div>

观看[图 2-1](#img2-1)可以注意到，所有的阴影投射物以及摄像头视锥切片被包含在光锥中，光锥外不应该有需要被渲染的物体。换句话说，光锥是摄像头视锥切片的边界盒子（长方体），它的近平面延伸到足够包裹所有可能的遮蔽物。如果在树上方有个遮蔽物 B，这个盒子就需要被适当地延长，否则 B 不会投射出阴影。

## 视锥的分割：平行分片阴影贴图

CSM 算法的第一步需要计算出视锥的分割位置，下文取法来自[Parallel-Split Shadow Maps for Large-Scale Virtual Environments](https://dl.acm.org/doi/10.1145/1128923.1128975)，不一定要这样取，不讲究的取法是凭直觉取，也可以用别人总结好的取法。

<div style='text-align:center' id='img2-2'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/pssm.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-2: pssm示意图</span></p>
</div>

### 推导过程

下文是我对分割公式的推导过程的理解。

关于上图一些量解释一下:

1. 因为是对深度分割，现在仅对深度进行考虑
2. $d_s$代表阴影贴图上一个像素的长宽，$d_p$代表阴影贴图上一个像素最终投影在近平面上或者显示在屏幕上的阴影尺寸。
3. 阴影是否有锯齿，取决于$d_p$的大小，也就是说阴影贴图上一个像素长度的内容代表屏幕上多少个像素长度的内容，如果$d_p$大于一个像素的长度，那么就会产生锯齿等。
4. 由于$d_p$和$d_s$没在一个坐标系下，不好比较，所以引入$d_z$，$d_z$是为了建立$d_s$和$d_p$关系式而引入的中间量。这个我一开始不理解，看了“Perspective Shadow Maps<sup>[[4]](#参考)</sup>”这篇论文才理解了，“Parallel-Split Shadow Maps for Large-Scale Virtual Environments<sup>[[6]](#参考)</sup>”引用了“Light Space Perspective Shadow Maps<sup>[[5]](#参考)</sup>”，而“Light Space Perspective Shadow Maps”又引用了“Perspective Shadow Maps”，三者关于透视误差这一处的描述都相近，但我觉得“Perspective Shadow Maps”说得清楚点，其他两个我看了老半天就是看不懂。

> In Figure, the light beams through a texel with the size ds × ds in normalized texture space (i.e. s ∈ [0,1]) falls on a surface with the length dz in world space (e.g. dz = (f − n)ds for standard shadow maps).

上面原文中$d_z$我没理解错的话大概意思是将阴影贴图坐标范围映射到$[0, 1]$，用阴影贴图表示视锥中$[n, f]$范围的内容，阴影贴图上$d_s$长度的单个像素表示的内容在世界空间中的对应长度用$d_z$表示。

> ϕ and θ denote the angles between the surface normal and vector to the screen and shadow map plane respectively.

$\varphi$是垂直指向近平面的向量与被投影的面的法向量的夹角；$\theta$说是垂直指向阴影贴图平面的向量与被投影的面的法向量的夹角，但我觉得有点奇怪，实际上阴影贴图平面和被投影的面应该完全不在一个坐标系下啊，位置没办法比较吧？难道把阴影贴图平面视作水平的面？那还需要$\theta$干嘛，不就是$\varphi$的余角吗？后面何必整出什么$\cos{\varphi}/\cos{\theta}$那么复杂，不就是个正切值吗？

三角形相似，n 是近平面的位置，z 是被投影曲面的位置，又有：

$$
d_y = \frac{zd_p}{n}
$$

z 是被投影的平面的位置

$$
d_y = d_z\frac{\cos{\varphi}}{\cos{\theta}}
$$

那么可以得到$d_p$与$d_s$的比较关系

$$
\frac{d_p}{d_s} = n\frac{d_z}{zd_s}\frac{\cos{\varphi}}{\cos{\theta}}
$$

这个式子里 n 是固定的，$\frac{d_z}{zd_s}$被称为透视误差（perspective aliasing）,$\frac{\cos{\varphi}}{\cos{\theta}}$被称为投影误差（projection）。这种定义似乎最早出自“Perspective Shadow Maps<sup>[[4]](#参考)</sup>”

投影误差是没法处理的，因为它取决于场景中物体表面的形状，要处理只能在不同倾斜率的表面所对应的阴影贴图区域设置不同的分辨率，这显然不太可能实现。

因此 pssm 考虑的是如何减小透视误差。

**对数分割方法**

理想情况下，较好的透视误差分配方法是保持$d_p/d_s$在整个深度范围内一致。即满足（$\rho$为常量）：

$$
\rho = \frac{d_z}{zd_s}
$$

由上式，在$[0, s], s \in [0, 1]$上对$d_s$积分有：

$$
\int_0^sd_s = \frac{1}{\rho}\int_n^{z}\frac{1}{z}d_z \implies s = \frac{\ln{z/n}}{\rho}
$$

因为$z = f$时$s = 1$，有

$$
\rho = \ln{f/n} \implies s = \frac{\ln{z/n}}{\ln{f/n}} \implies n(f/n)^s = z
$$

将公式离散化，记视锥第 i 个分割块的深度为$c_i^{log}$，有

$$
c_i^{log} = n(f/n)^{s_i}
$$

> Because this split scheme is designed to produce the theoretically even distribution of perspective aliasing errors, the resolution allocated for [Ci ,Ci−1] should be i/m of the overall texture resolution. Substituting si = i/m in Equation (3) gives:

> 因为这种分割方案只是透视误差的理论上的分配方案，第 i 块分块的分辨率分配应该占全部纹理分辨率的 i/N（N 为视锥的分割份数）。(也就是说将分辨率平均分配，理论上每个分割块需要用到的阴影贴图分辨率是一样的，每个块分到 1/N，第 i 个块的远平面对应的阴影贴图坐标自然是 i/N)

最终得到

$$
c_i^{log} = n(f/n)^{i / N}
$$

对数分割方法的问题在于分割出来的块中，靠近近平面的块的长度会非常小，只能包括非常小一部分对象。这是因为我们之前的公式都是理想情况下的公式，假设了视锥$[n, f]$范围内的所有内容都可以准确映射到阴影贴图的$[0, 1]$范围内，阴影贴图既不会漏掉某些内容，也不会包括不可见的内容。这在实践中不可能实现（因为我们仅考虑了深度，其他坐标轴方向的的内容没办法完全贴合的映射到阴影贴图中）。

<div style='text-align:center' id='img2-3'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/light_frustum_and_view_frustum.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-3: 光线的截体包括不可见区域</span></p>
</div>

上述问题会导致过了视锥的分割区后阴影的清晰度剧烈突变，因为较靠近近平面的视锥区域过采样而较远的视锥欠采样。

**平均分配**

很简单，就是按深度平均分配视锥，每个视锥深度跨度都是$(f - n)/N$，$N$为分割数

此时第 i 个分片的近平面坐标即为

$$
z_i = n + (f - n)(i / N)
$$

这种方法导致较靠近近平面的视锥区域欠采样而较远的视锥过采样。

**综合分配方法**

因为上述两种方法都存在问题，比较符合实际的方法是将上述方法加权综合的结果

即

$$
z_i = \lambda n(f / n)^{i / N} + (1 - \lambda)(f - n)(i / N)
$$

## 光锥计算

光锥的计算方法较为简单

1. 对于每一个视锥分割块，求出其八个顶点的世界坐标，求法：
    + 根据opengl(3)中的知识，最终八个顶点会被转化为标准设备坐标，$p_n = (x_n, y_n, z_n)$
    + 根据转化步骤的逆步骤，可求世界坐标

    $$
    p_w = M_{view}^{-1} * M_{proj}^{-1} * 
    \begin{pmatrix}
    -z_ex_n\\-z_ey_n\\-z_ez_n\\-z_e
    \end{pmatrix}
    $$

2. 计算视锥的八个顶点在光源视角下的坐标，$M$为光线的模型观察矩阵，$P$为正交投影矩阵，且由于我们考虑的是平行光，光锥为长方体，此时可以设$P$为单位矩阵，则八个顶点在光源坐标系下的坐标$p_h = PMp_w$，设8个顶点在每个方向上的最小值为$m_i$最大值为$M_i$，摄像机下的新的投影矩阵$P$的按照正交矩阵的的公式可直接得出(这里有个问题在于——阴影可能由视锥之外的物体产生，下面的公式远近平面的取法显然忽略了视锥外的物体，因此实际上在取$m_z$可能需要做些调整)：

$$
P = \begin{pmatrix}
\frac{2}{M_x - m_x}&0&0&-\frac{M_x + m_x}{M_x - m_x}\\
0&\frac{2}{M_y - m_y}&0&-\frac{M_y + m_y}{M_y - m_y}\\
0&0&-\frac{2}{M_z - m_z}&-\frac{M_z + m_z}{M_z - m_z}\\
0&0&0&1
\end{pmatrix}
$$


示例代码如下

```c++
// 标准设备坐标
const glm::vec4 normalized_pos[8] = {
    glm::vec4(1.0f),
    glm::vec4(-1.0f, 1.0f, 1.0f, 1.0f),
    glm::vec4(1.0f, -1.0f, 1.0f, 1.0f),
    glm::vec4(-1.0f, -1.0f, 1.0f, 1.0f),
    glm::vec4(1.0f, 1.0f, -1.0f, 1.0f),
    glm::vec4(-1.0f, 1.0f, -1.0f, 1.0f),
    glm::vec4(1.0f, -1.0f, -1.0f, 1.0f),
    glm::vec4(-1.0f, -1.0f, -1.0f, 1.0f)};

// 计算光锥的投影*观察矩阵
void CSM::computeLightProjection(int i, float aspect, glm::mat4 &view_inverse)
{
    float minz, maxz, minx, maxx, miny, maxy, z_near, z_far, z_e;
    glm::vec4 light_space_pos;

    // 分割块近平面深度
    z_near = i < 1 ? 0.1f : z_values[i - 1];
    // 分割块远平面深度
    z_far = z_values[i];

    // 计算投影矩阵逆矩阵
    glm::mat4 projection_inverse = glm::inverse(glm::perspective(glm::radians(45.0f), aspect, z_near, z_far));

    for (int j = 0; j < 8; j++)
    {
        // 由于是左手坐标系，深度坐标为负值，其中normalized_pos[j]为标准设备坐标
        z_e = normalized_pos[j].z > 0 ? -z_far : -z_near;
        // 光线空间下的坐标
        light_space_pos = light_view * view_inverse * projection_inverse * (-z_e * normalized_pos[j]);

        // 比较并保存minx,maxx等八个值。
        // ...
    }

    // 创建矩阵，-maxz - 10是为了适当延伸光锥近平面，因为某些遮挡物虽然位于视锥分割块的边界盒外面，但是会在视锥分割快区域范围内产生阴影
    light_projection_view[i] = glm::ortho(minx, maxx, miny, maxy, -maxz - 10, -minz) * light_view;
}
```

<div style='text-align:center' id='img2-4'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/has_no_stretch.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-4: 如图如果不适当将光锥向近平面方向延长，就会漏掉一些阴影</span></p>
</div>

## 渲染阴影贴图

进行一轮渲染，需要对每个阴影贴图渲染一次
大致步骤：
+ 更新视锥分割点
+ 循环渲染n次，n为分割数
  - 渲染至第i个阴影贴图时，计算光线视角下的对应第i个视锥分割块的投影矩阵
  - 绑定第i个阴影贴图到自定义帧缓冲的深度缓冲
  - 将视锥的投影矩阵以及观察矩阵传入着色器，渲染自定义帧缓冲

代码如下

```c++

void CSM::updateSplitZ(int N)
{
    const float lambda = adj.lambda;
    float log_split, uniform_split;
    for (int i = 1; i <= SPLIT_QUANTITY; i++)
    {
        // 对数分割
        log_split = near * pow(far / near, (float)i / N);
        // 平均分割
        uniform_split = near + (far - near) * ((float)i / N);
        // 加权
        z_values[i - 1] = lambda * log_split + (1.0f - lambda) * uniform_split;
    }
}

void CSM::draw()
{
    // 宽高比
    float aspect = HEIGHT == 0 ? 0 : (float)WIDTH / HEIGHT;
    // 观察矩阵逆矩阵
    glm::mat4 view_inverse = glm::inverse(camera.getViewMatrix());
    // 参数名
    char light_projection_view_param[30] = "light_projection_view[0]";

    // 计算光线视角下的观察矩阵
    light_view = glm::lookAt(-(adj.light.direction * glm::vec3(10.0f)), glm::vec3(0.0f), glm::vec3(0.0f, 1.0f, 0.0f));

    // 更新分割位置
    updateSplitZ();

    glViewport(0, 0, SHADOW_MAP_WIDTH, SHADOW_MAP_HEIGHT);
    glBindFramebuffer(GL_FRAMEBUFFER, fbo);
    shadow_shader.useProgram();
    
    // 防止阴影悬浮
    glCullFace(GL_FRONT);

    for (int i = 0; i < SPLIT_QUANTITY; i++)
    {
        // 绑定纹理数组某一层到帧缓冲的深度缓冲
        glFramebufferTextureLayer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, texture_array, 0, i);

        glClear(GL_DEPTH_BUFFER_BIT);

        // 计算光线的投影矩阵
        computeLightProjection(i, aspect, view_inverse);
        shadow_shader.setUniformMatrix4("projection_view", light_projection_view[i]);

        drawScence(shadow_shader);
    }

    // ...
}
```

## 渲染场景并计算阴影

渲染场景
主要是片段着色器中比较深度时有不同：
+ 判断点位于哪张深度贴图：
  - 需要摄像机观察坐标系下的顶点坐标$p_e$（从顶点着色器传入），将$p_e$的深度值，与计算出的$z_i$进行比较，判断顶点位于哪一个深度贴图中
  - 需要世界坐标系下的顶点坐标（从顶点坐标系传入，或者根据摄像机观察矩阵的逆矩阵左乘$p_e$计算出来）$p_w$，将$p_w$通过阴影贴图的观察投影矩阵变换到阴影贴图对应的光线坐标空间下的$p_i$，再将坐标变换到$[0, 1]$范围内，根据$p_i.xy$取得深度贴图中对应点的值，与$p_i.z$进行比较

```glsl
#version 330 core

struct Light
{
    vec3 direction;

    vec3 specular;
    vec3 diffuse;
    vec3 ambient;
};

struct Material {
    sampler2D tex;
    float shininess;
};

// 分割数量
#define SPLIT_QUANTITY 3

uniform Light light;
// 各分割块远平面
uniform float z_values[SPLIT_QUANTITY];
// 各分割块的光锥投影观察矩阵
uniform mat4 light_projection_view[SPLIT_QUANTITY];
// 纹理数组
uniform sampler2DArray shadow_texel;
uniform vec3 view_pos;
uniform Material material;
uniform int floor;

in VS_OUT {
    vec4 FragPosCamera;
    vec4 FragPosWorld;
    vec3 Normal;
    vec2 TexCoord;
} fs_in;

out vec4 FragColor;

int split = 0;

float compute_shadow()
{
    // 根据摄像机空间的深度坐标决定当前顶点位于第几个分割块
    for(int i = 0; i < SPLIT_QUANTITY; i++)
    {
        if(z_values[i] >= -fs_in.FragPosCamera.z)
        {
            split = i;
            break;
        }
    }
    float bias = 0.0001;
    // 计算光线空间下的顶点坐标
    vec4 FragPosLight = light_projection_view[split] * fs_in.FragPosWorld;
    // 映射到标准设备空间
    vec3 FragPos = 0.5f * ((FragPosLight.xyz / FragPosLight.w) + 1.0f);
    float depth = FragPos.z;
    // 读取已经保存的深度
    float closest_depth = texture(shadow_texel, vec3(FragPos.xy, float(split))).r;
    // 比较深度得到是否存在阴影
    return depth <= closest_depth + bias ? 1.0f : 0.0f;
}

// 光线计算
vec3 compute_light()
{
    vec3 lightDir = light.direction;
    float distance = length(lightDir);
    lightDir = normalize(lightDir);
    float diff = max(dot(-lightDir, fs_in.Normal), 0);
    vec3 diffuse = diff * light.diffuse;

    vec3 viewDir = normalize(view_pos - fs_in.FragPosWorld.xyz);
    vec3 half = normalize(-lightDir + viewDir);
    float spec = pow(max(dot(half, fs_in.Normal), 0.0f), material.shininess);

    vec3 specular = spec * light.specular;
    vec3 ambient = light.ambient;

    float shadow = compute_shadow();
    float attenuation = 1 / (distance * distance);
    
    return (specular + diffuse) * shadow + ambient;
}

void main()
{
    vec3 texel = texture(material.tex, fs_in.TexCoord).rgb;
    texel = pow(texel, vec3(2.2));

    vec3 color = compute_light();

    // 不同分割块中的地板显示不同颜色
    if(floor == 1)
    {
        if(split == 0)
        {
            color *= vec3(1.0f, 0.0f, 0.0f);
        }
        else if(split == 1)
        {
            color *= vec3(0.0f, 1.0f, 0.0f);
        }
        else
        {
            color *= vec3(0.0f, 0.0f, 1.0f);
        }
    }
    else
    {
        color *= texel;
    }

    color = pow(color, vec3(1/2.2));
    
    FragColor = vec4(color, 1.0f);
}
```

<a id="参考"></a>

## 绘制步骤代码如下

```c++

void CSM::draw()
{
    float aspect = HEIGHT == 0 ? 0 : (float)WIDTH / HEIGHT;
    glm::mat4 view_inverse = glm::inverse(camera.getViewMatrix());

    char light_projection_view_param[30] = "light_projection_view[0]";

    // 根据平行光方向生成光线观察矩阵
    light_view = glm::lookAt(-(adj.light.direction * glm::vec3(10.0f)), glm::vec3(0.0f), glm::vec3(0.0f, 1.0f, 0.0f));

    // 更新视锥分割位置
    updateSplitZ();

    // 必须设置viewport
    glViewport(0, 0, SHADOW_MAP_WIDTH, SHADOW_MAP_HEIGHT);
    glBindFramebuffer(GL_FRAMEBUFFER, fbo);

    shadow_shader.useProgram();

    // 剔除正面防止阴影悬浮
    glCullFace(GL_FRONT);

    for (int i = 0; i < SPLIT_QUANTITY; i++)
    {
        // 绑定纹理数组的某一层到帧缓冲的深度缓冲
        glFramebufferTextureLayer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, texture_array, 0, i);
        glClear(GL_DEPTH_BUFFER_BIT);

        // 计算第i个视锥分块对应的光线的投影*观察矩阵
        computeLightProjection(i, aspect, view_inverse);
        // 投影观察矩阵传入着色器
        shadow_shader.setUniformMatrix4("projection_view", light_projection_view[i]);

        // 绘制场景
        drawScence(shadow_shader);
    }

    glBindFramebuffer(GL_FRAMEBUFFER, 0);
    glViewport(0, 0, WIDTH, HEIGHT);
    glClear(GL_DEPTH_BUFFER_BIT | GL_COLOR_BUFFER_BIT);

    shader.useProgram();
    glCullFace(GL_BACK);
    shader.setUniformMatrix4("projection", projection);

    for (int j = 0; j < SPLIT_QUANTITY; j++)
    {
        light_projection_view_param[22] = j + 48;
        // 光线的投影观察矩阵传入着色器
        shader.setUniformMatrix4(light_projection_view_param, light_projection_view[j]);
    }
    
    // 绘制场景
    drawScence(shader);

    // ...
    // 其他操作
}
```

## PCF

片段着色器中加3*3的pcf如下

```glsl
vec2 offsets[9] = vec2[](
    vec2(1.0f, 1.0f),
    vec2(1.0f, -1.0f),
    vec2(1.0f, 0.0f),
    vec2(0.0f, 1.0f),
    vec2(0.0f, -1.0f),
    vec2(0.0f, 0.0f),
    vec2(-1.0f, 1.0f),
    vec2(-1.0f, -1.0f),
    vec2(-1.0f, 0.0f)
);

float compute_shadow()
{
    int samplers = 9;
    for(int i = 0; i < SPLIT_QUANTITY; i++)
    {
        if(z_values[i] >= -fs_in.FragPosCamera.z)
        {
            split = i;
            break;
        }
    }
    float bias = 0.0001;
    vec4 FragPosLight = light_projection_view[split] * fs_in.FragPosWorld;
    vec3 FragPos = 0.5f * ((FragPosLight.xyz / FragPosLight.w) + 1.0f);
    float depth = FragPos.z;
    ivec3 tsize = textureSize(shadow_texel, 0);
    float shadow = 0.0f;
    for(int j = 0; j < 9; j++)
    {
        vec2 offset = vec2(1.0f / tsize.x, 1.0f / tsize.y) * offsets[j];
        float closest_depth = texture(shadow_texel, vec3(FragPos.xy + offset, float(split))).r;
        shadow += depth <= closest_depth + bias ? 1.0f : 0.0f;
    }
    return shadow / 9.0f;
}
```

最终结果

<div style='text-align:center' id='img2-5'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/csm_pcf.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-5: csm的最终实现结果</span></p>
</div>

## 仍然存在的问题

1. 某些情况下平移摄像机会导致阴影抖动比较明显
2. 视锥分块交界处可以看到比较明显的断层

前面为了弄清楚一些细节，实在在csm耽误了太多时间了，加上这两个问题不是特别容易解决，故暂时跳过，留待之后处理，毕竟还想在今年上半年学完Learn OpenGL，下半年做些实践呢

## 函数

用到几个没用过的api

`void glTexImage3D(GLenum target, GLint level, GLint internalformat, GLsizei width, GLsizei height, GLsizei depth, GLint border, GLenum format, GLenum type, const void * data);`

+ 绑定3d纹理的数据
+ target: 目标纹理类型，可以是`GL_TEXTURE_3D`, `GL_PROXY_TEXTURE_3D`, `GL_TEXTURE_2D_ARRAY` 或 `GL_PROXY_TEXTURE_2D_ARRAY`
+ level: 多级渐远纹理级别
+ internalformat: 颜色组件格式
+ width,height:宽高
+ depth:3d纹理的深度或者纹理数组的层次
+ border:必须为0
+ format:像素数据的格式，可以是`GL_RED`, `GL_RG`, `GL_RGB`, `GL_BGR`, `GL_RGBA`, `GL_BGRA`, `GL_RED_INTEGER`, `GL_RG_INTEGER`, `GL_RGB_INTEGER`, `GL_BGR_INTEGER`, `GL_RGBA_INTEGER`, `GL_BGRA_INTEGER`, `GL_STENCIL_INDEX`, `GL_DEPTH_COMPONENT`, `GL_DEPTH_STENCIL`
+ type: 像素数据的类型，可以是`GL_UNSIGNED_BYTE`, `GL_BYTE`, `GL_UNSIGNED_SHORT`, `GL_SHORT`, `GL_UNSIGNED_INT`, `GL_INT`, `GL_HALF_FLOAT`, `GL_FLOAT`, `GL_UNSIGNED_BYTE_3_3_2`等等...
+ data: 数据，可以设为NULL仅申请空间

`void glFramebufferTextureLayer(GLenum target, GLenum attachment, GLuint texture, GLint level, GLint layer);`

+ 用来将3d纹理或者2d纹理数组的某一层绑定到指定帧缓冲的逻辑缓冲之一。
+ target:指定的帧缓冲
+ attatchment:要绑定到的帧缓冲绑定点
+ texture:纹理id,可指定的纹理类型为3d纹理、2d纹理数组、立方体贴图数组等等
+ level:多级渐远纹理级别
+ layer:要绑定的纹理层次

因为用到2d纹理数组，着色器中取深度的方法和之前也不太一样，同样调用`texture`函数，但是第二个参数是三维向量

`float closest_depth = texture(shadow_texel, vec3(FragPos.xy, float(split))).r;`

# 参考

1. [Cascaded Shadow Maps(CSM)实时阴影的原理与实现](https://zhuanlan.zhihu.com/p/53689987) authored by [fangcun](https://www.zhihu.com/people/jie-ren-35)@[知乎](https://www.zhihu.com)
2. [联级阴影贴图 CSM(Cascaded shadow map)原理与实现](https://blog.csdn.net/qq_39300235/article/details/107765941) authored by [0 小龙虾 0](https://blog.csdn.net/qq_39300235)@[csdn](https://www.csdn.net/)
3. [Cascaded Shadow Maps](https://developer.download.nvidia.com/SDK/10.5/opengl/src/cascaded_shadow_maps/doc/cascaded_shadow_maps.pdf) by NVIDIA Corporation
4. [Stamminger, Marc, and George Drettakis. “Perspective Shadow Maps.” Proceedings of the 29th Annual Conference on Computer Graphics and Interactive Techniques, vol. 21, no. 3, 2002, pp. 557–562](https://academic.microsoft.com/paper/2244302129)
5. [Wimmer, Michael, et al. “Light Space Perspective Shadow Maps.” EGSR’04 Proceedings of the Fifteenth Eurographics Conference on Rendering Techniques, 2004, pp. 143–151.](https://academic.microsoft.com/paper/1489229580/reference?showAllAuthors=1)
6. [Zhang, Fan, et al. “Parallel-Split Shadow Maps for Large-Scale Virtual Environments.” Proceedings of the 2006 ACM International Conference on Virtual Reality Continuum and Its Applications, 2006, pp. 311–318.](https://academic.microsoft.com/paper/2089935131/reference?showAllAuthors=1)
7. [用 DX11 实现 Cascaded shadow map](https://zhuanlan.zhihu.com/p/116731971?from_voters_page=true) authored by [宋开心](https://www.zhihu.com/people/Arcsong)@[知乎](https://www.zhihu.comF)
8. [《Cascaded Shadow Map(CSM)中的一些问题》](https://blog.csdn.net/pizi0475/article/details/7933743) authored by [pizi0475](https://blog.csdn.net/pizi0475)@[csdn](https://www.csdn.net/)
9. [opengl 4.x reference pages](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/)