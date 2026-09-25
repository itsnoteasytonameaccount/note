---
title: opengl(2):着色器和纹理
tags:
  - opengl
  - c++
date: 2020/12/28
---

为学懂webgl，从opengl入手。本文内容主要来自[LearnOpenGL CN](https://learnopengl-cn.github.io/)文档，加上个人理解，以及一些细节补充、经验记录。本人学习过程中的详细代码参见[此处](https://github.com/itsnoteasytonameaccount/OpenGL-learning)

要学习推荐看原文[LearnOpenGL CN](https://learnopengl-cn.github.io/)，有评论区，加载评论区需要翻墙

另可在[此处](https://www.khronos.org/opengl/wiki/Getting_Started)找到其他opengl官方推荐的资料、教程网站
<!-- more -->

# 着色器

GPU 中运行着色器程序的顺序如下(蓝色的为可由用户定义的)

![图片无法显示](../image/opengl/pipeline.png)

图形渲染管线的第一个部分是顶点着色器(Vertex Shader)，它把一个单独的顶点作为输入。顶点着色器主要的目的是把 3D 坐标转为另一种 3D 坐标（后面会解释），同时顶点着色器允许我们对顶点属性进行一些基本处理。

图元装配(Primitive Assembly)阶段将顶点着色器输出的所有顶点作为输入（如果是 GL_POINTS，那么就是一个顶点），并所有的点装配成指定图元的形状；本节例子中是一个三角形。

图元装配阶段的输出会传递给几何着色器(Geometry Shader)。几何着色器把图元形式的一系列顶点的集合作为输入，它可以通过产生新顶点构造出新的（或是其它的）图元来生成其他形状。例子中，它生成了另一个三角形。

几何着色器的输出会被传入光栅化阶段(Rasterization Stage)，这里它会把图元映射为最终屏幕上相应的像素，生成供片段着色器(Fragment Shader)使用的片段(Fragment)。在片段着色器运行之前会执行裁切(Clipping)。裁切会丢弃超出你的视图以外的所有像素，用来提升执行效率。

OpenGL 中的一个片段是 OpenGL 渲染一个像素所需的所有数据。

片段着色器的主要目的是计算一个像素的最终颜色，这也是所有 OpenGL 高级效果产生的地方。通常，片段着色器包含 3D 场景的数据（比如光照、阴影、光的颜色等等），这些数据可以被用来计算最终像素的颜色。

在所有对应颜色值确定以后，最终的对象将会被传到最后一个阶段，我们叫做 Alpha 测试和混合(Blending)阶段。这个阶段检测片段的对应的深度（和模板(Stencil)）值（后面会讲），用它们来判断这个像素是其它物体的前面还是后面，决定是否应该丢弃。这个阶段也会检查 alpha 值（alpha 值定义了一个物体的透明度）并对物体进行混合(Blend)。所以，即使在片段着色器中计算出来了一个像素输出的颜色，在渲染多个三角形的时候最后的像素颜色也可能完全不同。

## `in`、`out`、`uniform`修饰符

`in`和`out`在较早版本为`attribute`和`varying`

`in`声明的属性值会继承管线之前阶段用`out`声明的同名属性的值。

顶点着色器中的`in`的数据则是由程序决定

**uniform**

> 首先，uniform 是全局的(Global)，全局意味着 uniform 变量必须在每个着色器程序对象中都是独一无二的，而且它可以被着色器程序的任意着色器在任意阶段访问，然后，无论你把 uniform 值设置成什么，uniform 会一直保存它们的数据，直到它们被重置或更新

> 如果你声明了一个 uniform 却在 GLSL 代码中没用过，编译器会静默移除这个变量，导致最后编译出的版本中并不会包含它，这可能导致几个非常麻烦的错误

可以通过通过`glGetUniformLocation`查询`uniform`的值，通过`glUniform*`设置`uniform`的值

**glUniform 的后缀规则如下**

| 后缀 | 含义                                   |
| :--- | :------------------------------------- |
| f    | 函数需要一个 float 作为它的值          |
| i    | 函数需要一个 int 作为它的值            |
| ui   | 函数需要一个 unsigned int 作为它的值   |
| 3f   | 函数需要 3 个 float 作为它的值         |
| fv   | 函数需要一个 float 向量/数组作为它的值 |

具体模式和作用参见下文[函数](#函数)部分

## 数据类型

包含 c 的大部分基础类型`int`、`float`、`double`、`uint`和`bool`

### 向量

| 类型  | 含义                              |
| :---- | :-------------------------------- |
| vecn  | 包含 n 个 float 分量的默认向量    |
| bvecn | 包含 n 个 bool 分量的向量         |
| ivecn | 包含 n 个 int 分量的向量          |
| uvecn | 包含 n 个 unsigned int 分量的向量 |
| dvecn | 包含 n 个 double 分量的向量       |

> 一个向量的分量可以通过`vec.x`这种方式获取，这里`x`是指这个向量的第一个分量。你可以分别使用`.x`、`.y`、`.z`和`.w`来获取它们的第 1、2、3、4 个分量。GLSL 也允许你对颜色使用`rgba`，或是对纹理坐标使用`stpq`访问相同的分量。

**示例**

```glsl
vec2 someVec;
vec4 differentVec = someVec.xyxx;
vec3 anotherVec = differentVec.zyw;
vec4 otherVec = someVec.xxxx + anotherVec.yxzy;
```

## 预设变量

`gl_Position`：顶点着色器中设置此变量来决定顶点位置

## 内建类型

`sampler2D`：采样器，默认为`GL_TEXTURE0`的采样，可以设置，具体见[下文](#纹理叠加)描述

```glsl
// 声明一个采样器变量
uniform sampler2D ourTexture;
```

# 纹理

## glTexParameter\*

设置纹理参数

定义如下

```c++
typedef void (APIENTRYP PFNGLTEXPARAMETERFPROC)(GLenum target, GLenum pname, GLfloat param);
GLAPI PFNGLTEXPARAMETERFPROC glad_glTexParameterf;
#define glTexParameterf glad_glTexParameterf
typedef void (APIENTRYP PFNGLTEXPARAMETERFVPROC)(GLenum target, GLenum pname, const GLfloat *params);
GLAPI PFNGLTEXPARAMETERFVPROC glad_glTexParameterfv;
#define glTexParameterfv glad_glTexParameterfv
typedef void (APIENTRYP PFNGLTEXPARAMETERIPROC)(GLenum target, GLenum pname, GLint param);
GLAPI PFNGLTEXPARAMETERIPROC glad_glTexParameteri;
#define glTexParameteri glad_glTexParameteri
typedef void (APIENTRYP PFNGLTEXPARAMETERIVPROC)(GLenum target, GLenum pname, const GLint *params);
GLAPI PFNGLTEXPARAMETERIVPROC glad_glTexParameteriv;
#define glTexParameteriv glad_glTexParameteriv
```

以上函数命名后缀规则同`glUniform`

参数含义

- target： 指定目标纹理，它只能取值`GL_TEXTURE_1D`或者`GL_TEXTURE_2D`；
- pname：指定单值纹理参数的符号名称，可以选择的符号常数如下:`GL_TEXTURE_MIN_FILTER`、`GL_TEXTURE_MAG_FILTER`、`GL_TEXTURE_WRAP_S`、 `GL_TEXTURE_WRAP_T`, 后缀为`fv`和`iv`的的函数第还可传`GL_TEXTURE_BORDER_COLOR`和`GL_TEXTURE_PRIORITY`;
- param&#124;\*params：指定 pname 的值

## 纹理环绕方式

决定设置的纹理坐标在(0,0)到(1,1)区间外时如何绘制
通过`glTexParameteri`设置`GL_TEXTURE_WRAP_T`,`GL_TEXTURE_WRAP_S`的值来设置

| 环绕方式           | 描述                                                                                       |
| :----------------- | :----------------------------------------------------------------------------------------- |
| GL_REPEAT          | 对纹理的默认行为。重复纹理图，默认像。                                                     |
| GL_MIRRORED_REPEAT | 和 GL_REPEAT 一样，但每次重复图片是镜像放置的。                                            |
| GL_CLAMP_TO_EDGE   | 纹理坐标会被约束在 0 到 1 之间，超出的部分会重复纹理坐标的边缘，产生一种边缘被拉伸的效果。 |
| GL_CLAMP_TO_BORDER | 超出的坐标为用户指定的边缘颜色。                                                           |

**表现**

![图片无法显示](../image/opengl/texture_wrapping.png)

## 纹理过滤

决定将纹理放大或缩小时像素如何映射

通过`glTexParameteri`设置`GL_TEXTURE_MIN_FILTER`,`GL_TEXTURE_MAG_FILTER`的值来设置

对于`GL_TEXTURE_MIN_FILTER`，`GL_TEXTURE_MAG_FILTER`可设置

| 值           | 效果     | 举例                                              |
| :----------- | :------- | :------------------------------------------------ |
| `GL_NEAREST` | 就近取值 | 例如:将 a[10] 映射到 b[100] ,b[51]取 a[5]         |
| `GL_LINEAR`  | 线性插值 | 将 a[10]映射到 b[100],b[51]取 a[5]\*0.9+a[6]\*0.1 |

效果

![图片无法显示](../image/opengl/texture_filtering.png)

### 多级渐远纹理

配置`GL_TEXTURE_MIN_FILTER`时还可以取多级渐远纹理，多级渐远纹理是一组按 50%逐级等比缩小的纹理，如下

![图片无法显示](../image/opengl/mipmaps.png)

可在创建完纹理后使用`glGenerateMipmaps`生成

| 过滤方式                  | 描述                                                                     |
| :------------------------ | :----------------------------------------------------------------------- |
| GL_NEAREST_MIPMAP_NEAREST | 使用最邻近的多级渐远纹理来匹配像素大小，并使用邻近插值进行纹理采样       |
| GL_LINEAR_MIPMAP_NEAREST  | 使用最邻近的多级渐远纹理级别，并使用线性插值进行采样                     |
| GL_NEAREST_MIPMAP_LINEAR  | 在两个最匹配像素大小的多级渐远纹理之间进行线性插值，使用邻近插值进行采样 |
| GL_LINEAR_MIPMAP_LINEAR   | 在两个邻近的多级渐远纹理之间使用线性插值，并使用线性插值进行采样         |

**举例**

假设有一张原始尺寸为 128\*128 的图片，载入成了纹理并且设置了多级渐远纹理`glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_NEAREST);`，
然后用`glGenerateMipmaps`生成了尺寸分别为 128\*128、64\*64、32\*32、16\*16、8\*8、4\*4、2\*2、1\*1的多级纹理。
那么将纹理绘制到某个 60\*60 的表面上时，会采用64\*64的纹理，对表面某点的颜色取值时，会采用线性线性插值的方式。

## 纹理使用步骤

### 创建纹理并设置参数

```c++
unsigned int texture;
glGenTextures(id, texture);
// 默认活跃的是GL_TEXTURE0
// glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, texture);

glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST_MINMAP_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINREA);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_MIRRORED_REPEAT);
```

### 纹理中填入图片数据并生成多级纹理渐变

读取图片数据使用的是 [stb_image.h](https://github.com/nothings/stb/blob/master/stb_image.h)

```c++
glTexImage2D(GL_TEXTURE_2D, 0, GL_RBG, width, height, 0, GL_RBG, GL_UNSIGNED_INT, data);
glGenerateMinmap(GL_TEXTURE_2D);
```

处理完后修改片段着色器中的代码如下即可

```glsl
#version 330 core
out vec4 FragColor;

in vec3 ourColor;
in vec2 TexCoord;

uniform sampler2D ourTexture;

void main()
{
    FragColor = texture(ourTexture, TexCoord);
    // 颜色叠加
    FragColor = texture(ourTexture, TexCoord) * vec4(ourColor, 1.0);
}
```

## 纹理叠加

通过修改当前活跃的纹理 GL_TEXTURE(0-15+)，可以同时绑定多个纹理，如下

```c++
GL_TEXTURE0 + n == GL_TEXTUREn; // true
```

```c++
// 假设已创建两个纹理texture和texture1
// texture绑定到GL_TEXTURE0
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, texture);
// texture1绑定到GL_TEXTURE1
glActiveTexture(GL_TEXTURE1);
glBindTexture(GL_TEXTURE_2D, texture1);
```

然后可以通过修改片段着色器来叠加纹理。

```glsl
#version 330 core
...

uniform sampler2D texture1;
uniform sampler2D texture2;

void main()
{
    FragColor = mix(texture(texture1, TexCoord), texture(texture2, TexCoord), 0.2);
}
```

还需要指定两个采样器属于哪个纹理单元

```c++
// 只需要在程序编译成功，纹理绑定后指定1次，不需要在渲染循环中指定
glUniform1i(glGetUniformLocation(ourShader.ID, "texture1"), 0);
glUniform1i(glGetUniformLocation(ourShader.ID, "texture2"), 1);
```

> 最终输出颜色现在是两个纹理的结合。GLSL 内建的 mix 函数需要接受两个值作为参数，并对它们根据第三个参数进行线性插值。如果第三个值是 0.0，它会返回第一个输入；如果是 1.0，会返回第二个输入值。0.2 会返回 80%的第一个输入颜色和 20%的第二个输入颜色，即返回两个纹理的混合色。

# 函數

## glUniform{1|2|3|4}{i|f}

向 uniform 中填入{1|2|3|4}个{浮点型|整型}数据

例如`glUniform3f`

**定义**

`typedef void (APIENTRYP PFNGLUNIFORM3FPROC)(GLint location, GLfloat v0, GLfloat v1, GLfloat v2);`

**参数列表**

- location: uniform 在程序中的位置，可以用`glGetUniformLocation`获取
- v0: 第一个浮点数
- v1: 第二个浮点数
- v2: 第三个浮点数

## glUniform{1|2|3|4}{i|f}v

修改一个 uniform 或一个 uniform 数组的值，向每个 uniform 中填入长度为{1|2|3|4}的{浮点型|整型}数据

**参数列表**

- location: uniform 在程序中的位置，可以用`glGetUniformLocation`获取
- count: 要修改几个 uniform
- values: 数据数组的首地址指针

## glUniformMatrix{2|3|4}fv

修改一个或多个矩阵类型的 uniform 变量

`typedef void (APIENTRYP PFNGLUNIFORMMATRIX4FVPROC)(GLint location, GLsizei count, GLboolean transpose, const GLfloat *value)`

参数列表

- location: uniform 变量在着色器程序中的位置
- count: uniform 数量
- transpose:是否转置
- value:矩阵数据数组首地址指针

## glTexImage2D

`void glTexImage2D(GLenum target, GLint level, GLint internalformat, GLsizei width, GLint border, GLenum format, GLenum type, const void *pixels)`

+ target: 目标纹理
+ level: 用来指定多级渐远纹理的级别，0为基本级别即原纹理。如果想手动设置每级多级渐远纹理的数据，可以指定这个参数。
  > Mipmaps are defined using `glTexImage1D` or `glTexImage2D` with the **level-of-detail** argument indicating the order of the mipmaps. **Level** 0 is the original texture; **level** bold max(n, m) is the final 1x1 mipmap.
+ internalformat: 纹理格式，如`GL_RGB`
+ width,height: 宽高
+ border: 边长必须为0或1
+ format: 据说必须和inernalformat一致
+ type: 图像数据的数据类型`GL_UNSIGNED_BYTE, GL_BYTE, GL_BITMAP, GL_UNSIGNED_SHORT, GL_SHORT, GL_UNSIGNED_INT, GL_INT, and GL_FLOAT`
+ pixels: 图像数据

# 参考

1. [learnopengl CN:着色器](https://learnopengl-cn.github.io/01%20Getting%20started/04%20Hello%20Triangle/)
2. [learnopengl CN:纹理](https://learnopengl-cn.github.io/01%20Getting%20started/06%20Textures/)
3. [glTexParameter Functions](https://docs.microsoft.com/en-us/windows/win32/opengl/gltexparameter-functions)