---
title: opengl(11):法线贴图，视差贴图
date: 2021/05/15
tags:
  - opengl
---

为学懂 webgl，从 opengl 入手。本文内容主要来自[LearnOpenGL CN](https://learnopengl-cn.github.io/)文档，加上个人理解，以及一些细节补充、经验记录。本人学习过程中的详细代码参见[此处](https://github.com/itsnoteasytonameaccount/OpenGL-learning)

要学习推荐看原文[LearnOpenGL CN](https://learnopengl-cn.github.io/)，有评论区，加载评论区需要翻墙

另可在[此处](https://www.khronos.org/opengl/wiki/Getting_Started)找到其他 opengl 官方推荐的资料、教程网站

<!-- more -->

# 法线贴图

法线贴图比较好理解，之前我们用到纹理的地方，法向量都是和纹理所在平面垂直的，这样计算出来的光线都相当于在平坦表面上的光线。若想用纹理显示砖墙这类粗糙表面，仍然采用垂直的法向量就会导致砖墙缺乏质感、真实感，因为粗糙表面某点的法向量是可能向各个方向发散的。

<div style='text-align:center' id='img1-1'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/normal_mapping_flat.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图1-1: 如图由于法向量不正确，导致墙面光照看起来不真实</span></p>
</div>

为了解决这个问题引入了法线贴图，将某个面切线空间（tangent space）的法向量经过一定处理后将各分量限制在$[0,1]$范围内，再将数据存在图片的 rgb 数据中。如下是砖墙的法线贴图，由于切线空间法向量的 z 分量值都在 z 轴正向，所以颜色偏紫。

<div style='text-align:center' id='img1-2'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/normal_mapping_normal_map.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图1-2: 法线贴图</span></p>
</div>

取法向量的时候就可以通过 texture 函数取

```glsl
vec3 normal = vec3(texture(normal_texture, TexCoord));
normal = normalize(normal * 2.0f - 1.0f);
```

使用法线贴图后在砖墙上计算光线的感觉就不一样了

<div style='text-align:center' id='img1-3'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/normal_mapping_correct.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图1-3: 运用法线贴图后砖墙上光线计算</span></p>
</div>

不过这时候还存在一个问题，目前取到的法向量是位于切线空间下的，不转化到世界空间下会导致多数情况下的光线计算仍然不正确，因此需要有一种算法转化法向量

<div style='text-align:center' id='img1-4'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/normal_mapping_ground_normals.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图1-4: 法向量不正确</span></p>
</div>

## 切线空间

法线贴图的法向量是位于切线空间下的，在切线空间下的法线总是大致指向 z 轴正向。
切线空间是相对于三角形所表示平面的空间，其 xy 轴都位于平面上，z 轴与平面法向量同向。

由于法线贴图保存的法向量数据都位于切线空间，因此需要构建一个转化矩阵来将之转化到世界坐标系下。

求法思路：
设$N_t$为某平面的切线空间法向量，$N_w$为某平面的世界空间法向量，有

$$
M * N_w = N_t \implies N_w = M^{-1} * N_t
$$

矩阵 M 是将世界坐标系转化为法线贴图切线坐标系的矩阵。
结合我们在 opengl(3)中的转换坐标系一节中已有提及的求矩阵的方法可知，要求矩阵 M，只要求出世界坐标系下的如下图中三个单位向量 TBN 即可。

<div style='text-align:center' id='img1-5'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/normal_mapping_tbn_vectors.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图1-5: 为求转化矩阵需要求的向量</span></p>
</div>

其中 N 已知为平面法向量，重点是求 B、T。

求 BT 的前提条件是知道平面上的任意不共线三点在世界坐标系以及切线坐标系下的坐标值，然后有步骤如下：
根据三点的世界坐标系下坐标可求出不共线向量$E_1$，$E_2$，根据三点的切线坐标系下的坐标值可分别求出$E_1,E_2$两向量在 B、T 上的投影长度（投影和 B、T 反向时为长度负值）$\Delta U_1, \Delta V_1, \Delta U_2, \Delta V_2$

<div style='text-align:center' id='img1-6'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/normal_mapping_surface_edges.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图1-6: 计算T、B需要的条件</span></p>
</div>

用 TB 来表示$E_1,E_2$有等式

$$
\begin{cases}
E_1 = \Delta V_1 * B + \Delta U_1 T\\
E_2 = \Delta V_2 * B + \Delta U_2 T
\end{cases}
$$

或者用矩阵乘法表示

$$
\begin{pmatrix}
E_{1x} & E_{1y} & E_{1z}\\
E_{2x} & E_{2y} & E_{2z}
\end{pmatrix} =
\begin{pmatrix}
\Delta V_1 & \Delta U_1\\
\Delta V_2 & \Delta U_2
\end{pmatrix}
\begin{pmatrix}
B_x & B_y & B_z\\
T_x & T_y & T_z
\end{pmatrix}
$$

所以

$$
\begin{pmatrix}
B_x & B_y & B_z\\
T_x & T_y & T_z
\end{pmatrix} =
\begin{pmatrix}
\Delta V_1 & \Delta U_1\\
\Delta V_2 & \Delta U_2
\end{pmatrix}^{-1}
\begin{pmatrix}
E_{1x} & E_{1y} & E_{1z}\\
E_{2x} & E_{2y} & E_{2z}
\end{pmatrix}
$$

即

$$
\begin{pmatrix}
B_x & B_y & B_z\\
T_x & T_y & T_z
\end{pmatrix} = \frac{1}{\Delta V_1 \Delta U_2 - \Delta U_1 \Delta V_2}
\begin{pmatrix}
\Delta U_2 & -\Delta U_1\\
-\Delta V_2 & \Delta V_1
\end{pmatrix}
\begin{pmatrix}
E_{1x} & E_{1y} & E_{1z}\\
E_{2x} & E_{2y} & E_{2z}
\end{pmatrix}
$$

可以得

$$
B_x = \frac{\Delta U_2 E_{1x} - \Delta U_1 E_{2x}}{\Delta V_1 \Delta U_2 - \Delta U_1 \Delta V_2}
$$

其他坐标值计算方式相同

最终要求的矩阵 M 如下，显然 M 是正交矩阵（行向量组是单位正交向量组），有$M^{-1} = M^{\tau}$(转置矩阵的运算成本比逆矩阵小多了，因此在程序中计算$M^{-1}$时可以用$M^{\tau}$代替)

$$
\begin{pmatrix}
T_x & T_y & T_z\\
B_x & B_y & B_z\\
N_x & N_y & N_z
\end{pmatrix}
$$

## 代码中计算

在代码中计算 TBN 矩阵，首先要知道如何取$E_1, E_2, \Delta U_1, \Delta V_1, \Delta U_2, \Delta V_2$等值，opengl 中绘图方式是绘制一个个三角形面，因此上述六个值通过三角形三个顶点的世界坐标和切线空间坐标来计算即可。对于砖墙来说，有四个顶点分别组成两个三角形面，如下。

```c++
    // 世界坐标
    glm::vec3 pos1(-1.0, 1.0, 0.0);
    glm::vec3 pos2(-1.0, -1.0, 0.0);
    glm::vec3 pos3(1.0, -1.0, 0.0);
    glm::vec3 pos4(1.0, 1.0, 0.0);
    // 切线空间坐标即各顶点纹理坐标
    glm::vec2 uv1(0.0, 1.0);
    glm::vec2 uv2(0.0, 0.0);
    glm::vec2 uv3(1.0, 0.0);
    glm::vec2 uv4(1.0, 1.0);
```

由此可以得到向量$E_1, E_2$，以及值$\Delta U_1, \Delta V_1, \Delta U_2, \Delta V_2$如下

```c++
    // E1
    glm::vec3 edge1 = pos2 - pos1;
    // E2
    glm::vec3 edge2 = pos3 - pos1;
    // U1和V1的值
    glm::vec2 deltaUV1 = uv2 - uv1;
    // U2和V2的值
    glm::vec2 deltaUV2 = uv3 - uv1;
```

取到值后就按照上面的公式计算即可，计算出结果后传入着色器即可

着色器中代码

```glsl
#version 330 core

layout (location = 0) in vec3 aFragPos;
layout (location = 1) in vec3 aNormal;
layout (location = 2) in vec2 aTextCoord;
layout (location = 3) in vec3 tangent;
layout (location = 4) in vec3 bitangent;

uniform mat4 projection;
uniform mat4 model;
uniform mat4 view;
uniform mat3 normal_matrix;

out VS_OUT {
    vec3 FragPos;
    vec3 Normal;
    vec2 TexCoord;
    mat3 TBN;
} vs_out;

void main()
{
  // ...
   vec3 T = normalize(normal_matrix * tangent);
   vec3 B = normalize(normal_matrix * bitangent);
   vec3 N = normalize(normal_matrix * aNormal);
   // 注意这个mat构造的矩阵已经是转置后的矩阵了
   vs_out.TBN = mat3(T, B, N);
}
```

注意上面代码中的TBN矩阵已经是转置后的如下矩阵了

$$
\begin{pmatrix}
T_x & B_x & N_x\\
T_y & B_y & N_y\\
T_z & B_z & N_z\\
\end{pmatrix}
$$

如果不放心的话可以通过如下代码测试一下

```glsl
    mat3 matrix = mat3(vec3(0.0, 1.0, 0.0), vec3(0.0), vec3(0.0));
    vec3 result = matrix * vec3(1.0);
    FragColor = vec4(result, 1.0f);
```

如果matrix是

$$
\begin{pmatrix}
0 & 1 & 0\\
0 & 0 & 0\\
0 & 0 & 0\\
\end{pmatrix}
$$

输出的应当是$(1.0, 0.0, 0.0)$红色

如果matrix是

$$
\begin{pmatrix}
0 & 0 & 0\\
1 & 0 & 0\\
0 & 0 & 0\\
\end{pmatrix}
$$

输出的应当是$(0.0, 1.0, 0.0)$绿色

实际结果是绿色

全部代码可参见文章开头给出的地址。

## 复杂物体

assimp可以通过载入模型时假如aiProcess_CalcTangentSpace选项来为每个顶点计算切线和副切线

```c++
const aiScene* scene = importer.ReadFile(
    path, aiProcess_Triangulate | aiProcess_FlipUVs | aiProcess_CalcTangentSpace
);
```

加载完后使用

```c++
    void setupMesh()
    {
        bool hasNormals = mesh->HasNormals();
        long long verticesSize = mesh->mNumVertices * sizeof(C_STRUCT aiVector3D);
        long long normalsSize = hasNormals ? verticesSize : 0;
        long long texturesSize = mesh->mTextureCoords[0] == nullptr ? mesh->mNumVertices * 2 * sizeof(float) : verticesSize;
        long long facesSize = 0;
        glGenVertexArrays(1, &VAO);
        glGenBuffers(1, &VBO);
        glGenBuffers(1, &EBO);
        glBindVertexArray(VAO);
        glBindBuffer(GL_ARRAY_BUFFER, VBO);
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
        glBufferData(GL_ARRAY_BUFFER, verticesSize * 3 + normalsSize + texturesSize, nullptr, GL_STATIC_DRAW);

        // ...

        if (mesh->mTangents != NULL)
        {
            glEnableVertexAttribArray(3);
            glBufferSubData(GL_ARRAY_BUFFER, verticesSize + normalsSize + texturesSize, verticesSize, mesh->mTangents);
            glVertexAttribPointer(3, 3, GL_FLOAT, GL_FALSE, sizeof(C_STRUCT aiVector3D), (void *)(verticesSize + normalsSize + texturesSize));
        }

        if (mesh->mBitangents != NULL)
        {
            glEnableVertexAttribArray(4);
            glBufferSubData(GL_ARRAY_BUFFER, verticesSize * 2 + normalsSize + texturesSize, verticesSize, mesh->mBitangents);
            glVertexAttribPointer(4, 3, GL_FLOAT, GL_FALSE, sizeof(C_STRUCT aiVector3D), (void *)(verticesSize * 2 + normalsSize + texturesSize));
        }

        // ...

        glBindVertexArray(0);
        glBindBuffer(GL_ARRAY_BUFFER, 0);
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);
    };
    /* data */
```

最后下面是使用了法线贴图和没使用法线贴图的对比结果

<div style='text-align:center' id='img1-7'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/normal_mapping_compare.gif' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图1-7: 法线贴图使用与否效果对比</span></p>
</div>

<a id="参考"></a>

# 视差贴图

视差贴图比较好理解，就是把切线空间中的顶点深度存放在图片里，然后绘制时根据深度来决定所取纹理坐标的偏移量。（这样做的理由：比较深的顶点可能会被视线越过，此时看到的是附近的其他顶点）

## 简单实现

最简单，但实际上不太合理的做法是根据当前顶点处对应的深度来取偏移量，如果当前顶点深度越深，取的偏移量越大，这样做的原因可能是：深度一般是渐变的，比较深的顶点附近的顶点深度也比较大视线同样会被视线越过。

<div style='text-align:center' id='img2-1'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/parallax_mapping_depth.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-1: 原理</span></p>
</div>

### 代码实现

纹理坐标偏移的方向是根据视线方向决定的，因此需要将视线方向向量转换到到切线空间以方便计算纹理坐标。

将坐标从世界空间转换到切线空间的矩阵是法线贴图中TBN矩阵的转置。
顶点着色器中如下

```glsl
#version 330 core
// ...

out VS_OUT {
    vec3 FragPos;
    vec2 TexCoords;
    vec3 TangentLightPos;
    vec3 TangentViewPos;
    vec3 TangentFragPos;
} vs_out;

// ...

void main()
{
    // ...
    mat3 TBN = transpose(mat3(T, B, N));

    // 也可以不计算光线的切线空间坐标，直接用原来方法计算光线即可
    vs_out.TangentLightPos = TBN * lightPos;
    vs_out.TangentViewPos  = TBN * viewPos;
    vs_out.TangentFragPos  = TBN * vs_out.FragPos;
}   
```

之后在片段着色器中加入计算纹理坐标的方法

```glsl
    vec2 texCoords = ParallaxMapping(fs_in.TexCoords,  viewDir);
```

其中`ParallaxMapping`函数实现如下

```glsl
vec2 ParallaxMapping(vec2 texCoords, vec3 viewDir)
{ 
    float height =  texture(depthMap, texCoords).r;    
    // height_scale用来缩放，避免效果过于强烈，除以z的作用是，使得视线与平面角度越小时，偏移量越大。
    vec2 p = viewDir.xy / viewDir.z * (height * height_scale);
    return texCoords - p;    
} 
```

边缘会有一些超出纹理坐标范围的采样，可以丢弃

```glsl
texCoords = ParallaxMapping(fs_in.TexCoords,  viewDir);
if(texCoords.x > 1.0 || texCoords.y > 1.0 || texCoords.x < 0.0 || texCoords.y < 0.0)
    discard;
```

这种做法还是很容易出问题的。

## 陡峭视差映射

如下是我的理解方式：
陡峭视差映射的做法是沿着视线方向每次间隔一定深度一个个地取点，不断重复直到视线穿过物体表面（视线从高于表面上的点，到低于表面上的点），将最终的点的xy坐标作为纹理坐标。

<div style='text-align:center' id='img2-2'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/parallax_mapping_steep_parallax_mapping_diagram.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-2: 陡峭视差映射方法</span></p>
</div>

片段着色器中实现如下

```glsl
vec2 comput_texcoord(vec2 TexCoord)
{
    // 我的代码里把摄像机位置作为原点，因此省略了摄像机坐标
    vec3 view_dir = normalize(-fs_in.TangentFragPos);
    if(view_dir.z == 0.0f)
    {
        return TexCoord;
    }
    // 将0-1深度范围分成很多层
    float depth_change_rate = 1.0f / divide;
    // 由于是沿着视线方向，因此xy的变化率需要按照深度的变化率来缩放
    vec2 direction = view_dir.xy * depth_change_rate / view_dir.z;

    float current_layer_depth = 0.0f;
    float current_map_depth = texture(material.texture_disp, TexCoord).r;
    vec2 current_texcoord = TexCoord;

    // 逐层比较直至视线穿过表面
    while(current_layer_depth < current_map_depth)
    {
        current_texcoord -= direction;
        current_map_depth = texture(material.texture_disp, current_texcoord).r;
        current_layer_depth += depth_change_rate;
    }

    // 返回得到的纹理坐标
    return current_texcoord;
}
```

这样最终的到的效果如下

<div style='text-align:center' id='img2-3'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/steep_parallax_mapping_effect1.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-3: 陡峭视差贴图效果</span></p>
</div>

这样看起来砖墙缝隙处深度有点过大了，主要原因是随着高度变化的偏移量设置得略大，因此添加一个参数适当调整一下

```glsl
    // 也可以直接把系数depth_change_rate / view_dir.z * height_scale合并成一个height_scale
    vec2 direction = view_dir.xy * depth_change_rate / view_dir.z * height_scale;
```

再次运行看起来效果就好很多了

<div style='text-align:center' id='img2-4'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/steep_parallax_mapping_effect2.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-4: 添加参数后的效果</span></p>
</div>

也可以通过视线方向来调整分层数量，视线垂直平面时，需要的分层数更少

```glsl
const float minLayers = 8.0;
const float maxLayers = 32.0;
// numLayers对应上面代码里的divide
float numLayers = mix(maxLayers, minLayers, max(dot(vec3(0.0, 0.0, 1.0), viewDir), 0.0));  
```

不过这样存在的问题是从侧面看时存在明显的分层现象如下

<div style='text-align:center' id='img2-5'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/steep_parallax_mapping_problem.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-5: 存在的问题</span></p>
</div>

因此需要引入接下来的内容

## 视差遮蔽映射

事实上，取视线穿越表面前后的两点，根据两点与相应顶点的深度差距计算权重，取到平均位置(下图中红色交叉点)，这样取到的位置更接近视线实际能看到的点的位置，因此需要在陡峭阴影映射的基础上增加操作。

<div style='text-align:center' id='img2-6'>

<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/parallax_mapping_parallax_occlusion_mapping_diagram.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-6: 视差遮蔽映射</span></p>
</div>

平均位置的取法($P_b,P_a$分别为穿越表面前后的点的坐标，$\lambda$根据[图2-6](#img2-6)中两三角形相似取得)

$$
(P_a - P_b) * \lambda_1 + P_b
$$

或者

$$
P_a - (P_a - P_b) * \lambda_2
$$

需要修改片段着色器

```glsl
vec2 comput_texcoord(vec2 TexCoord)
{
    vec3 view_dir = normalize(-fs_in.TangentFragPos);
    if(view_dir.z == 0.0f)
    {
        return TexCoord;
    }
    float depth_change_rate = 1.0f / divide;
    vec2 direction = view_dir.xy * depth_change_rate / view_dir.z * height_scale;
    float current_layer_depth = 0.0f;
    float current_map_depth = texture(material.texture_disp, TexCoord).r;
    float prev_map_depth = current_map_depth;
    vec2 current_texcoord = TexCoord;
    while(current_layer_depth < current_map_depth)
    {
        // 保存上一次的顶点深度
        prev_map_depth = current_map_depth;
        current_texcoord -= direction;
        current_map_depth = texture(material.texture_disp, current_texcoord).r;
        current_layer_depth += depth_change_rate;
    }

    // 计算视线穿过后的深度差距
    float after_depth = current_map_depth - current_layer_depth;
    // 计算视线穿过前的深度差距
    float before_depth = prev_map_depth - current_layer_depth + depth_change_rate;

    float weight = after_depth / (after_depth - before_depth);
    // 计算平均位置，direction对应上面的Pa-Pb
    current_texcoord = current_texcoord + weight * direction; 
    return current_texcoord;
}
```

最终效果

<div style='text-align:center' id='img2-7'>
<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/final_effect1.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-7: 最终效果1</span></p>
</div>

<div style='text-align:center' id='img2-8'>
<p style='margin-bottom: 5px'><img style='margin-bottom: 0px' src='../image/opengl/final_effect2.png' alt='图片加载失败'/></p>
<p style='margin: 0 0 20px 0;; line-height:1.5'><span style='font-size:12px'>图2-8: 最终效果2</span></p>
</div>

# 参考

1. [Learn OpenGL](https://learnopengl.com/Advanced-Lighting/Normal-Mapping)
1. [Learn OpenGL CN](https://learnopengl-cn.github.io/05%20Advanced%20Lighting/04%20Normal%20Mapping/)
