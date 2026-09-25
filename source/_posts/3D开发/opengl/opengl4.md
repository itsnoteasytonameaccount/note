---
title: opengl(4):颜色、基础光照、材质
tags:
  - opengl
date: 2021/01/29
---

为学懂webgl，从opengl入手。本文内容主要来自[LearnOpenGL CN](https://learnopengl-cn.github.io/)文档，加上个人理解，以及一些细节补充、经验记录。本人学习过程中的详细代码参见[此处](https://github.com/itsnoteasytonameaccount/OpenGL-learning)

要学习推荐看原文[LearnOpenGL CN](https://learnopengl-cn.github.io/)，有评论区，加载评论区需要翻墙

另可在[此处](https://www.khronos.org/opengl/wiki/Getting_Started)找到其他opengl官方推荐的资料、教程网站

<!-- more -->

# 颜色

[LearnOpenGL CN](https://learnopengl-cn.github.io/)上给出的反射光的颜色计算方式是两颜色分量相乘

# 基础光照

完全模拟现实的光难度太大，opengl 中可用一些简化模型尽量达到看起来真实的效果

## 冯氏光照模型(Phong Lighting Model)

分为环境光(Ambient)、漫反射(Diffuse)和镜面光照(Specular)

### 环境光

全局光，将之视作全局存在并且所有地方都强度都相同，因此可以较为简单计算。

设置一个变量为环境光强度即可

```glsl
uniform vec3 lightColor;
uniform vec3 objectColor;
// ...
void main()
{
    float ambientStrength = 0.1;
    vec3 ambient = ambientStrength * lightColor;

    vec3 result = ambient * objectColor;
	// ...
}
```

### 漫反射

漫反射方向视作从物体表面反射至所有可以看到该表面的方向，并且强度颜色都相同，类似于求投影，为入射角的余弦值乘以光线 rgb 向量

因此需要求入射角度余弦值$\cos\theta$，需要知道表面的单位法向量$\vec{n}$和入射单位方向向量的相反向量$\vec{l'}$，则:

$$
\cos\theta = \vec{n}\cdot\vec{l'}
$$

在着色器中用世界坐标系计算即可，步骤如下

1.  顶点着色器中定义输出变量`FragPos`，值为顶点的世界坐标系坐标
2.  由于立方体的单位法向量很容易计算，直接在顶点着色器定义输入属性传入每个顶点的法向量`aNormal`，然后用输出变量`Normal`传递到片段着色器。
3.  光的位置是固定的，定义一个`uniform vec3 lightPos`，从程序中传入。
4.  计算结果

顶点着色器中如下

```glsl
layout (location 1) vec3 aPos;
layout (location 1) vec3 aNormal;
out vec3 FragPos;
out vec3 Normal;
uniform mat4 model;
// ...
void main()
{
	// ...
	FragPos = vec3(model * vec4(aPos,1));
	Normal = aNormal;
}
```

片段着色器中如下

```glsl
in Normal;
in FragPos;
uniform vec3 lightPos;
uniform vec3 objectColor;
// ...
void main()
{
	vec3 norm = normalize(Normal);
	vec3 lightDir = normalize(lightPos - FragPos);
	// 点乘求余弦，小于0取0
	float diff = max(dot(norm, lightDir), 0.0);
	vec3 diffuse = diff * lightColor;
	// ...
}
```

其实着色器中`vec3 norm = normalize(Normal);`处我有些疑惑，根据之前的章节所述：

> 顶点着色器主要的目的是把 3D 坐标转为另一种 3D 坐标（后面会解释），同时顶点着色器允许我们对顶点属性进行一些基本处理。片段着色器的主要目的是计算一个像素的最终颜色，这也是所有 OpenGL 高级效果产生的地方。

片段着色器的计算次数应当是比顶点着色器多的，那么对法向量标准化的操作放在顶点着色器中是不是更好呢？（或者直接传入的就是标准化的法向量）

#### 注意

上文说到漫反射是在世界坐标系计算的，因此物体表面的法向量也应当右乘模型矩阵。但若直接右乘模型矩阵有如下问题。

1.  文中法向量是自由向量，不应当受平移变换影响，而只应受缩放和旋转操作影响。
2.  当对法向量应用不等比例的缩放时，法向量会变得不再垂直物体表面

**问题 1**

问题 1 解决比较简单，因为通过矩阵平移需要向量的第四个分量为 1，所以右乘时将法向量第四个分量设为 0 即可，也可以不加第四个分量，让法向量右乘模型矩阵的三阶顺序主子式。

```glsl
Normal = mat3(model) * aNormal;
// 或
// 不过需要注意的是这种方法可能会导致结果的第四个分量不为0，标准化时需要排除第四个分量的影响
Normal = model * vec4(aNormal, 0.0)
```

**问题 2**

第二个问题需要构建一个专门用于法向量的转换矩阵$\large G$

设多边形表面切向量为$\large T$，法向量为$\large N$，模型矩阵为$\large M$，转换后的切向量为$\large T'$，转换后的法向量为$\large N'$，$\large E$为单位向量，则有

$$
\begin{aligned}
\begin{cases}
\large N' = GN\\
\large T' = MT\\
\large N' \cdot T' = N \cdot T = 0
\end{cases} &\implies
\large (GN) \cdot (MT) = N\cdot T \tag{1}\\
&\implies \large (GN)^{\tau}(MT) = N^{\tau}T = 0 \\
&\implies \large N^{\tau}G^{\tau}MT = N^{\tau}T = 0 \\
&\implies \large G^{\tau}M = E\\
&\implies \large G = (M^{-1})^{\tau}
\end{aligned}
$$

<span style="font-size:12px">_(1)向量点乘可以看作对应矩阵相乘_</span>

如果是为了学习，可以直接用着色器的内置求逆矩阵和转置矩阵的函数（考虑到性能，不建议这么做，尽量在 CPU 中计算后传递到着色器）

```glsl
Normal = mat3(transpose(inverse(model))) * aNormal;
```

### 镜面光照

对于镜面反射的光线，要计算镜面分量，需要计算反射光线方向与视线方向夹角的余弦值。

![图片无法显示](../image/opengl/basic_lighting_specular_theory.png)

计算镜面光照一般在观察坐标系下，但是可以用世界坐标来进行计算，这样更符合直觉

在世界坐标系下，假设摄像机位置坐标为$\large P_1$，平面上某点的坐标为$\large P_2$，该点处的法向量为$\large N$，点光源的坐标为$\large P_3$，标准化函数为

$$normalize(\vec{a}) = \frac{\vec{a}}{|\vec{a}|}$$
反射函数为

$$reflect(\vec{a},\vec{n}) = \vec{a} - 2*\frac{\vec{a} \cdot \vec{n}}{|\vec{n}|}*\frac{\vec{n}}{|\vec{n}|}$$

<div style='text-align:center'>

<p><img src='../image/opengl/reflect.png' alt='图片加载失败'/></p>
<p><span style='font-size:12px'>求反射向量方法</span></p>
</div>

最终

$$
\large \cos\theta = normalize(P_1 - P_2) \cdot reflect(normalize(P_2 - P_3),N)
$$

在观察坐标系下只需要乘上 3 阶的观察矩阵即可

$$
\large  \cos\theta = (M_{view} * normalize(P_1 - P_2)) \cdot  (M_{view} * normalize(reflect(P_2 - P_3,N)))
$$

求到余弦值，冯氏光照关于镜面光照需要设置一个反光度参数 shininess，这个参数大概是反映$\cos\theta$和反射光强度之间的关系并不是线性的，另外还需要设置一个参数 specularStrength，避免镜面光照产生太大影响

设 shininess 为 32，最终镜面分量如下

```glsl
float spec = pow(max(dot(viewDir, reflectDir), 0.0), 32);
vec3 specular = specularStrength * spec * lightColor;
```

片段着色器中完整代码如下

```glsl
// 点的位置
in vec3 FragPos;
// 法向量
in vec3 Normal;
// 光源位置
uniform vec3 lightPos;
// 摄像机位置
uniform vec3 viewPos;

void main()
{
	// 和光线的方向向量其实是相反的
	vec3 lightDir = normalize(lightPos - FragPos);
	// 视线方向
	vec3 viewDir = normalize(viewPos - FragPos);
	// 法向量标准化
	vec3 norm = normalize(Normal);
	vec3 reflectDir = reflect(-lightDir, norm);
	float spec = pow(max(dot(viewDir, reflectDir), 0.0), 32);
	vec3 specular = specularStrength * spec * lightColor;
	// ...
}
```

## 在观察坐标系下计算冯氏光照

环境光不用变

**漫反射的计算方法**

传入的点光源和法线矩阵坐标变化

```glsl
shader.setUniform3fv("lightPos", glm::vec3(view * glm::vec4(lightPos, 1.0f)));
object_shader.setUniformMatrix3("normalModel", glm::transpose(glm::inverse(glm::mat3(view * model))));
```

顶点着色器

```glsl
// ...
// 法线矩阵
uniform mat3 normalModel;
void main()
{
	// ...
	FragPos = vec3(view * model * vec4(aPos, 1.0f));
	// FragPos = vec3(model * vec4(aPos, 1.0f));
	Normal = normalModel * aNormal;
}
```

**镜面反射的计算方法**

点光源坐标及顶点着色器中的改变和漫反射相同

需要在片段着色器将摄像机位置坐标视为原点即可

```glsl
// ...
// uniform vec3 viewPos;
void main()
{
	// ...
	vec3 viewDir =  normalize(-FragPos);
    // vec3 viewDir = normalize(viewPos - FragPos);
	// ...
}
```

## Gouraud着色

在顶点着色器中计算冯氏光照的所有分量

顶点着色器

```glsl
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;
uniform mat4 projection;
uniform mat4 view;
uniform mat4 model;
uniform mat3 normalModel;
uniform vec3 lightPos;
uniform vec3 lightColor;
uniform vec3 objectColor;
out vec3 result;

void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0f);
	
	// 顶点观察坐标
	vec3 FragPos = vec3(view * model * vec4(aPos, 1.0f));

	// 计算要用到的向量
    vec3 Normal = normalize(normalModel * aNormal);
	vec3 viewDir = normalize(-FragPos);
	vec3 lightDir = normalize(lightPos - FragPos);
	vec3 reflectDir = reflect(-lightDir, Normal);

	// 计算系数
	float ambientStrength = 0.1f;
	float diff = max(dot(lightDir, Normal), 0.0f);
	float spec = 0.5f * pow(max(dot(reflectDir, viewDir), 0.0f), 32);

	// 计算结果
	result = (ambientStrength + diff + spec) * lightColor * objectColor;
}
```

片段着色器

```glsl
#version 330 core
in vec3 result;
out vec4 color;

void main()
{
	color = vec4(result, 1.0f);
}
```

# 材质

在片段着色器中定义结构体作为物体的材质和光源颜色

片段着色器

```glsl
#version 330 core

struct Material{
    vec3 diffuse;
    vec3 ambient;
    vec3 specular;
    float shininess;
};

struct Light{
    vec3 position;
    vec3 diffuse;
    vec3 specular;
    vec3 ambient;
};

in vec3 FragPos;
in vec3 Normal;
uniform vec3 lightPos;
uniform vec3 lightColor;
uniform Material material;
uniform Light light;
out vec4 FragColor;

void main()
{
    vec3 ambient = light.ambient * material.ambient;
    
    vec3 lightDir = normalize(lightPos - FragPos);
    float diff = max(dot(lightDir, Normal), 0.0f);
    vec3 diffuse = light.diffuse * diff * material.diffuse;

    vec3 viewDir = normalize(-FragPos);
    vec3 reflectDir = reflect(-lightDir, Normal);
    float spec = pow(max(dot(reflectDir, viewDir), 0.0f), material.shininess);
    vec3 specular = light.specular * spec * material.specular;

    FragColor = vec4(ambient + specular + diffuse, 1.0f);
}
```

c++代码中设置方式例如

```c++
// ...
object_shader.setUniform3fv("light.position", glm::vec3(view * glm::vec4(light_pos, 1.0f)));

// 注意：这里的乘不是叉乘只是分量相乘
object_shader.setUniform3fv("light.ambient", light_color * glm::vec3(0.2f));
object_shader.setUniform3fv("light.diffuse", light_color * glm::vec3(0.5f));
object_shader.setUniform3fv("light.specular", glm::vec3(1.0f));
object_shader.setUniform3fv("material.ambient", ambient);
object_shader.setUniform3fv("material.diffuse", diffuse);
object_shader.setUniform3fv("material.specular", specular);
object_shader.setUniform1f("material.shininess", shininess);
// ...
```

# 参考

1. [LearnOpenGL CN](https://learnopengl-cn.github.io/02%20Lighting/02%20Basic%20Lighting/)
2. [百度百科:法向量](https://baike.baidu.com/item/%E6%B3%95%E5%90%91%E9%87%8F/1161324?fr=aladdin)
3. [The Normal Matrix](http://www.lighthouse3d.com/tutorials/glsl-12-tutorial/the-normal-matrix/)
