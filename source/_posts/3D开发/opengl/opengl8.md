---
title: opengl(8):高级数据、高级glsl、几何着色器、实例化、抗锯齿
date: 2021/02/25 16:46
tags:
  - opengl
---

为学懂webgl，从opengl入手。本文内容主要来自[LearnOpenGL CN](https://learnopengl-cn.github.io/)文档，加上个人理解，以及一些细节补充、经验记录。本人学习过程中的详细代码参见[此处](https://github.com/itsnoteasytonameaccount/OpenGL-learning)

要学习推荐看原文[LearnOpenGL CN](https://learnopengl-cn.github.io/)，有评论区，加载评论区需要翻墙

另可在[此处](https://www.khronos.org/opengl/wiki/Getting_Started)找到其他opengl官方推荐的资料、教程网站
<!-- more -->

# 高级数据

这部分比较简单，只是介绍了几个简单的数据操作函数

`void glBufferSubData(GLenum target, GLintptr offset, GLsizeiptr size, const void *data)`
用于分块填入数据，使用之前必须通过`glBufferData`创建了内存区域。

- target: 目标缓冲区类型
- offset: 起始位置
- size: 填入数据大小
- data: 数据列表指针

`void * glMapBuffer(GLenum target, GLenum access)`
用于生成缓冲区的指针

- target: 目标缓冲区类型
- access: 读写权限，可取`GL_READ_ONLY,GL_WRITE_ONLY,GL_READ_WRITE`
- @return: 生成的首地址指针

`void * glUnmapBuffer(GLenum target, GLenum access)`
用于无效`glMapBuffer`生成的指针

- target: 目标缓冲区类型

`void glCopyBufferSubData(GLenum readTarget, GLenum writeTarget, GLintptr readOffset, GLintptr writeOffset, GLsizeiptr size)`
用于缓冲区间复制数据，使用前可先用`glBindBuffer(GL_COPY_WRITE_BUFFER, vbo1),glBindBuffer(GL_COPY_READ_BUFFER, vbo2)`等操作绑定缓冲区

- readTarget: 源缓冲区类型
- writeTarget: 目标缓冲区类型
- readOffset: 源起始位置
- writeOffset: 目标起始位置
- size: 复制数据大小

## 示例

可以修改之前 mesh 的处理方式，省去大量的遍历操作

```c++
Mesh::Mesh(aiMesh *mesh, vector<mesh::Texture> &textures, mesh::Material &material, bool has_texture)
{
    this->mesh = mesh;
    this->textures = textures;
    this->material = material;
    this->has_texture = has_texture;

    setupMesh();
}

void Mesh::setupMesh()
{
    bool hasNormals = mesh->HasNormals();
	// 点数据大小
    long long verticesSize = mesh->mNumVertices * sizeof(C_STRUCT aiVector3D);
	// 法线数据大小
    long long normalsSize = hasNormals ? verticesSize : 0;
	// 纹理坐标数据大小
    long long texturesSize = mesh->mTextureCoords[0] == nullptr ? mesh->mNumVertices * 2 * sizeof(float) : verticesSize;
    long long facesSize = 0;

	// 绑定
    glGenVertexArrays(1, &VAO);
    glGenBuffers(1, &VBO);
    glGenBuffers(1, &EBO);
    glBindVertexArray(VAO);
    glBindBuffer(GL_ARRAY_BUFFER, VBO);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);

	// 申请特定大小的空缓冲区域
    glBufferData(GL_ARRAY_BUFFER, verticesSize + normalsSize + texturesSize, nullptr, GL_STATIC_DRAW);

    glBufferSubData(GL_ARRAY_BUFFER, 0, verticesSize, mesh->mVertices);

	// 填入顶点数据
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(C_STRUCT aiVector3D), (void *)0);

	// 存在法线时才启用顶点属性数组
    if (hasNormals)
    {
        glBufferSubData(GL_ARRAY_BUFFER, verticesSize, normalsSize, mesh->mNormals);
        glEnableVertexAttribArray(1);
        glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(C_STRUCT aiVector3D), (void *)verticesSize);
    }

    glEnableVertexAttribArray(2);
	// 判断第一组纹理坐标是否存在，不存在时把每个顶点的纹理坐标设为(0.0f, 0.0f)
    if (mesh->mTextureCoords[0] == nullptr)
    {
        char *ptr = (char *)glMapBuffer(GL_ARRAY_BUFFER, GL_WRITE_ONLY);
        memset(ptr + verticesSize + normalsSize, 0, sizeof(texturesSize));
        glUnmapBuffer(GL_ARRAY_BUFFER);
        glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 0, (void *)(verticesSize + normalsSize));
    }
	// 存在直接填入
    else
    {
        glBufferSubData(GL_ARRAY_BUFFER, verticesSize + normalsSize, texturesSize, mesh->mTextureCoords[0]);
        glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, sizeof(C_STRUCT aiVector3D), (void *)(verticesSize + normalsSize));
    }

	// 计算索引数，并保存供drawElements使用
    for (int i = 0; i < mesh->mNumFaces; i++)
    {
        indicesNum += mesh->mFaces[i].mNumIndices;
    }
	// 索引数据尺寸
    facesSize = indicesNum * sizeof(unsigned int);
	// 申请空内存
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, facesSize, nullptr, GL_STATIC_DRAW);

	// 将所有索引数据填入缓冲
    for (int j = mesh->mNumFaces - 1; j > -1; j--)
    {
        long long indicesSize = mesh->mFaces[j].mNumIndices * sizeof(unsigned int);
        facesSize -= indicesSize;
        glBufferSubData(GL_ELEMENT_ARRAY_BUFFER, facesSize, indicesSize, mesh->mFaces[j].mIndices);
    }

    glBindVertexArray(0);
    glBindBuffer(GL_ARRAY_BUFFER, 0);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);
}

// ...
```

# 高级 glsl

## 内建变量

**顶点着色器**

`gl_PointSize`
顶点着色器中控制顶点大小，需要用`glEnable(GL_PROGRAM_POINT_SIZE);`启用

要看到效果，绘制的类型选择`GL_POINTS`

`gl_VertexID`

> 整型变量 gl_VertexID 储存了正在绘制顶点的当前 ID。当（使用 glDrawElements）进行索引渲染的时候，这个变量会存储正在绘制顶点的当前索引。当（使用 glDrawArrays）不使用索引进行绘制的时候，这个变量会储存从渲染调用开始的已处理顶点数量。

**片段着色器**

`gl_FragCoord`

点的屏幕坐标，三维向量，xy 范围根据`glViewport`设置的值决定，$z \in [0, 1]$

可以用来根据这个值给窗左右两半部分设置不同的效果进行对比

`gl_FrontFacing`

用来判断当前面是正向面还是负向面

`gl_FragDepth`

用来在片段着色器中写入深度，但是可能对性能有较大影响，因为对这个变量写入会导致禁用提前深度测试。

opengl4.2 后可以重新声明这个变量给其限定条件

```glsl
layout (depth_<condition>) out float gl_FragDepth;
```

其中`condition`可取值如下

| 条件      | 描述                                                        |
| :-------- | :---------------------------------------------------------- |
| any       | 默认值。提前深度测试是禁用的，你会损失很多性能              |
| greater   | 你只能让深度值比 gl_FragCoord.z 更大                        |
| less      | 你只能让深度值比 gl_FragCoord.z 更小                        |
| unchanged | 如果你要写入 gl_FragDepth，你将只能写入 gl_FragCoord.z 的值 |

> 通过将深度条件设置为 greater 或者 less，OpenGL 就能假设你只会写入比当前片段深度值更大或者更小的值了。这样子的话，当深度值比片段的深度值要小的时候，OpenGL 仍是能够进行提前深度测试的。

## 接口块

简单地说就是把一组数据合并在一起，方便管理数据

举例：
顶点着色器定义输出时如下

```glsl
#version 330 core
// ...
layout (location = 2) in vec2 aTexCoords;

out VS_OUT
{
    vec2 TexCoords;
} fs_out;

void main()
{
    // ...
    fs_out.TexCoords = aTexCoords;
}
```

那么片段着色器中申明输入如下

```glsl
// 接口块的声明需要一致
in VS_OUT
{
    vec2 TexCoords;
// 接口块的变量名不许呀一致
} fs_in;

```

## uniform 缓冲对象

将一个缓冲内的数据作为`uniform`的数据，这个`uniform`是一组数据的组合。

`uniform`在着色器中声明示例如下

```glsl
#version 330 core
// ...

layout (std140) uniform Matrices
{
    mat4 projection;
    mat4 view;
};

uniform mat4 model;

void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0);
}
```

### uniform 块布局

uniform 块分配内存和 c 一样也需要字节对齐。不同 uniform 块在不同的硬件上分配的内存可能不同，有的硬件上会对 uniform 的数据大小进行补齐达到字节对齐的目的，有的硬件上会调整 Uniform 中变量的结构来达到字节对齐的目的。

默认情况下着色器使用的是叫共享布局的内存布局，每种数据的对齐量由硬件规定，并且设定对齐量后在硬件上运行的着色器共享这个对齐量规定。并且这种布局允许调整变量位置，来达到对齐的目的。

因此我们无法确切知道一个变量在 uniform 块中的确切位置。

其中一个办法是用`glGetUniformIndices`查询

`void glGetUniformIndices( GLuint program, GLsizei uniformCount, const GLchar **uniformNames, GLuint *uniformIndices);`

- program:着色器程序
- uniformCount:变量个数
- uniformName:变量名列表
- uniformIndices:用来存放获取到的索引的数组

也有通过规定布局的方法，如下，layout 后括号内跟的就是使用 std140 布局的声明

```glsl
layout (std140) uniform ExampleBlock
{
    float value;
    vec3  vector;
    mat4  matrix;
    float values[3];
    bool  boolean;
    int   integer;
};
```

std140 布局下常用变量的对齐量如下其中，(也可在[OpenGL 的 Uniform 缓冲规范]中查看(https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_uniform_buffer_object.txt)，在大概中间的位置，搜索 Examples 可以找到)，$N = 4$

| 类型                   | 布局规则                                                       |
| :--------------------- | :------------------------------------------------------------- |
| 标量，比如 int 和 bool | 每个标量的基准对齐量为 N。                                     |
| 向量                   | 2N 或者 4N。这意味着 vec3 的基准对齐量为 4N。                  |
| 标量或向量的数组       | 每个元素的基准对齐量与 vec4 的相同。                           |
| 矩阵                   | 储存为列向量的数组，每个向量的基准对齐量与 vec4 的相同。       |
| 结构体                 | 等于所有元素根据规则计算后的大小，但会填充到 vec4 大小的倍数。 |

### 使用 uniform 缓冲

创建缓冲时相同

```c++
    glGenBuffers(1, &ubo);
    glBindBuffer(GL_UNIFORM_BUFFER, ubo);
    glBufferData(GL_UNIFORM_BUFFER, 2 * sizeof(glm::mat4), NULL, GL_STATIC_DRAW);
    glBindBuffer(GL_UNIFORM_BUFFER, 0);
```

不同的是缓冲创建后不用设置顶点属性指针之类的，而是设置这个缓冲的绑定点

```c++
    glBindBufferRange(GL_UNIFORM_BUFFER, 0, ubo, 0, 2 * sizeof(glm::mat4));
```

此时 uniform 数据不是通过`glUniform`填入，而是设置绑定点

```c++
    glUniformBlockBinding(program, glGetUniformBlockIndex(program, name), bind_point);
```

> Uniform 缓冲对象比起独立的 uniform 有很多好处。
>
> 1. 一次设置很多 uniform 会比一个一个设置多个 uniform 要快很多。
> 2. 比起在多个着色器中修改同样的 uniform，在 Uniform 缓冲中修改一次会更容易一些。
> 3. 最后一个好处可能不会立即显现，如果使用 Uniform 缓冲对象的话，你可以在着色器中使用更多的 uniform。OpenGL 限制了它能够处理的 uniform 数量，这可以通过 GL_MAX_VERTEX_UNIFORM_COMPONENTS 来查询。当使用 Uniform 缓冲对象时，最大的数量会更高。所以，当你达到了 uniform 的最大数量时（比如再做骨骼动画(Skeletal Animation)的时候），你总是可以选择使用 Uniform 缓冲对象。

# 几何着色器

在顶点和片段着色器之间有一个可选的几何着色器(Geometry Shader)，几何着色器的输入是一个图元（如点或三角形）的一组顶点。几何着色器可以在顶点发送到下一着色器阶段之前对它们随意变换。然而，几何着色器最有趣的地方在于，它能够将（这一组）顶点变换为完全不同的图元，并且还能生成比原来更多的顶点。

例：

```glsl
#version 330 core
layout (points) in;
layout (line_strip, max_vertices = 2) out;

void main() {
    gl_Position = gl_in[0].gl_Position + vec4(-0.1, 0.0, 0.0, 0.0);
    EmitVertex();

    gl_Position = gl_in[0].gl_Position + vec4( 0.1, 0.0, 0.0, 0.0);
    EmitVertex();

    EndPrimitive();
}
```

如上例，片段着色器可选输入图元如下

- points：绘制`GL_POINTS`图元时。
- lines：绘制`GL_LINES`或`GL_LINE_STRIP`时
- lines_adjacency：`GL_LINES_ADJACENCY`或`GL_LINE_STRIP_ADJACENCY`
- triangles：`GL_TRIANGLES`、`GL_TRIANGLE_STRIP`或`GL_TRIANGLE_FAN`
- triangles_adjacency：`GL_TRIANGLES_ADJACENCY`或`GL_TRIANGLE_STRIP_ADJACENCY`

可选输出图元如下：

- points
- line_strip
- triangle_strip

line_strip 将一组点按次序前后相连形成的线段组，即假如有顺序为 1,2,3 的三个点，那么形成 1-2,2-3 两条线段
triangle_strip 类似 line_strip，新的点与前两个点组成三角形，即假如有顺序为 1,2,3,4 的四个点，会形成 1-2-3,2-3-4 两个三角形

上一个着色器传入的顶点通过如下结构体接收

```glsl
in gl_Vertex
{
    vec4  gl_Position;
    float gl_PointSize;
    float gl_ClipDistance[];
} gl_in[];
```

通过`EmitVertex()` `EndPrimitive()`两个指令决定输出的图元，`EmitVertex()`为提交顶点，`EndPrimitive()`为结束当前图元绘制（如果继续输出下一个点，该点作为新图元的起点。例如输出为 line_strip 时，绘制完 1,2 两点后`EndPrimitive()`，再绘制 3,4 两点后`EndPrimitive()`输出的就是 1-2,3-4 两条线段，而不是 1-2,2-3,3-4 三条相接线段）

由此可以看出上例的代码是输出了两个点决定的一条线段。

因此实现将三角形按其所在面的法线平移一段距离的集合着色器如下

```glsl
#version 330 core

layout (triangles) in;
layout (triangle_strip, max_vertices = 3) out;

in VS_OUT {
    vec2 texCoords;
    // 顶点的法线不是三角形的法线，不能用
    vec4 normal;
} gs_in[];

out vec2 TexCoords;
out vec4 Normal;
uniform float time;

// 通过三角形边向量叉乘获取法向量，需要注意向量方向
vec3 GetNormal()
{
    vec3 vector1 = vec3(gl_in[0].gl_Position) - vec3(gl_in[1].gl_Position);
    vec3 vector2 = vec3(gl_in[2].gl_Position) - vec3(gl_in[1].gl_Position);
    return normalize(cross(vector1, vector2));
}

// 位移
vec4 explode(vec4 position, vec3 normal)
{
    float magnitude = 0.3;
    vec3 direction = normal * ((sin(time) + 1.0) / 2.0) * magnitude;
    return position + vec4(direction, 0.0);
}

void main()
{
    vec3 normal = GetNormal();

    // 三个点沿着三角形面法向量位移并提交
    gl_Position = explode(gl_in[0].gl_Position, normal);
    TexCoords = gs_in[0].texCoords;
    Normal = gs_in[0].normal;
    EmitVertex();
    gl_Position = explode(gl_in[1].gl_Position, normal);
    TexCoords = gs_in[1].texCoords;
    Normal = gs_in[1].normal;
    EmitVertex();
    gl_Position = explode(gl_in[2].gl_Position, normal);
    TexCoords = gs_in[2].texCoords;
    Normal = gs_in[2].normal;
    EmitVertex();

    // 结束图元绘制
    EndPrimitive();
}
```

显示法线的几何着色器如下

```glsl
#version 330 core

layout (triangles) in;
layout (line_strip, max_vertices = 6) out;

in VS_OUT {
    vec2 texCoords;
    vec4 normal;
} gs_in[];

out vec2 TexCoords;
out vec4 Normal;
uniform float time;

void GenerateLine(int index)
{
    // 选取过顶点并且和该点法向量平行的直线上的两点
    gl_Position = gl_in[index].gl_Position;
    EmitVertex();
    gl_Position = gl_in[index].gl_Position + vec4(gs_in[index].normal.xyz, 0.0) * 0.05;
    EmitVertex();

    // 结束单个图元绘制
    EndPrimitive();
}

void main()
{
    // 三个顶点的法线都要绘制
    GenerateLine(0);
    GenerateLine(1);
    GenerateLine(2);
}
```

效果图如下

![图片无法显示](../image/opengl/show_normal.png)
![图片无法显示](../image/opengl/object_blast.png)

# 实例化

使用`glDrawArraysInstanced`和`glDrawElementsInstanced`能有效减少 cpu 和 gpu 间的通信，此时顶点着色器中可以读取到名为 gl_InstanceID 的变量，该变量的值为当前实例序号，从 0 开始。

两函数定义如下

`void glDrawElementsInstanced(GLenum mode, GLsizei count, GLenum type, const void *indices, GLsizei instancecount)`

- 根据索引缓冲对象绘制多个实例
- mode:图元类型，`GL_POINTS`, `GL_LINE_STRIP`, `GL_LINE_LOOP`, `GL_LINES`, `GL_TRIANGLE_STRIP`, `GL_TRIANGLE_FAN`, `GL_TRIANGLES`, `GL_QUAD_STRIP`, `GL_QUADS`, `GL_POLYGON`
- type:索引数据类型，`GL_UNSIGNED_BYTE`, `GL_UNSIGNED_SHORT`, or `GL_UNSIGNED_INT`.
- indices: 索引数组的指针，或者设为 0（绑定了 EBO 的前提下）
- instancecount: 实例数量

`void glDrawArraysInstanced(GLenum mode, GLint first, GLsizei count, GLsizei instancecount)`

- 根据顶点缓冲对象绘制多个实例
- mode:图元类型，`GL_POINTS`, `GL_LINE_STRIP`, `GL_LINE_LOOP`, `GL_LINES`, `GL_TRIANGLE_STRIP`, `GL_TRIANGLE_FAN`, `GL_TRIANGLES`, `GL_QUAD_STRIP`, `GL_QUADS`, `GL_POLYGON`.
- first:第一个点索引
- count:点个数

有了这两个函数可以将每个实例的模型矩阵存放在一个 uniform 数组中，在代码中逐个设置好后，在顶点着色器中通过`gl_InstanceID`来获取对应的矩阵。

**顶点着色器中**

```glsl
#version 330 core
layout (location = 0) in vec2 aPos;
layout (location = 1) in vec3 aColor;

out vec3 fColor;

uniform vec2 offsets[100];

void main()
{
    vec2 offset = offsets[gl_InstanceID];
    gl_Position = vec4(aPos + offset, 0.0, 1.0);
    fColor = aColor;
}
```

**程序代码中**

```c++
// ...
// 生成偏移量数据
glm::vec2 translations[100];
int index = 0;
float offset = 0.1f;
for(int y = -10; y < 10; y += 2)
{
    for(int x = -10; x < 10; x += 2)
    {
        glm::vec2 translation;
        translation.x = (float)x / 10.0f + offset;
        translation.y = (float)y / 10.0f + offset;
        translations[index++] = translation;
    }
}
// ...
// 设置uniform数组
shader.use();
for(unsigned int i = 0; i < 100; i++)
{
    stringstream ss;
    string index;
    ss << i;
    index = ss.str();
    shader.setVec2(("offsets[" + index + "]").c_str(), translations[i]);
}
// ...
```

## 实例化数组

但是 uniform 存在数量限制，因此当需要绘制的实例个数较多时，uniform 就没办法解决，因此可以使用实例化数组，涉及的函数如下

`void glVertexAttribDivisor(GLuint index, GLuint divisor)`

- index: 顶点属性的索引
- divisor: 决定 index 指定的顶点属性更新一次需要经过几次实例绘制

此时可以不需要设置 uniform 而是绑定顶点缓冲对象，并设置顶点属性指针

**顶点着色器中改为如下**

```glsl
#version 330 core
layout (location = 0) in vec2 aPos;
layout (location = 1) in vec3 aColor;
layout (location = 2) in vec2 aOffset;

out vec3 fColor;

void main()
{
    gl_Position = vec4(aPos + aOffset, 0.0, 1.0);
    fColor = aColor;
}
```

**程序代码改为如下**

```c++
// ...
// 生成偏移量数据
glm::vec2 translations[100];
int index = 0;
float offset = 0.1f;
for(int y = -10; y < 10; y += 2)
{
    for(int x = -10; x < 10; x += 2)
    {
        glm::vec2 translation;
        translation.x = (float)x / 10.0f + offset;
        translation.y = (float)y / 10.0f + offset;
        translations[index++] = translation;
    }
}

// ...
glGenVertexArrays(1, &vao);
glGenBuffers(1, &vbo);
glGenBuffers(1, &vbo2);
glBindVertexArray(vao);
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glBufferData(GL_ARRAY_BUFFER, sizeof(glm::vec2) * 100, translations,GL_STATIC_DRAW);
glEnableVertexAttribArray(2);
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 0, (void *)0);
glVertexAttribDivisor(2, 1);
// ...
```

## 实现小行星的示例程序效果

![图片无法显示](../image/opengl/instancing.png)

没有使用实例化的情况下，绘制 70000 个小行星较明显卡顿

![图片无法显示](../image/opengl/instancing2.png)

使用实例化的情况下，绘制 1400000 个小行星较明显卡顿，但还是比没有使用实例化绘制 70000 个时的情况要好

![图片无法显示](../image/opengl/instancing3.png)

> glBufferData 多次调用时后一次会释放前一次的内存，并创建新的内存，因此可以放心重复调用来改变缓冲对象内存区大小

# 抗锯齿

## 锯齿产生原因

由于屏幕像素实际上是一格格的固定宽高的方格，因此斜线映射到屏幕上时实际上无法将屏幕像素与斜线上的点准确地对应。

光栅器采用的办法简化一下很像小学教的估算三角形面积的方法（占方格超过一半的也计一个方格，少于一半的不计），过像素中心点的着色，不过的着色，这种做法就会如下图一样

![图片无法显示](../image/opengl/anti_aliasing_rasterization.png)
![图片无法显示](../image/opengl/anti_aliasing_rasterization_filled.png)

## 抗锯齿

为了解决上面的问题需要抗锯齿技术

### SSAA

超采样抗锯齿(Super Sample Anti-aliasing, SSAA）

先绘制分辨率更大的图像（例如：将上面的同样宽度的区域，划分为更多的网格，就能相对更准确表示三角形），再下采样(Downsample)（用线性插值来缩放什么的）到低分辨率

- 优点：简单易操作，效果最好
- 缺点：性能消耗大

### MSAA

多重采样抗锯齿（Multisample Anti-aliasing, MSAA）
这玩意没一个人能讲明白的，反而一堆脑瘫装逼犯，中英文切换不累吗？用一大堆乱七八糟的词，什么都说不清楚，就是将同一个模糊的说法换个花样重复，其间还夹杂着一堆鸟语单词，不好翻译的术语用鸟语也就罢了，简单的"能工作"，"样本"，"覆盖"，用 nm 的鸟语呢？看着就不爽。

暂时简单理解成：

1. 每个像素多个子样本。
2. 对于要绘制的如三角形之类的某个图形，其边缘不管包含某个像素的几个样本都会导致这个像素在光栅化阶段后作为片段进入片段着色器进行一次着色。
3. 说法 1：然后被三角形覆盖的子样本会取着色器输出的颜色；说法 2：该像素内所有样本都取着色器输出的颜色，但深度不同（如被三角形覆盖的子样本深度取三角形上顶点的深度，其余取最大深度）。
4. 对子样本单独深度测试。
5. 最终某像素颜色由所有子样本均值决定。
6. 也就是说 MSAA 相对于 SSAA 区别在于省了片段着色器对每个子样本单独着色的操作。

## glfw 启用多重采样抗锯齿

很简单，

- 设置多重采样样本数`glfwWindowHint(GLFW_SAMPLES, 4)`
- 启用多重采样`glEnable(GL_MULTISAMPLE)`

## 使用自己的多重采样缓冲

### 创建步骤

```c++
void AntiAliasing::createMultisampleFramebuffer()
{
    glGenFramebuffers(1, &fbo);
    glBindFramebuffer(GL_FRAMEBUFFER, fbo);

    // 创建多重采样纹理
    glGenTextures(1, &texture);
    glBindTexture(GL_TEXTURE_2D_MULTISAMPLE, texture);
    glTexImage2DMultisample(GL_TEXTURE_2D_MULTISAMPLE, 4, GL_RGB, WIDTH, HEIGHT, GL_TRUE);
    glBindTexture(GL_TEXTURE_2D_MULTISAMPLE, 0);
    // 绑定多重采样纹理为颜色附件
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D_MULTISAMPLE, texture, 0);
    // 创建多重采样渲染缓冲
    glGenRenderbuffers(1, &rbo);
    glBindRenderbuffer(GL_RENDERBUFFER, rbo);
    glRenderbufferStorageMultisample(GL_RENDERBUFFER, 4, GL_DEPTH24_STENCIL8, WIDTH, HEIGHT);
    glBindRenderbuffer(GL_RENDERBUFFER, 0);
    // 绑定渲染缓冲为深度和模板缓冲
    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_RENDERBUFFER, rbo);

    if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE)
    {
        std::cout << "framebuffer create error" << std::endl;
        return;
    }
    glBindFramebuffer(GL_FRAMEBUFFER, 0);
}
```

### 还原到默认帧缓冲

__涉及函数__

`void glBlitFramebuffer(GLint srcX0,GLint srcY0,GLint srcX1,GLint srcY1,GLint dstX0,GLint dstY0,GLint dstX1,GLint dstY1,GLbitfield mask,GLenum filter)`
+ src[x|y][0|1]: 源帧缓冲的左上和右下坐标
+ dst[x|y][0|1]: 目标帧缓冲的左上和右下坐标
+ mask:要被复制的缓冲，`GL_COLOR_BUFFER_BIT`, `GL_DEPTH_BUFFER_BIT` 和 `GL_STENCIL_BUFFER_BIT`.
+ filter: 插值方式，取`GL_NEAREST` 或 `GL_LINEAR`，含义参见纹理的插值方式

用`glBlitFramebuffer`将图像数据冲多重采样帧缓冲到默认的帧缓冲，并采用线性插值来缩放就实现了将多重采样帧缓冲的内容还原并绘制到默认帧缓冲的过程

### 抗锯齿+添加特效

+ 创建一个中间帧缓冲，和一个多级采样纹理缓冲
+ 图形绘制到多级采样纹理缓冲中，还原到中间帧缓冲中
+ 再将中间帧缓冲的绑定的颜色附件纹理直接绘制到一个全屏矩形上，这次绘制时添加特效。

### 自定义多重采样抗锯齿方式

可以直接将多重采样缓冲绑定的多级采样纹理绘制到默认帧缓冲上，此时做的修改如下

__c++代码中__

```c++
    // 绑定多级采样纹理
    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D_MULTISAMPLE, texture);
    square_shader.setUniform1i("texms", 0);

    // 绘制全屏矩形    
    glDrawArrays(GL_TRIANGLES, 0, 6);
    // 一定要解绑，否则似乎会出问题
    glBindTexture(GL_TEXTURE_2D_MULTISAMPLE, 0);
```

__片段着色器中__

注意`texelFetch`和`texture`坐标参数的不同之处

```glsl
    // 获取纹理尺寸
    ivec2 texSize = textureSize(texms);

    // 取均值
    FragColor = vec4(0.0f);
    FragColor += texelFetch(texms, ivec2(TexCoords * texSize), 0)/4;
    FragColor += texelFetch(texms, ivec2(TexCoords * texSize), 1)/4;
    FragColor += texelFetch(texms, ivec2(TexCoords * texSize), 2)/4;
    FragColor += texelFetch(texms, ivec2(TexCoords * texSize), 3)/4;
```

`ivec textureSize(gsampler sampler​, int lod​)`
+ 用来获取特定细节级别（lod，level-of-detail:指的是多级渐远纹理的级别下文同）的纹理尺寸
+ sampler:纹理对应的`sampler`变量，可取类型参考[Opengl Wiki:Sampler (GLSL)](https://www.khronos.org/opengl/wiki/Sampler_(GLSL))
+ lod:多级渐远纹理的级别，没有多级渐远纹理的sampler不可用
+ @return:返回向量的维数取决于sampler类型

`gvec texture(gsampler sampler​, vec texCoord​[, float bias​])`
+ 用标准化后（横纵坐标除以向量模）的纹理坐标从纹理中取样
+ sampler:纹理对应的`sampler`变量，可取类型参考[Opengl Wiki:Sampler (GLSL)](https://www.khronos.org/opengl/wiki/Sampler_(GLSL))
+ textCoord:标准化后的纹理坐标
+ bias:多级渐远纹理偏移量（LOD bias）,没有多级渐远纹理的sampler不可用
+ @return: 如果sampler的类型是Shadow samplers的类型，那么返回值是一个代表对比度（comparison）结果的浮点型。其余情况返回值是向量，维度取决于sampler类型。

`gvec texelFetch(gsampler sampler​, ivec texCoord​[, int lod​] [, int sample​])`
+ 用未标准化的坐标从纹理中取样
+ sampler: 纹理对应的`sampler`变量，可取类型参考[Opengl Wiki:Sampler (GLSL)](https://www.khronos.org/opengl/wiki/Sampler_(GLSL))
+ textCoord: 像素坐标，这个坐标是整形组成的二维向量，代表纹理图像某个像素的坐标，和`texture`的`texCoord`不同，`texture`的`texCoord`是标准化后的浮点型坐标
+ lod: 多级渐远纹理级别，对于没有多级渐远纹理的纹理类型来说，这个参数不存在
+ sampler: 第几个样本，对于不是多重采样的纹理，此参数不存在
+ @return: 返回值的向量维数取决于纹理的类型

### 最终效果

自定义抗锯齿的效果和抗锯齿是一样的

__无抗锯齿__

![图片无法显示](../image/opengl/msaa_off.png)

__抗锯齿__

![图片无法显示](../image/opengl/msaa_on.png)

__抗锯齿+模糊__

![图片无法显示](../image/opengl/msaa_blur.png)

__自定义抗锯齿__

![图片无法显示](../image/opengl/msaa_custom.png)

# 参考

1. [《GLES3.0 中文 API-glGetUniformIndices》](https://blog.csdn.net/msk1111/article/details/103796757) authored by [MSK1111](https://blog.csdn.net/MSK1111)@[csdn](https://www.csdn.net/)
2. [glVertexAttribDivisor](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glVertexAttribDivisor.xhtml)
3. [Opengl Wiki:Sampler (GLSL)](https://www.khronos.org/opengl/wiki/Sampler_(GLSL))
