---
title: opengl(7):深度测试、模板测试、混合、面剔除、帧缓冲、立方体贴图
tags:
  - opengl
date: 2021/02/12
---

为学懂webgl，从opengl入手。本文内容主要来自[LearnOpenGL CN](https://learnopengl-cn.github.io/)文档，加上个人理解，以及一些细节补充、经验记录。本人学习过程中的详细代码参见[此处](https://github.com/itsnoteasytonameaccount/OpenGL-learning)

要学习推荐看原文[LearnOpenGL CN](https://learnopengl-cn.github.io/)，有评论区，加载评论区需要翻墙

另可在[此处](https://www.khronos.org/opengl/wiki/Getting_Started)找到其他opengl官方推荐的资料、教程网站

<!-- more -->

# 深度测试

> 当深度测试(Depth Testing)被启用的时候，OpenGL 会将一个片段的深度值与深度缓冲的内容进行对比。OpenGL 会执行一个深度测试，如果这个测试通过了的话，深度缓冲将会更新为新的深度值。如果深度测试失败了，片段将会被丢弃。

因为只需要显示挡在最前面的片段，所以深度缓冲做的就是保留在最前面片段的深度，和常见的取最小值的方法相同，如下。

```c++
if(depth[x][y] > frag.frag_depth)
{
	depth[x][y] = frag.frag_depth;
}
else
{
	delete frag;
}
```

深度测试在片段着色器之后在屏幕空间运行，可以在片段着色器中通过 gl_FragCoord 访问。gl_FragCoord.z 就是需要和深度缓冲比较的值。

> 现在大部分的 GPU 都提供一个叫做提前深度测试(Early Depth Testing)的硬件特性。提前深度测试允许深度测试在片段着色器之前运行。只要我们清楚一个片段永远不会是可见的（它在其他物体之后），我们就能提前丢弃这个片段。片段着色器通常开销都是很大的，所以我们应该尽可能避免运行它们。当使用提前深度测试时，片段着色器的一个限制是你不能写入片段的深度值。如果一个片段着色器对它的深度值进行了写入，提前深度测试是不可能的。OpenGL 不能提前知道深度值。

## 相关函数

- `void glClearDepth(GLdouble depth)`指定用于清除深度缓冲的值。
- `void glEnable(GLenum cap)`启用特性，`glEnable(GL_DEPTH_TEST)`启用深度测试
- `void glClear(GLenum cap)`清除，`glClear(GL_DEPTH_BUFFER_BIT)`用`glClearDepth`指定的值清除深度缓冲
- `void glDepthFunc(GLenum func)`指定深度缓冲的比较方式，可取值如下
- `void glDepthMask(GLboolean flag)`启用深度测试时禁用或启用写入

| 函数        | 描述                                              |
| :---------- | :------------------------------------------------ |
| GL_ALWAYS   | 永远通过深度测试                                  |
| GL_NEVER    | 永远不通过深度测试                                |
| GL_LESS     | 在片段深度值小于缓冲的深度值时通过测试 ，默认情况 |
| GL_EQUAL    | 在片段深度值等于缓冲区的深度值时通过测试          |
| GL_LEQUAL   | 在片段深度值小于等于缓冲区的深度值时通过测试      |
| GL_GREATER  | 在片段深度值大于缓冲区的深度值时通过测试          |
| GL_NOTEQUAL | 在片段深度值不等于缓冲区的深度值时通过测试        |
| GL_GEQUAL   | 在片段深度值大于等于缓冲区的深度值时通过测试      |

## 深度精确度

### 线性关系

深度缓冲中的深度坐标在 0.0 到 0.1 之间，需要定义一种将观察坐标转化到深度取值范围的方法。可取线性转化方式如下：

$$
F_{depth} = \frac{z - near}{far - near}
$$

关系图像如下

![图片无法显示](../image/opengl/depth_linear_graph.png)

但是这种关系几乎不会被使用，原因是不满足"使近处的物体深度精确度高，远处的物体深度精确度低"这一需要（远处的物体相对近处不用太精细）。

<div style="padding: 20px; background-color: #00000055">

另外我认为还有一个原因是使用线性关系的话，深度只能在片段着色器中计算，没办法通过投影矩阵直接转化。理由如下：

在之前的 opengl(3)中提到了如何通过观察坐标$(x_e, y_e, z_e, w_e)$得到标准设备坐标$(x_n, y_n, z_n)$。其中需要先用投影矩阵将观察坐标转化为裁剪坐标$(x_c, y_c, z_c, w_c)$

按照之前的计算方法
如果$F_{depth} = \frac{z_e - (-n)}{(-f) - (-n)}$（左手坐标），又有$F_{depth} = (z_n + 1)/2$，那么此时

$$
z_n = \frac{2z_e + n + f}{n - f}
$$

又因$z_n = z_c/w_c$且之前求到$w_c$最好取$-z_e$，那么就有

$$
z_c = z_n * w_c = \frac{2z_e^2 + (n + f)z_e}{f - n}
$$

又

$$
\begin{pmatrix}
x_c\\y_c\\z_c\\w_c
\end{pmatrix} = {\large M_{proj}} *
\begin{pmatrix}
x_e\\y_e\\z_e\\w_e
\end{pmatrix}
$$

要怎样设置投影矩阵才能使$z_c = \frac{2z_e^2 + (n + f)z_e}{f - n}$呢？应该没有办法吧，反正我想不到。

</div>

所以得采用非线性的转化方式如下：

$$
F_{depth} = \frac{1/z - 1/near}{1/far - 1/near}
$$

图像如下

![图片无法显示](../image/opengl/depth_non_linear_graph.png)

这个公式取$near = -n, far = -f$进行一定运算，就能得到前面 opengl(3)中的结果

$$
\begin{aligned}
z_n &= 2F_{depth} - 1 \\
&= \frac{2fn}{(f - n)z} + \frac{2f}{f - n} - 1\\
&= \frac{n+f}{f-n} + \frac{2fn}{(f-n)z}
\end{aligned}
$$

### 在片段着色器中计算线性关系

计算线性关系需要先得到$z_e$，（需要注意这个$z_e \in [-n, -f]$）

$$
z_e = 2fn/(z_n * (f - n) - n - f)
$$

然后

$$
\begin{aligned}
F_{depth} &= \frac{z_e - (-n)}{(-f) - (-n)}\\
&= \frac{-z_e -n}{f - n}\\
&=  \frac{2fn/(n + f - z_n * (f - n)) - n}{f - n}
\end{aligned}
$$

片段着色器代码

```glsl
#version 330 core
out vec4 FragColor;

float near = 0.1;
float far  = 100.0;

float LinearizeDepth(float depth)
{
    float z = depth * 2.0 - 1.0; // back to NDC
    return (2.0 * near * far) / (far + near - z * (far - near));
}

void main()
{
    float depth = (LinearizeDepth(gl_FragCoord.z) - near)/ (far - near);
    FragColor = vec4(vec3(depth), 1.0);
}
```

## 深度冲突

按我的理解，举个简单例子，假设深度精确到 0.01，将 1000 映射到$[0.00, 1.00]$区间内，其中 1 映射为$0.001$，2 映射为$0.002$，但由于精确度的关系，两者都被认为是$0.00$，无法判断哪个大。

导致的结果如下

![图片无法显示](../image/opengl/z_fight.png)

> 上面这个情况是不是还可以得到执行深度测试的片段顺序是随机的？不然的话，如果按照物体调用`glDrawArrays`的顺序进行深度测试，且判断方式是单纯的`depth = gl_FragCoord.z < depth ? gl_FragCoord.z : depth`，即使`gl_FragCoord.z`的值相同，后调用`glDrawArrays`的物体的片段就会被舍去了。（或者说不是这么简单的机制？）

### 解决深度冲突

1. 永远不要把物体靠得太近
2. 将近平面到远平面间的跨度设置得小一些
3. 使用更高精度的缓冲
4. 近平面设置得远一些

# 模板测试

- 模板测试，同样是用来决定是丢弃还是保留片段的
- 模板测试在片段着色器之后执行
- 模板测试根据模板缓冲进行，模板缓冲中每个模板值 8 位.所以每个像素/片段一共能有 256 种不同的模板值。我们可以将这些模板值设置为我们想要的值，然后当某一个片段有某一个模板值的时候，我们就可以选择丢弃或是保留这个片段了。
- glfw 自动创建了一个模板缓冲，其他库有可能有差别

模板缓冲的一个简单的例子如下：

![图片无法显示](../image/opengl/stencil_buffer.png)

模板缓冲操作允许我们在渲染片段时将模板缓冲设定为一个特定的值。通过在渲染时修改模板缓冲的内容，我们写入了模板缓冲。在同一个（或者接下来的）渲染迭代中，我们可以读取这些值，来决定丢弃还是保留某个片段。使用模板缓冲的时候你可以尽情发挥，但大体的步骤如下：

- 启用模板缓冲的写入。
- 渲染物体，更新模板缓冲的内容。
- 禁用模板缓冲的写入。
- 渲染（其它）物体，这次根据模板缓冲的内容丢弃特定的片段。

## 相关函数

`void glStencilFunc(GLenum func, GLint ref, GLuint mask)`

- func 设置模板测试判断方式，可用的选项有：GL_NEVER、GL_LESS、GL_LEQUAL、GL_GREATER、GL_GEQUAL、GL_EQUAL、GL_NOTEQUAL 和 GL_ALWAYS。它们的语义和深度缓冲的函数类似。
- ref 参考值，模板值会和这个值比较
- mask 在参考值与模板值比较前，与参考值进行与运算

`void glStencilMask(GLuint mask)`

- mask:掩码，模板测试通过，写入模板缓冲前与要写入的值进行与运算，注意设为 0x00 时最终不是写入 0 而是禁止写入，设为 0xff 按原样写入，会影响 glClear，设为 0x00 时 glClear 无法清除

`void glStencilOp(GLenum fail, GLenum zfail, GLenum zpass)`

- fail 模板测试失败时的行为
- zfail 模板测试通过，深度测试失败时的行为
- zpass 模板测试和深度测试都通过时的行为
- 三个参数都可取如下值，默认情况是`glStencilOp(GL_KEEP, GL_KEEP, GL_KEEP)`

| 行为         | 描述                                                   |
| :----------- | :----------------------------------------------------- |
| GL_KEEP      | 保持当前储存的模板值                                   |
| GL_ZERO      | 将模板值设置为 0                                       |
| GL_REPLACE   | 将模板值设置为 glStencilFunc 函数设置的 ref 值         |
| GL_INCR      | 如果模板值小于最大值则将模板值加 1                     |
| GL_INCR_WRAP | 与 GL_INCR 一样，但如果模板值超过了最大值则归零        |
| GL_DECR      | 如果模板值大于最小值则将模板值减 1                     |
| GL_DECR_WRAP | 与 GL_DECR 一样，但如果模板值小于 0 则将其设置为最大值 |
| GL_INVERT    | 按位翻转当前的模板缓冲值                               |

另外同样需要启用以及在每次绘制前清除模板缓冲
`glEnable(GL_STENCIL_TEST)` `glClearStencil(0x00)` `glClear(GL_STENCIL_BUFFER_BIT)`

## 物体轮廓

代码如下

```c++

void DepthTest::draw()
{
    glStencilOp(GL_KEEP, GL_KEEP, GL_REPLACE);
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT);

    glm::mat4 model(1.0f);
    glm::mat4 view = camera.getViewMatrix();

    shader.useProgram();
    // 为了方便，用一个uniform来决定绘制颜色还是纹理
    shader.setUniform1i("is_texture", 1);
    // glDepthFunc(GL_ALWAYS);
    glStencilFunc(GL_ALWAYS, 1, 0xff);
    // 地板绘制时不能影响模板缓冲
    glStencilMask(0x00);
    glActiveTexture(GL_TEXTURE1);
    glBindTexture(GL_TEXTURE_2D, texture2);
    shader.setUniform1i("tex", 1);
    shader.setUniformMatrix4("model", glm::mat4(1.0f));
    shader.setUniformMatrix4("view", view);
    shader.setUniformMatrix4("projection", projection);
    glBindVertexArray(vao);
    // 地板
    glDrawArrays(GL_TRIANGLE_STRIP, 0, 6);

    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D, texture1);
    shader.setUniform1i("tex", 0);

    // 方块绘制时写入模板缓冲
    glStencilMask(0xff);
    // glDepthFunc(GL_LESS);

    model = glm::translate(model, glm::vec3(-1.0f, 0.0f, -1.0f));
    model = glm::scale(model, glm::vec3(0.9f, 0.9f, 0.9f));
    shader.setUniformMatrix4("model", model);
    box.bind();
    box.draw();

    model = glm::translate(glm::mat4(1.0f), glm::vec3(2.0f, 0.0f, 0.0f));
    model = glm::scale(model, glm::vec3(0.9f, 0.9f, 0.9f));
    shader.setUniformMatrix4("model", model);
    box.draw();

    shader.setUniform1i("is_texture", 0);
    glStencilFunc(GL_NOTEQUAL, 1, 0xff);
    glStencilMask(0x00);

    model = glm::translate(glm::mat4(1.0f), glm::vec3(-1.0f, 0.0f, -1.0f));
    shader.setUniformMatrix4("model", model);
    box.draw();
    model = glm::translate(glm::mat4(1.0f), glm::vec3(2.0f, 0.0f, 0.0f));
    shader.setUniformMatrix4("model", model);
    box.draw();
    glStencilMask(0xff);
}
```

# 混合

opengl 这点还是有点别扭，并不会自动显示透明物体后面的物体。需要启用混合。

## 透明物体

绘制带透明部分的纹理有时候可以在片段着色器中用`discard`丢弃。

```glsl
if(FragColor.a < 0.1)
{
    discard;
}
```

## 半透明物体

但是绘制半透明物体时没法丢弃，需要用`glEnable(GL_BLEND)`启用混合

混合计算方式

$$
\vec{C}_{result} = \vec{C}_{source} ∗ F_{source} + \vec{C}_{destination} ∗ F_{destination}
$$

启用混合后，当一个带透明度的片段在另一个片段前面（更加靠近摄像机的位置）绘制时，只要先绘制了后面的片段，这个后面的片段就不会因为深度测试不通过而被丢弃，而是根据上面的公式，混合产生新的颜色$\vec{C}_{result}$。此时在后面的片段的颜色向量就是目标颜色向量$\vec{C}_{destination}$，前面的带透明度的片段的颜色向量就是源颜色向量$\vec{C}_{source}$，这两个值由 opengl 自动决定，源因子和目标因子$F_{source},F_{destination}$的值则可由我们决定。

## 相关函数

`void glBlendFuncSeparate(GLenum sfactorRGB, GLenum dfactorRGB, GLenum sfactorAlpha, GLenum dfactorAlpha);`

- 用于分别设置 RGB 的因子值，和透明度的因子值
- sfactorRGB:RGB 源因子值
- dfactorRGB:RGB 目标因子值
- sfactorAlpha:透明度源因子值
- dfactorAlpha:透明度目标因子值

`void glBlendFunc(GLenum sfactor, GLenum dfactor)`

- sfactorRGB:RGB 源因子值
- dfactorRGB:RGB 目标因子值

因子值的可选项

| 参数                        | 描述                                    |
| :-------------------------- | :-------------------------------------- |
| GL_ZERO                     | 因子等于 0                              |
| GL_ONE                      | 因子等于 1                              |
| GL_SRC_COLOR                | 因子等于源颜色向量 C¯source             |
| GL_ONE_MINUS_SRC_COLOR      | 因子等于 1−C¯source                     |
| GL_DST_COLOR                | 因子等于目标颜色向量 C¯destination      |
| GL_ONE_MINUS_DST_COLOR      | 因子等于 1−C¯destination                |
| GL_SRC_ALPHA                | 因子等于 C¯source 的 alpha 分量         |
| GL_ONE_MINUS_SRC_ALPHA      | 因子等于 1− C¯source 的 alpha 分量      |
| GL_DST_ALPHA                | 因子等于 C¯destination 的 alpha 分量    |
| GL_ONE_MINUS_DST_ALPHA      | 因子等于 1− C¯destination 的 alpha 分量 |
| GL_CONSTANT_COLOR           | 因子等于常数颜色向量 C¯constant         |
| GL_ONE_MINUS_CONSTANT_COLOR | 因子等于 1−C¯constant                   |
| GL_CONSTANT_ALPHA           | 因子等于 C¯constant 的 alpha 分量       |
| GL_ONE_MINUS_CONSTANT_ALPHA | 因子等于 1− C¯constant 的 alpha 分量    |

## 用法

如下若想将透明的绿色绘制到红色前面，启用混合后，调用`glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)`设置了两个因子值

![图片无法显示](../image/opengl/blending_equation.png)

最终得到的结果就是

$$
\vec{C}_{result} = \begin{pmatrix}
1.0\\0.0\\0.0\\1.0
\end{pmatrix} * (1.0 - 0.6) + \begin{pmatrix}
0.0\\1.0\\0.0\\0.6
\end{pmatrix} * 0.6
$$

## 排序

上面也提到启用混合和深度测试后，想要得到理想的效果，需要先绘制后面的物体，才能绘制前面带透明度的物体，否则后面的物体的片段会被丢弃。

因此需要按照理摄像机的距离由远到近绘制物体。

接下来我们把距离和它对应的位置向量存储到一个 STL 库的 map 数据结构中。map 会自动根据键值(Key)对它的值排序，所以只要我们添加了所有的位置，并以它的距离作为键，它们就会自动根据距离值排序了。

```c++
    map<float, glm::vec3> sorted;
    float distance;
    for (int i = 0; i < vegetation.size(); i++)
    {
        distance = glm::length(camera.getPosition() - vegetation[i]);
        sorted[distance] = vegetation[i];
    }
    for(map<float, glm::vec3>::reverse_iterator it = sorted.rbegin(); it != sorted.rend(); ++it)
    {
        model = glm::translate(glm::mat4(1.0f), it->second);
        shader.setUniformMatrix4("model", model);
        glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
    }
```

另外似乎还可以如下，这样可以不排序

```c++
    // 即使深度测试通过也不写入深度缓冲
    glDepthMask(GL_FALSE);
    map<float, glm::vec3> sorted;
    float distance;
    for (int i = 0; i < vegetation.size(); i++)
    {
        model = glm::translate(glm::mat4(1.0f), vegetation[i - 1]);
        shader.setUniformMatrix4("model", model);
        glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
    }
    glDepthMask(GL_TRUE);
```

最后效果

![图片无法显示](../image/opengl/blend.png)

# 面剔除

剔除看不到的片段可以大量节省片段着色器的调用。opengl 通过三角形顶点的环绕顺序决定哪些是正向面（面向观察者的面），哪些是负向面（观察者看不到的面）。

## 环绕顺序

按特定顺序定义每个面点的顺序，当这些面面向观察者时，会沿一个方向环绕，背向观察者时则沿另一个方向环绕，如下

![图片无法显示](../image/opengl/faceculling_frontback.png)

> 我记得 3dmax 中插入的平面、图片就是这样，正面看正常，但转到背面就看不到，不知道是不是面剔除起的作用。

## 相关函数

用`glEnable(GL_CULL_FACE)`启用

`void glCullFace(GLenum mode)`
决定要剔除的面的类型，可选项

- GL_FRONT，正面
- GL_BACK，背面，默认值
- GL_FRONT_AND_BACK，正面和背面

`void glFrontFace(GLenum mode)`
设置顶点按什么方向环绕才是正面，可选项

- GL_CCW，逆时针，默认值
- GL_CW，顺时针

## 效果

首先需要调整三角形点的顺逆时针方向，全部调整为正面逆时针
然后启用面剔除效果如下

剔除正面

![图片无法显示](../image/opengl/cull_front.png)

剔除背面，看起来很正常

![图片无法显示](../image/opengl/cull_face.png)

剔除正面，从远处看

![图片无法显示](../image/opengl/cull_ccw.png)

# 帧缓冲

之前绘制到的都是默认的帧缓冲，是创建窗口时 glfw 替我们生成的，绘制到默认缓冲会显示在窗口中。

帧缓冲创建也是老三样

- 生成`glGenFramebuffers(GLsizei n, GLuint *framebuffers);`
- 绑定`glBindFramebuffer(GLenum target, GLuint framebuffer);`

然后需要进行一些设置，一个完整的帧缓冲需要满足如下条件

- 附加至少一个缓冲（颜色、深度或模板缓冲）。
- 至少有一个颜色附件(Attachment)。
- 所有的附件都必须是完整的（保留了内存）。
- 每个缓冲都应该有相同的样本数。

## 纹理附件

创建空纹理用来存放 opengl 绘制的图像数据

```c++
    unsigned int texture;
    glActiveTexture(GL_TEXTURE0);
    glGenTextures(id, &texture);
    glBindTexture(GL_TEXTURE_2D, texture);
    // 设置数据为null，等于申请了一片特定格式内存
    glTexImage2D(GL_TEXTURE_2D, 0, internalformat, width, height, 0, format, type, NULL);
    // 不需要生成多级渐远纹理
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

    // 绑定纹理为颜色附件
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, texture, 0);
    // 纹理还可以绑定为深度、模板缓冲
    // 此时创建的参数需要修改，生成24位深度8位模板每个样本共32位的纹理
    // glTexImage2D(GL_TEXTURE_2D, 0, GL_DEPTH24_STENCIL8, width, height, 0,GL_DEPTH_STENCIL, GL_UNSIGNED_INT_24_8, NULL);
    // 绑定为深度、模板缓冲
    // glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_TEXTURE_2D, texture, 0);
```

`void glFrameBufferTexture2D(GLenum target, GLenum attachment, GLenum textarget, GLuint texture, GLint level)`

- target:目标帧缓冲
- attachment:附件类型，颜色附件可以有多个，`glad.h`文件中定义了`GL_COLOR_ATTACHMENT[0~31]`共 32 个，其他可取`GL_DEPTH_STENCIL_ATTACHMENT,GL_STENCIL_ATTACHMENT,GL_DEPTH_ATTACHMENT`
- textarget:纹理类型，可取`GL_TEXTURE_2D, GL_TEXTURE_CUBE_MAP_POSITIVE_X, GL_TEXTURE_CUBE_MAP_NEGATIVE_X, GL_TEXTURE_CUBE_MAP_POSITIVE_Y, GL_TEXTURE_CUBE_MAP_NEGATIVE_Y, GL_TEXTURE_CUBE_MAP_POSITIVE_Z GL_TEXTURE_CUBE_MAP_NEGATIVE_Z`
- texture:纹理
- level:多级渐远纹理级别

## 渲染缓冲对象

> 渲染缓冲对象(Renderbuffer Object)是在纹理之后引入到 OpenGL 中，作为一个可用的帧缓冲附件类型的，所以在过去纹理是唯一可用的附件。和纹理图像一样，渲染缓冲对象是一个真正的缓冲，即一系列的字节、整数、像素等。渲染缓冲对象附加的好处是，它会将数据储存为 OpenGL 原生的渲染格式，它是为离屏渲染到帧缓冲优化过的。

> 渲染缓冲对象直接将所有的渲染数据储存到它的缓冲中，不会做任何针对纹理格式的转换，让它变为一个更快的可写储存介质。然而，渲染缓冲对象通常都是只写的，所以你不能读取它们（比如使用纹理访问）。当然你仍然还是能够使用 glReadPixels 来读取它，这会从当前绑定的帧缓冲，而不是附件本身，中返回特定区域的像素。

生成`glGenRenderbuffers(id, &rbo)`，绑定`glBindRenderbuffer(GL_RENDERBUFFER, rbo)`

设置

```c++
    // 初始化内存
    glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH24_STENCIL8, 800, 600);
    // 绑定到帧缓冲
    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_RENDERBUFFER, rbo);
```

`void glRenderbufferStorage(GLenum target, GLenum internalformat, GLsizei width, GLsizei height);`
+ 创建目标渲染缓冲的内存
+ target: 目标渲染缓冲
+ internalformat: 格式
+ width,height: 宽高

`void glFramebufferRenderbuffer(GLenum target, GLenum attachment, GLenum renderbuffertarget, GLuint renderbuffer)`

+ 绑定目标渲染缓冲到目标帧缓冲
+ target: 目标帧缓冲
+ attachment: 附件类型，`GL_DEPTH_STENCIL_ATTACHMENT,GL_STENCIL_ATTACHMENT,GL_DEPTH_ATTACHMENT`
+ renderbuffertarget: 目标渲染缓冲
+ renderbuffer: 绚烂缓冲

## 用处

用自定义的帧缓冲的好处是，可以先绘制一次，然后把得到的图像数据处理后再绘制到默认的帧缓冲中，因此可以添加一些后期特效

第二次绘制只需要把自定义的帧缓冲绑定的纹理绘制一遍即可，因此第二遍绘制的着色器非常简单

**顶点着色器**

```glsl
#version 330 core

layout (location = 0) in vec3 aPos;
layout (location = 1) in vec2 aTexCoords;

out vec2 TexCoords;

void main()
{
    gl_Position = vec4(aPos, 1.0f);
    TexCoords = aTexCoords;
}
```

**片段着色器**

```glsl
#version 330 core
in vec2 TexCoords;

uniform sampler2D tex;

out vec4 FragColor;

void main()
{
    FragColor = texture(tex, TexCoords);
}
```

并且可以在片段给绘制添加后期效果，添加效果的片段着色器如下

```glsl
#version 330 core
in vec2 TexCoords;

// 用来控制效果切换
uniform int effect;
uniform sampler2D tex;

out vec4 FragColor;

#define INVERSION 0
#define GRAYSCALE 1
#define BLUR 2
#define SHARPEN 3
#define EDGE_DETECTION 4

// 偏移量，纹理大小是800*600，不知道小于1/600会怎么样
const float offset = 1.0f / 300.0f;

vec2[] offsets = vec2[](
    vec2(-offset, offset),
    vec2(-offset, 0),
    vec2(-offset, -offset),
    vec2(offset, offset),
    vec2(offset, 0),
    vec2(offset, -offset),
    vec2(0, offset),
    vec2(0, 0),
    vec2(0, -offset)
);

// 负片
vec4 Inversion()
{
    return vec4(1.0f - texture(tex, TexCoords).xyz, 1.0f);
}

// 灰度
vec4 Grayscale()
{
    vec4 result = texture(tex, TexCoords);
    float average = 0.2126 * result.r + 0.7152 * result.g + 0.0722 * result.b;
    return vec4(average, average, average, 1.0f);
}

// 核效果
vec4 KernelEffect(float kernel[9])
{
    vec3 col = vec3(0.0f);
    for(int i = 0; i < 9; i++)
    {
        col += kernel[i] * texture(tex, TexCoords + offsets[i]).xyz;
    }
    return vec4(col, 1.0f);
}

void main()
{
    switch(effect)
    {
    case INVERSION:
        FragColor = Inversion();
        break;
    case GRAYSCALE:
        FragColor = Grayscale();
        break;
    case BLUR:
        FragColor = KernelEffect(float[](
            1.0 / 16, 2.0 / 16, 1.0 / 16,
            2.0 / 16, 4.0 / 16, 2.0 / 16,
            1.0 / 16, 2.0 / 16, 1.0 / 16
        ));
        break;
    case SHARPEN:
        FragColor = KernelEffect(float[](
            -1, -1, -1,
            -1,  9, -1,
            -1, -1, -1
        ));
        break;
    case EDGE_DETECTION:
        FragColor = KernelEffect(float[](
             1,  1,  1,
             1, -8,  1,
             1,  1,  1
        ));
        break;
    default:
        FragColor = texture(tex, TexCoords);
        break;
    }
}

```

绘制代码

```c++

void Framebuffers::draw()
{
    // 先绘制到自定义帧缓冲
    glBindFramebuffer(GL_FRAMEBUFFER, fbo);
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    glm::mat4 model(1.0f);
    glm::mat4 view = camera.getViewMatrix();

    shader.useProgram();
    shader.setUniform1i("is_texture", 1);
    // glDepthFunc(GL_ALWAYS);
    glActiveTexture(GL_TEXTURE1);
    glBindTexture(GL_TEXTURE_2D, texture2);
    shader.setUniform1i("tex", 1);
    shader.setUniformMatrix4("model", glm::mat4(1.0f));
    shader.setUniformMatrix4("view", view);
    shader.setUniformMatrix4("projection", projection);
    glBindVertexArray(vao);
    glDrawArrays(GL_TRIANGLE_STRIP, 0, 6);

    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D, texture1);
    shader.setUniform1i("tex", 0);

    model = glm::translate(model, glm::vec3(-1.0f, 0.0f, -1.0f));
    shader.setUniformMatrix4("model", model);
    box.bind();
    box.draw();

    model = glm::translate(glm::mat4(1.0f), glm::vec3(2.0f, 0.0f, 0.0f));
    shader.setUniformMatrix4("model", model);
    box.draw();

    // 改用默认帧缓冲
    glBindFramebuffer(GL_FRAMEBUFFER, 0);

    framebuffers_shader.useProgram();

    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    glBindVertexArray(vao1);

    glActiveTexture(GL_TEXTURE0);
    // 绑定帧缓冲使用的纹理
    glBindTexture(GL_TEXTURE_2D, texture3);
    framebuffers_shader.setUniform1i("tex", 0);

    // imgui控制后期效果切换
    framebuffers_shader.setUniform1i("effect", c.effect);
    if (c.effect == -2)
    {
        glDrawElements(GL_LINE_STRIP, 6, GL_UNSIGNED_INT, 0);
    }
    else
    {
        glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
    }
    // imgui窗口绘制
    c.effectSwitch();
}
```

最终效果

![图片无法显示](../image/opengl/grayscale.png)
![图片无法显示](../image/opengl/edge_detection.png)

# 立方体贴图

立方体贴图由 6 张图片组成，好处在于可以直接通过一个方向向量得到对应位置纹理值，而不再需要纹理坐标。只要立方体在原点，应用立方体贴图的立方体可以比较简单地根据某点在局部坐标系的坐标得到对应位置的纹理。

![图片无法显示](../image/opengl/cubemaps_sampling.png)

## 加载方法

大部分方法没变，改变如下

- 绑定的纹理类型变了
- 生成立方体贴图需要加载 6 张纹理
- `glTexImage2D`函数第一个参数变了，变成立方体的对应方位，代表各个方位的枚举类型可取值如下，

| 纹理目标                       | 方位 |
| :----------------------------- | :--- |
| GL_TEXTURE_CUBE_MAP_POSITIVE_X | 右   |
| GL_TEXTURE_CUBE_MAP_NEGATIVE_X | 左   |
| GL_TEXTURE_CUBE_MAP_POSITIVE_Y | 上   |
| GL_TEXTURE_CUBE_MAP_NEGATIVE_Y | 下   |
| GL_TEXTURE_CUBE_MAP_POSITIVE_Z | 后   |
| GL_TEXTURE_CUBE_MAP_NEGATIVE_Z | 前   |

- 环绕方式需要设为`GL_CLAMP_TO_EDGE`
  > 我们将环绕方式设置为 GL_CLAMP_TO_EDGE，这是因为正好处于两个面之间的纹理坐标可能不能击中一个面（由于一些硬件限制），所以通过使用 GL_CLAMP_TO_EDGE，OpenGL 将在我们对两个面之间采样的时候，永远返回它们的边界值。

```c++

int Texture::LoadCubemaps(const char *const path[], unsigned int *texture, int id, int num, unsigned int wrap_t, unsigned int wrap_s, unsigned int wrap_r, unsigned int min_filter, unsigned int mag_filter)
{
    glActiveTexture(GL_TEXTURE0 + num);
    glGenTextures(id, texture);
    // 第一个参数不再是GL_TEXTURE_2D
    glBindTexture(GL_TEXTURE_CUBE_MAP, *texture);

    int width, height, nrChannels;
    unsigned char *data;

    for (int i = 0; i < 6; i++)
    {
        data = stbi_load(path[i], &width, &height, &nrChannels, 0);
        if (data)
        {
            GLenum format;
            if (nrChannels == 1)
            {
                format = GL_RED;
            }
            else if (nrChannels == 3)
            {
                format = GL_RGB;
            }
            else if (nrChannels == 4)
            {
                format = GL_RGBA;
            }
            // 第一个参数递增的enum类型
            glTexImage2D(GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 0, format, width, height, 0, format, GL_UNSIGNED_BYTE, data);
            stbi_image_free(data);
        }
        else
        {
            std::cout << "load image " << path[i] << " failed" << std::endl;
            return LOAD_IMAGE_FAILED;
        }
    }
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_S, wrap_s);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_T, wrap_t);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_R, wrap_r);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, min_filter);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MAG_FILTER, mag_filter);
    return 0;
}
```

## 天空盒

立方体贴图的一个应用是天空盒。**一个 1*1*1 的中心位置和摄像机位置重合的盒子**。作为场景的背景存在。(天空盒需要单独用一个着色器程序)

因为 **中心位置和摄像机位置重合** 即中心位于观察坐标系的(0, 0, 0)。

天空盒的顶点从局部坐标系转换到裁剪坐标系方法如下：

- 可以首先将单位矩阵作为模型矩阵（可省略），这样天空盒中心和世界坐标系原点重合
- 移除观察矩阵的位移影响，但保留视角转动$mat3(M_{view}) * P_{vertex}$
- 右乘投影矩阵

所以顶点着色器中

```glsl
// 其中view = mat4(mat3(camera.getViewMatrix()))
gl_Position = (projection * view * vec4(aPos, 1.0f));
```

因为作为场景的背景存在，所以需要绘制在场景中所有物体的后面。

效率较低的实现方式是在绘制天空盒的时候先禁止深度缓冲写入，绘制场景中的物体时再启用。
另一种方式就是将深度值置为 1.0f，最深，并且设置深度测试通过方法为`GL_LEQUAL`，即调用`glDepthFunc(GL_LEQUAL)`。
由之前$z_n = z_c/w_c$，可以得知设置深度值为`1.0f`的方法如下

```glsl
gl_Position = (projection * view * vec4(aPos, 1.0f)).xyww;
```

这样做既保证了裁剪坐标不会受影响，深度小于0的该丢弃的片段还是会被丢弃，也使最终得到的标准坐标深度为1

## 环境映射

立方体贴图另一个应用就是环境映射，其中最流行的两个是反射(Reflection)和折射(Refraction)。

### 反射

根据视线方向向量的反射向量取立方体贴图对应位置的纹理，如下图

![图片无法显示](../image/opengl/cubemaps_reflection_theory.png)

所以只需要计算出反射向量即可，反射向量的计算方法在之前的 opengl(4)有说明，着色器中提供了现成的`reflect`函数

所以片段着色器中如下即可

```glsl
    // 立方体贴图只在乎方向，不需要标准化
    vec3 viewDir = FragPos - cameraPosition;
    vec3 reflectDir = reflect(viewDir, Normal);
    // ...
    vec4 reflectTexture = texture(samplerTexture, reflectDir);
    FragColor = reflectTexture;
```

> 反射和折射向量不能在观察坐标系下计算，因为计算出来的结果也是观察坐标系下的，但取立方体贴图上的纹理需要使用世界坐标系的向量

### 折射

根据视线方向向量的折射向量取立方体贴图对应位置的纹理，如下图

![图片无法显示](../image/opengl/cubemaps_refraction_theory.png)

__折射向量计算方法__

![图片无法显示](../image/opengl/refraction.png)

如图设交界面法向量为$\vec{n}$，垂直于法向量且与$\vec{i},\vec{n}$共面的向量为$\vec{m}$(都不需要是法向量)，有

$$
\begin{cases}
\vec{i} = \sin{\alpha}|\vec{i}|\frac{\vec{m}}{|\vec{m}|} - \cos\alpha|\vec{i}|\frac{\vec{n}}{|\vec{n}|}\\
\vec{r} = \sin{\beta}|\vec{r}|\frac{\vec{m}}{|\vec{m}|} - \cos\beta|\vec{r}|\frac{\vec{n}}{|\vec{n}|}\\
\cos\alpha = \frac{\vec{i} \cdot \vec{n}}{|\vec{i}||\vec{n}|}\\
|\vec{r}| = |\vec{i}|
\end{cases}
$$

又有折射率的关系（$n_1,n_2$是两种介质的绝对折射率）

$$
n_1\sin\alpha = n_2\sin\beta
$$

所以

$$
\vec{r} = \frac{n_1}{n_2}(\vec{i} + \frac{\vec{i} \cdot \vec{n}}{|\vec{n}|} \cdot \frac{\vec{n}}{|\vec{n}|}) - \sqrt{1 - \frac{n_1^2}{n_2^2}[1 - (\frac{\vec{i} \cdot \vec{n}}{|\vec{i}||\vec{n}|})^2]}\frac{|\vec{i}|\vec{n}}{|\vec{n}|}
$$


| 材质 | 绝对折射率 |
| :--- | :--------- |
| 空气 | 1.00       |
| 水   | 1.33       |
| 冰   | 1.309      |
| 玻璃 | 1.52       |
| 钻石 | 2.42       |

不过着色器中有现成的`refract`函数
着色器中代码如下

```glsl
    float ratio = 1.00 / 1.52;
    // 立方体贴图只在乎方向，不需要标准化
    vec3 viewDir = FragPos - cameraPosition;
    vec3 refractDir = refract(viewDir, Normal, ratio);
    // ...
    FragColor = texture(samplerTexture, refractDir);
```

### 练习

在之前加载过的模型中引入反射贴图

assimp定义了对应的常量`aiTextureType_REFLECTION`，加载方法和之前相同,
然后片段着色器中

```glsl
    vec4 reflectionTexture = texture(material.texture_reflection1, TexCoords);
    // 下面不一定对，但有效果
    FragColor = vec4((reflectionTexture * reflectTexture + diffuseTexture).xyz, 1.0f);
```

## 最终成果

c++代码

```c++
void Cubemaps::draw()
{
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    glm::mat4 view = camera.getViewMatrix();
    glm::mat4 model(1.0f);

    // 绘制天空盒
    cubemaps_shader.useProgram();
    glActiveTexture(GL_TEXTURE1);
    glBindTexture(GL_TEXTURE_CUBE_MAP, texture2);
    cubemaps_shader.setUniform1i("samplerTexture", 1);
    // 移除位移
    cubemaps_shader.setUniformMatrix4("view", glm::mat3(view));
    cubemaps_shader.setUniformMatrix4("projection", projection);
    box.bind();
    box.draw();

    shader.useProgram();
    // c是控制对象，用来切换绘制的模型类型、效果等等
    if (c.mode == DRAW_MODEL)
    {
        model = glm::translate(model, glm::vec3(c.translate[0], c.translate[1], c.translate[2]));
        model = glm::scale(model, glm::vec3(c.scale[0], c.scale[1], c.scale[2]));
    }
    else
    {
        model = glm::translate(model, glm::vec3(0.0f, 0.0f, -3.0f));
    }
    model = glm::rotate(model, glm::radians(degree), glm::vec3(0.0f, 1.0f, 0.0f));
    glActiveTexture(GL_TEXTURE1);
    glBindTexture(GL_TEXTURE_CUBE_MAP, texture2);
    shader.setUniform1i("samplerTexture", 1);
    shader.setUniformMatrix3("normalMatrix", glm::mat3(glm::transpose(glm::inverse(model))));
    shader.setUniformMatrix4("model", model);
    shader.setUniformMatrix4("view", view);
    shader.setUniformMatrix4("projection", projection);
    shader.setUniform3fv("cameraPosition", camera.getPosition());
    shader.setUniform1i("mapping", c.mapping);
    if (c.mode == DRAW_MODEL)
    {
        m->draw(shader);
    }
    else if(c.mode == DRAW_BOX)
    {
        box.bind();
        box.draw();
    }
    // 绘制ImGui窗口
    c.dataChange();
}
```

顶点着色器

```glsl
#version 330 core
layout (location = 0) in vec3 aFragPos;
layout (location = 1) in vec3 aNormal;
layout (location = 2) in vec2 aTexCoords;

uniform mat3 normalMatrix;
uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

out vec3 Normal;
out vec3 FragPos;
out vec2 TexCoords;

void main()
{
    FragPos = (model * vec4(aFragPos, 1.0f)).xyz;
    gl_Position = projection * view * vec4(FragPos, 1.0f);
    Normal = normalMatrix * aNormal;
    TexCoords = aTexCoords;
}
```

片段着色器
```glsl
#version 330 core

in vec3 Normal;
in vec3 FragPos;
in vec2 TexCoords;

// 只需要反射贴图和漫反射贴图
struct Material
{
    sampler2D texture_diffuse1;
    sampler2D texture_reflection1;
};

uniform samplerCube samplerTexture;
uniform vec3 cameraPosition;
uniform Material material;
uniform int mapping;

out vec4 FragColor;

void main()
{
    float ratio = 1.00 / 1.52;
    vec3 viewDir = FragPos - cameraPosition;
    // 反射向量
    vec3 reflectDir = reflect(viewDir, Normal);
    // 折射向量
    vec3 refractDir = refract(viewDir, Normal, ratio);
    vec4 reflectTexture = texture(samplerTexture, reflectDir);
    vec4 reflectionTexture = texture(material.texture_reflection1, TexCoords);
    vec4 diffuseTexture = texture(material.texture_diffuse1, TexCoords);

    if(mapping == 0)
    {
        // 反射
        FragColor = reflectTexture;
    }
    else if(mapping == 1)
    {
        // 折射
        FragColor = texture(samplerTexture, refractDir);
    }
    else if(mapping == 2)
    {
        // 使用反射贴图
        FragColor = vec4((reflectionTexture * reflectTexture + diffuseTexture).xyz, 1.0f);
    }
}
```

效果

![图片无法显示](../image/opengl/效果演示.gif)

# 参考

1. [《【3D数学基础】三维空间折射向量计算》](https://www.cnblogs.com/night-ride-depart/p/7429618.html) authored by [无尽的数字眩晕症](https://home.cnblogs.com/u/night-ride-depart/)@[博客园](https://www.cnblogs.com/)
2. [learnOpenGL CN](https://learnopengl-cn.github.io/03%20Model%20Loading/01%20Assimp/)