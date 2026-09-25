---
title: opengl(3):变换、坐标系统、摄像机
tags:
  - opengl
date: 2021/01/06
---

为学懂webgl，从opengl入手。本文内容主要来自[LearnOpenGL CN](https://learnopengl-cn.github.io/)文档，加上个人理解，以及一些细节补充、经验记录。本人学习过程中的详细代码参见[此处](https://github.com/itsnoteasytonameaccount/OpenGL-learning)

要学习推荐看原文[LearnOpenGL CN](https://learnopengl-cn.github.io/)，有评论区，加载评论区需要翻墙

另可在[此处](https://www.khronos.org/opengl/wiki/Getting_Started)找到其他opengl官方推荐的资料、教程网站

# 基础知识

构造坐标系统、实现简单摄像机涉及不少向量矩阵方面的知识。由于比较麻烦，此处只列举会用到的简单知识，暂时搁置这方面的学习，把涉及到的知识点放到参考中，以后学习。
实际上不做很深入地理解思考的话，会一些初高中的数学基础常识就能很好地实现摄像机了。

<!-- more -->

## 矩阵的运算规律

变换都得用到，较简单，不多提

### 库函数的使用

glm 库的变换函数说明，变幻时注意调用顺序

```c++
template<typename T, qualifier Q>
// 相当于第一个参数传入的矩阵左乘相应平移矩阵
GLM_FUNC_QUALIFIER mat<4, 4, T, Q> translate(mat<4, 4, T, Q> const& m, vec<3, T, Q> const& v)
{
    mat<4, 4, T, Q> Result(m);
    // m[0]是model矩阵的第一列
    // Result[3]是结果矩阵的第三列
    Result[3] = m[0] * v[0] + m[1] * v[1] + m[2] * v[2] + m[3];
    return Result;
}

// 相当于m1 * translate(glm::mat4(1.0f), v1) * translate(glm::mat4(1.0f), v2);
m2 = translate(m1, v1);
translate(m2, v2);

template<typename T, qualifier Q>
GLM_FUNC_QUALIFIER mat<4, 4, T, Q> rotate(mat<4, 4, T, Q> const& m, Tangle, vec<3, T, Q> const& v)
{
	T const a = angle;
	T const c = cos(a);
	T const s = sin(a);

	vec<3, T, Q> axis(normalize(v));
	vec<3, T, Q> temp((T(1) - c) * axis);

    // 构建旋转矩阵
	mat<4, 4, T, Q> Rotate;
	Rotate[0][0] = c + temp[0] * axis[0];
	Rotate[0][1] = temp[0] * axis[1] + s * axis[2];
	Rotate[0][2] = temp[0] * axis[2] - s * axis[1];

	Rotate[1][0] = temp[1] * axis[0] - s * axis[2];
	Rotate[1][1] = c + temp[1] * axis[1];
	Rotate[1][2] = temp[1] * axis[2] + s * axis[0];

	Rotate[2][0] = temp[2] * axis[0] + s * axis[1];
	Rotate[2][1] = temp[2] * axis[1] - s * axis[0];
	Rotate[2][2] = c + temp[2] * axis[2];

    // m左乘旋转矩阵
	mat<4, 4, T, Q> Result;
	Result[0] = m[0] * Rotate[0][0] + m[1] * Rotate[0][1] + m[2] * Rotate[0][2];
	Result[1] = m[0] * Rotate[1][0] + m[1] * Rotate[1][1] + m[2] * Rotate[1][2];
	Result[2] = m[0] * Rotate[2][0] + m[1] * Rotate[2][1] + m[2] * Rotate[2][2];
	Result[3] = m[3];
	return Result;
}

template<typename T, qualifier Q>
GLM_FUNC_QUALIFIER mat<4, 4, T, Q> scale(mat<4, 4, T, Q> const& m, vec<3,T, Q> const& v)
{
    // m左乘缩放矩阵
	mat<4, 4, T, Q> Result;
	Result[0] = m[0] * v[0];
	Result[1] = m[1] * v[1];
	Result[2] = m[2] * v[2];
	Result[3] = m[3];
	return Result;
}
```

## 向量积公式

这个差点忘了，而且也不麻烦，复习一下

### 向量积定义

两个向量叉乘，有

$$
|\vec{a}*\vec{b}| = |\vec{a}|*|\vec{b}|*\sin(<\vec{a},\vec{b}>)
$$

结果向量方向与两个向量垂直，并且满足右手螺旋定则（四指握拳，转向和从向量`a`到`b`转向相同，此时大拇指即指向结果向量方向。

### 计算公式推导

若$\vec{i},\vec{j},\vec{k}$为 x,y,z 三个坐标轴方向单位向量，且

$$
\begin{aligned}
&\vec{u} = (a, b, c) = a*\vec{i} + b*\vec{j} + c*\vec{k} \\
&\vec{v} = (e, f, g) = e*\vec{i} + f*\vec{j} + g*\vec{k}
\end{aligned}
$$

则

$$
\vec{u}*\vec{v} = ae(\vec{i}*\vec{i}) + af(\vec{i}*\vec{j}) + ... + cg(\vec{k}*\vec{k})
$$

由上文定义，有

$$
\begin{aligned}
&\vec{i}*\vec{j} = -\vec{j}*\vec{i} = \vec{k} \\
&\vec{i}*\vec{k} = -\vec{k}*\vec{i} = -\vec{j} \\
&\vec{j}*\vec{k} = -\vec{k}*\vec{j} = \vec{i} \\
&\vec{i}*\vec{i} = \vec{j}*\vec{j} = \vec{k}*\vec{k} = \vec{0}
\end{aligned}
$$

最终可推导出

$$
\begin{aligned}
\vec{u}*\vec{v}&= ae\vec{0} + af\vec{k} - ag\vec{j} - be\vec{k} + bf\vec{0} + bg\vec{i} + ce\vec{j} - cf\vec{i} + cg\vec{0}\\
&=(bg - cf)\vec{i} + (ce - ag)\vec{j} + (af - be)\vec{k} + (ae + cg + bf)\vec{0} \\
&=
\begin{pmatrix}
bg - cf,ce - ag,af - be
\end{pmatrix}
\end{aligned}
$$

为了方便记忆，行列式表示(本来不能把向量作为矩阵元素)：

$$
\vec{u}*\vec{v} =
det
\begin{pmatrix}
\vec{i}&\vec{j}&\vec{k} \\
a&b&c \\
e&f&g
\end{pmatrix}
$$

## 平移缩放变换用矩阵表示

$$
\begin{pmatrix}
S_x&0&0&T_x\\
0&S_y&0&T_y\\
0&0&S_z&T_z\\
0&0&0&1
\end{pmatrix} *
\begin{pmatrix}
x\\y\\z\\1
\end{pmatrix} =
\begin{pmatrix}
x*S_x + T_x\\y*S_y+T_y\\z*S_z+T_z\\1
\end{pmatrix}
$$

## 绕坐标轴旋转

3d 空间绕坐标轴旋转很简单，和平面一样就行，绕 z 轴如下,
其中，$\vec{l} = (x_0,y_0)$，$l = |\vec{l}|$：
(以下为逆时针，顺时针$\theta$变成$-\theta$就行)

$$
\begin{aligned}
&y_1 = l\sin{(\alpha+\theta)} = l\sin{\alpha}\cos{\theta} + l\cos{\alpha}\sin{\theta} = y_0\cos{\theta} + x_0\sin{\theta}\\
&x_1 = l\cos{(\alpha+\theta)} = l\cos{\alpha}\cos{\theta} - l\sin{\alpha}\sin{\theta} = x_0\cos{\theta} - y_0\sin{\theta}
\end{aligned}
$$

矩阵

$$
\begin{pmatrix}
\cos{\theta}&-\sin{\theta}&0&0\\
\sin{\theta}&\cos{\theta}&0&0\\
0&0&1&0\\0&0&0&1
\end{pmatrix}
$$

其余同理(同时可得到结论这些矩阵都是正交矩阵),
绕平行于坐标轴的轴旋转，只需进行一些平移操作即可

## 绕任意轴旋转

找了两种方法:一种是用 opengl 比较容易想到的压栈转换坐标系再弹栈<sup>[[5]](#参考资料)</sup>，第二种是用向量计算<sup>[[6]](#参考资料)</sup>，链接附在参考资料中。为巩固记忆，自己看完后根据思路推导一遍。(本文用的都是右手坐标系，参考文献里有的是左手坐标系)

_补充：用下文[转换坐标系](#转换坐标系)的方法也可，转换后在转换回去_

### 坐标轴变换

仅谈论转轴经过原点情况（不经过原点需要先进行一次平移变换），取轴对应的向量$\vec{a} = (R_x,R_y,R_z)$

可能需要了解：将空间中所有点左移可以视作将坐标轴右移，所有点绕轴顺时针旋转可以视作坐标轴绕轴逆时针旋转。

<div style="text-align:center">

</p><img src="../image/opengl/alpha.png" alt="图片无法显示" /></p>
<p><span style="font-size:12px">示意图</span></p>
</div>

通过旋转将$\vec{a}$与 xoz 面重合

首先需要绕 z 轴顺时针旋转，设旋转角度为$\alpha$($\vec{a}$在 xoy 平面上的投影与 x 轴的夹角)，此时变换矩阵为

$$
\begin{aligned}
\large P_1 &=
\begin{pmatrix}
\cos{\alpha}&\sin{\alpha}&0&0\\-\sin{\alpha}&\cos{\alpha}&0&0\\0&0&1&0\\0&0&0&1
\end{pmatrix}\\\\
&=\begin{pmatrix}
R_x \over \sqrt{R_x^2 + R_y^2} &  \frac{R_y}{\sqrt{R_x^2+R_y^2}} & 0 &0\\
-\frac{R_y}{\sqrt{R_x^2 + R_y^2}} & R_x \over \sqrt{R_x^2 + R_y^2} & 0 & 0 \\
0&0&1&0\\0&0&0&1
\end{pmatrix}
\end{aligned}
$$

再绕 y 轴旋转至与 z 轴重合，设此时$\vec{a}$与 z 轴夹角为$\beta$，可得

<div style="text-align:center">

<p><img src="../image/opengl/beta.png" alt="图片无法显示"/></p>
<p style="font-size:12px">第一次旋转后向量位置</div>
</div>

$$
\begin{aligned}
\large P_2 &=
\begin{pmatrix}
\cos{\beta}&0&-\sin{\beta}&0  \\
0&1&0&0\\
\sin{\beta}& 0&\cos{\beta} & 0\\
0&0&0&1
\end{pmatrix} \\\\ &=
\begin{pmatrix}
 \frac{R_z}{\sqrt{R_x^2+R_y^2+R_z^2}}&0&-\frac{\sqrt{R_x^2 + R_y^2}}{\sqrt{R_x^2+R_y^2+R_z^2}} & 0  \\
0&1&0&0\\
\frac{\sqrt{R_x^2+R_y^2}}{\sqrt{R_x^2+R_y^2+R_z^2}}&0&R_z \over \sqrt{R_x^2+R_y^2+R_z^2} &0 \\
0&0&0&1
\end{pmatrix}
\end{aligned}
$$

最后绕 z 轴旋转$\theta$度的矩阵为

$$
\large A =
\begin{pmatrix}
\cos{\theta}&-\sin{\theta}&0&0\\
\sin{\theta}&\cos{\theta}&0&0\\
0&0&1&0\\0&0&0&1
\end{pmatrix}
$$

$\large P_1 \large P_2$的逆矩阵正好是它们的转置矩阵,总体变换

$$
\begin{aligned}
 \large P
 &= \large P_1^{\tau}P_2^{\tau}AP_2P_1
\end{aligned}
$$

不知道是不是本人推导过程出了问题，结果实在难算，就不算了，这个果然只有用代码实现才可以用。

### 向量运算

对空间任一点$p=(p_x,p_y,p_z)$绕任意轴 a 旋转，轴 a 对应的单位向量$\vec{a} = (r_x,r_y,r_z)$已知，用自由向量$\vec{P} = (p_x,p_y,p_z)$代替点 p 来讨论点 p 坐标变化，将$\vec{P}$分解为轴 a 上的分量$\vec{A}$以及垂直于轴 A 的分量$\vec{A_1}$，如图

![图片无法显示](../image/opengl/分量.png)

有

$$
\begin{cases}
\vec{A} = (\vec{P}\cdot\vec{a})\vec{a} \\\tag{1}
\vec{A_1} = \vec{P} - \vec{A}
\end{cases}
$$

将$\vec{P}$旋转一定角度后得到$\vec{P_1} = \vec{A} + \vec{A_2}$
可以将$\vec{A_2}$分解为与$\vec{A_1}$方向相同的$\vec{A_4}$和垂直于$\vec{A_1}$的$\vec{A_3}$

可得

$$
\begin{aligned}
\vec{A_4}=\cos{\alpha}\vec{A_1} \tag{2}\\
\end{aligned}
$$

由右手螺旋定则又有$\vec{A}*\vec{P}$方向与$\vec{A_3}$相同，以下等式成立

$$
\begin{cases}
\lambda\vec{A}*\vec{P} = \vec{A_3}  \\
|\vec{A}*\vec{P}| = |\vec{A}|*|\vec{A_1}| \\
|\vec{A_3}| = |\vec{A_1}|*\sin{\alpha} \tag{3}
\end{cases}
$$

解方程组(3)可得

$$\lambda = \frac{\sin{\alpha}}{|\vec{A}|}$$

$$
\vec{A_3} = \frac{\sin{\alpha}*\vec{A}*\vec{P}}{|\vec{A}|} = \sin{\alpha}*\vec{a}*\vec{P} \tag{4}
$$

联立(1),(2),(4)最终可得

$$
\begin{aligned}
\vec{P_1} &= \vec{A_2} + \vec{A} \\
& = \vec{A_3} + \vec{A_4} + \vec{A} \\
&= \sin{\alpha}*\vec{a}*\vec{P} + \cos{\alpha}(\vec{P} - (\vec{P}\cdot\vec{a})\vec{a})+(\vec{P}\cdot\vec{a})\vec{a}
\end{aligned}
$$

用矩阵表示

$$
(\vec{P}\cdot\vec{a})\vec{a} =
\begin{pmatrix}
r_x^2 & r_xr_y & r_xr_z  \\
r_xr_y & r_y^2 & r_yr_z \\
r_xr_z & r_yr_z & r_z^2
\end{pmatrix} *
\begin{pmatrix}
p_x\\p_y\\p_z
\end{pmatrix}
$$

$$
\sin{\alpha}*\vec{a}*\vec{P} =
\sin{\alpha}
\begin{pmatrix}
0 & -r_z & r_y \\
r_z & 0 & -r_x \\
-r_y & r_x & 0
\end{pmatrix}*
\begin{pmatrix}
p_x\\p_y\\p_z
\end{pmatrix}
$$

所以

$$
\begin{aligned}
\vec{P} &= (
  \sin{\alpha}
\begin{pmatrix}
0 & -r_z & r_y \\
r_z & 0 & -r_x\\
-r_y & r_x & 0
\end{pmatrix} + \cos{\alpha}\large E + (1 - \cos\alpha)\begin{pmatrix}
r_x^2 & r_xr_y & r_xr_z \\
r_xr_y & r_y^2 & r_yr_z\\
r_xr_z & r_yr_z & r_z^2
\end{pmatrix}
)*
\begin{pmatrix}
p_x\\p_y\\p_z
\end{pmatrix}\\
&=\begin{pmatrix}
\cos\alpha + r_x^2(1-\cos\alpha) & r_xr_y(1-\cos\alpha)-r_z\sin\alpha & r_xr_z(1-\cos\alpha)+r_y\sin\alpha \\
r_xr_y(1-\cos\alpha)+r_z\sin\alpha & \cos\alpha + r_y^2(1-\cos\alpha) & r_yr_z(1-\cos\alpha) - r_x\sin\alpha\\
r_xr_z(1-\cos\alpha) - r_y\sin\alpha & r_yr_z(1-\cos\alpha)+r_x\sin\alpha & \cos\alpha + r_z^2(1-\cos\alpha)
\end{pmatrix}*
\begin{pmatrix}
p_x\\p_y\\p_z
\end{pmatrix}\\
\end{aligned}
$$

![图片无法显示](../image/opengl/分解.png)

# 坐标系统

为了方便操作，opengl 引入了

- 局部空间(local space)
- 世界空间(world space)
- 观察空间(view space)
- 裁剪空间(clip space)
- 屏幕空间(screen space)

等概念，实际上是把从绘制到显示一个物体需要经历的步骤分离。

## 局部空间

本人的理解是：在绘制物体时，将绘制中的物体视作处于一个其他物体不同的独立空间，此时不考虑物体在空间中的实际状态（包括位置，旋转方向等），只考虑物体形状。这样的独立空间称为局部空间。

> **思考如下情况:**
> 若你需要绘制一个立方体，边长为 2，中心的坐标为$(x_1,y_1,z_1)$，在未分离世界空间和局部空间概念的情况下，可能你会计算出立方体顶点的坐标为$(x_1\pm1,y_1\pm1,z_1\pm1)$(或许不会有人这样傻，但此处只是拿个简单例子说明问题)。此时如果先不考虑立方体的位置，先绘制一个中心点为(0,0,0)，顶点坐标为$(\pm1,\pm1,\pm1)$的立方体，再通过变换矩阵$\large M_{model}$来改变立方体的位置（以及进行一些其他变换操作），就要简单得多。

上述不考虑立方体状态的坐标称为局部坐标，改变立方体状态的矩阵$\large M_{model}$可以模型矩阵，应用于每个物体上的模型矩阵是不同的。

模型矩阵的构造方法已在上文给出，组合各种平移、旋转、缩放矩阵即可

## 世界空间

相对的，经过了模型矩阵变换操作后物体位于的空间就称为世界空间。

## 观察空间

物体位于世界空间中相应的位置后，还需要设置观察物体的视角等，否则永远只能看到物体的一个面，此时就引入观察空间的概念（也被称为摄像机空间(Camera Space)或视觉空间(Eye Space)）。

用一个矩阵$\large M_{view}$改变世界空间中所有物体的坐标，可以视作改变了观察的位置和方向。矩阵$\large M_{view}$称为观察矩阵，观察矩阵作用于所有物体，通过观察矩阵改变后的空间称为观察空间。

观察矩阵的构造方法会在下文[摄像机](#摄像机)的部分给出

## 裁剪空间

> 在一个顶点着色器运行的最后，OpenGL 期望所有的坐标都能落在一个特定的范围内，且任何在这个范围之外的点都应该被裁剪掉(Clipped)。被裁剪掉的坐标就会被忽略，所以剩下的坐标就将变为屏幕上可见的片段。这也就是裁剪空间(Clip Space)名字的由来。

> 由投影矩阵创建的观察箱(Viewing Box)被称为平截头体(Frustum)，每个出现在平截头体范围内的坐标都会最终出现在用户的屏幕上。将特定范围内的坐标转化到标准化设备坐标系的过程（而且它很容易被映射到 2D 观察空间坐标）被称之为投影(Projection)，因为使用投影矩阵能将 3D 坐标投影(Project)到很容易映射到 2D 的标准化设备坐标系中。

> 一旦所有顶点被变换到裁剪空间，最终的操作——透视除法(Perspective Division)将会执行，在这个过程中我们将位置向量的 x，y，z 分量分别除以向量的齐次 w 分量；透视除法是将 4D 裁剪空间坐标变换为 3D 标准化设备坐标的过程。这一步会在每一个顶点着色器运行的最后被自动执行。

> 将观察坐标变换为裁剪坐标的投影矩阵可以为两种不同的形式，每种形式都定义了不同的平截头体。我们可以选择创建一个正交投影矩阵(Orthographic Projection Matrix)或一个透视投影矩阵(Perspective Projection Matrix)。

<div style='text-align:center'>

![图片加载失败](../image/opengl/正射投影.png)

<span style='font-size:12px'>正交投影（有的地方叫正射投影）</span>

</div>

<div style='text-align:center'>

<p><img src="../image/opengl/perspective_frustum.png" alt="图片无法显示" /></p>
<p><span style='font-size:12px'>透视投影</span></p>

</div>

### 构建透视投影矩阵

为了让屏幕按照预期显示我们绘制的内容，需要将如下的透视视锥映射为标准化设备坐标系对应的坐标范围

<div style='text-align:center'>

<p><img src="../image/opengl/透视平头截体.png" alt="图片无法显示" /></p>
<p><span style='font-size:12px'>透视视锥和标准化坐标系空间</span></p>
</div>

为达到这一目的，需要进行如下两步

1. 观察坐标转化为裁剪坐标
2. 裁剪坐标转化为标准化坐标

$$
\begin{pmatrix}
x_c\\y_c\\z_c\\w_c
\end{pmatrix} = \large M_{proj}
\begin{pmatrix}
x_e\\y_e\\z_e\\w_e
\end{pmatrix} \tag{1}
$$

$$
\begin{pmatrix}
x_n\\y_n\\z_n
\end{pmatrix} =
\begin{pmatrix}
x_c/w_c\\y_c/w_c\\z_c/w_c
\end{pmatrix} \tag{2}
$$

第二步由 Opengl 自动完成，需要求的是第一步的投影矩阵$\large M_{proj}$

#### 步骤

已知视锥的深度范围(-n,-f)_(实际上-f<-n，这样写的原因是观察坐标系是右手坐标系，标准设备坐标系是左手坐标系，z 轴方向相反)_，以及近处的横坐标范围(l,r)，纵坐标范围(b,t)，接下来需要将视锥中所有的点坐标映射到裁剪坐标

横坐标和纵坐标映射为标准设备坐标分为如下两步

**映射为投影坐标:** 如图所示，首先要求视锥中某点$(x_e,y_e,z_e)$在近处平面上的投影横坐标$x_p$和纵坐标$y_p$，根据相似很容易得到$x_p = -\frac{n}{z_e}x_e,y_p = -\frac{n}{z_e}y_e$

<div style='text-align:center'>

<p><img src="../image/opengl/映射到裁剪坐标.png" alt="图片无法显示" />
<p><span style='font-size:12px'>求投影坐标</span></p>

</div>

**映射标准设备坐标:** 将求到的$x_p$线性映射到标准设备坐标，设映射函数为

$$
x_n = \frac{2}{r-l}x_p + \beta
$$

将$x_p = r$时$x_n = 1$代入，得$\beta = -\frac{l+r}{r-l}$

最终求到

$$
x_n = -\frac{2n}{(r-l)z_e}x_e - \frac{l+r}{r-l} = (\frac{2n}{r-l}x_e + \frac{l+r}{r-l}z_e)/-z_e
$$

同理

$$
y_n = (\frac{2n}{t-b}y_e + \frac{t+b}{t-b}z_e)/-z_e
$$

又有

$$
y_n = y_c/w_c, x_n = x_c/w_c
$$

故可取$w_c = -z_e$，此时

$$
x_c = \frac{2n}{r-l}x_e + \frac{l+r}{r-l}z_e,y_c = \frac{2n}{t-b}y_e + \frac{t+b}{t-b}z_e
$$

2. 深度的映射略有不同，$z_c$和$x_e,y_e$无关，可以构造线性关系

$$
z_c = Az_e + Bw_e
$$

_注：这个关系的常数不是 B 而是$Bw_e$是矩阵乘法的规则使然，z_c 是由观察坐标乘以投影矩阵得到的_

则

$$
z_n = z_c/w_c = \frac{Az_e + Bw_e}{-z_e}
$$

将$w_e = 1$以及$z_e = -f$时$z_n = 1$,$z_e = -n$时$z_n = -1$代入，得

$$
\begin{cases}
-Af + B = f \\
-An + B = -n
\end{cases}
$$

解得

$$
\begin{cases}
A = \frac{n+f}{n-f}\\
B = \frac{2fn}{n-f}
\end{cases}\implies z_c = \frac{n+f}{n-f}z_e + \frac{2fn}{n-f}w_e
$$

总结

$$
\begin{cases}
&\frac{2n}{r-l}x_e&&+\frac{l+r}{r-l}z_e &&= x_c\\
&&\frac{2n}{t-b}y_e&+\frac{t+b}{t-b}z_e &&= y_c\\
&&&\frac{n+f}{n-f}z_e &+\frac{2fn}{n-f}w_e &= z_c\\
&&&-z_e &&= w_c
\end{cases} \implies \large M_{proj} = \begin{pmatrix}
\frac{2n}{r-l}&0&\frac{l+r}{r-l}&0\\
0&\frac{2n}{t-b}&\frac{t+b}{t-b} &0\\
0&0&\frac{n+f}{n-f} &\frac{2fn}{n-f}\\
0&0&-1&0
\end{pmatrix}
$$

当$l = -r,b = -t$时

$$
\large M_{proj} = \begin{pmatrix}
\frac{n}{r}&0&0&0\\
0&\frac{n}{t}&0 &0\\
0&0&\frac{n+f}{n-f} &\frac{2fn}{n-f}\\
0&0&-1&0
\end{pmatrix}
$$

> 注意：由于$z_n$和$z_e$不是线性关系，而是如下关系

$$
z_n = \frac{z_c}{w_c} = \frac{n+f}{f-n} + \frac{2fn}{(f-n)z_e}
$$

> 图像如下，所以当深度范围较大的时候，较远处两个邻近点的深度差距可能小于浮点数的精确度，导致比较深度时判断错误，为了尽量避免这个问题，建议不要将 f,n 之间的距离设得太大

<div style='text-align:center'>

<p><img src="../image/opengl/关系图像.png" alt="图片无法显示" /></p>
<p><span style='font-size:12px'>z<sub>n</sub>与z<sub>e</sub>关系图</span></p>
</div>

### 构建正交投影矩阵

可参考[参考资料](#参考资料)第 7 条

# 摄像机

> OpenGL 本身没有摄像机(Camera)的概念，但我们可以通过把场景中的所有物体往相反方向移动的方式来模拟出摄像机，产生一种我们在移动的感觉，而不是场景在移动。

因此需要构建观察矩阵来转换坐标。

## 转换坐标系

以二维平面为例，考虑新坐标原点和当前坐标系原点重合的情况(原点不重合加一个平移操作即可)。

设$\vec{\alpha} = (x_1,y_1),\vec{\beta} = (x_2,y_2)$为新坐标系横纵坐标上的单位向量。
对任意点用向量表示$\vec{p} = (x,y)$,$\vec{p}$在$\vec{\alpha},\vec{\beta}$上的投影长度分别为$\vec{p}\cdot\vec{\alpha},\vec{p}\cdot\vec{\beta}$，所以新坐标系中点坐标为$(x_1x + y_1y,x_2x+y_2y)$

对齐次坐标的转换矩阵为

$$
\begin{pmatrix}
x_1&y_1&0\\
x_2&y_2&0\\
0&0&1
\end{pmatrix}
$$

三维空间同理

## 世界坐标转化为观察坐标

> 当我们讨论摄像机/观察空间(Camera/View Space)的时候，是在讨论以摄像机的视角作为场景原点时场景中所有的顶点坐标：观察矩阵把所有的世界坐标变换为相对于摄像机位置与方向的观察坐标。

显然需要用到[转换坐标系](#转换坐标系)的方法，在此之前需要根据摄像机的位置，角度等构建新的坐标系。

> 要定义一个摄像机，我们需要它在世界空间中的位置、观察的方向、一个指向它右测的向量以及一个指向它上方的向量。细心的读者可能已经注意到我们实际上创建了一个三个单位轴相互垂直的、以摄像机的位置为原点的坐标系。

## 新坐标系构建方法

<div style='text-align:center'>

<p><img src="../image/opengl/camera_axes.png" alt="图片无法显示" /></p>
<p><span style='font-size:12px'>逐步构建坐标系</span></p>
</div>

1. 假设已知摄像机的位置$(a,b,c)$以及摄像机正前方某点的位置$(x_0,y_0,z_0)$，
   可以得到方向向量（上图紫色箭头）$\vec{d} = (a - x_0, b - y_0,c - z_0)$
2. 不考虑摄像机会绕自身轴线滚转的情况下，可以取向上的单位向量$\vec{j} = (0,1,0)$，此时指向摄像机正右的向量（上图红色箭头）$\vec{r} = \vec{j} * \vec{d}$，_(前提是$\vec{d}$和$\vec{j}$线性无关,即摄像机俯仰角度不为 90 度)_
3. 最后指向摄像机正上方的向量（上图绿色箭头）为$\vec{u} = \vec{d} * \vec{r}$
4. 将三个向量转化为单位向量$\vec{D} = (D_x,D_y,D_z), \vec{\R} = (R_x,R_y,R_z), \vec{U} = (U_x,U_y,U_z)$
5. 最后根据上文提到的[转换坐标系](#转换坐标系)的方法，构建观察矩阵
   <a id="link"></a>

   $$
   LookAt = \begin{pmatrix}
   R_x&R_y&R_z&0\\
   U_x&U_y&U_z&0\\
   D_x&D_y&D_z&0\\
   0&0&0&1
   \end{pmatrix} * \begin{pmatrix}
   1&0&0&-a\\
   0&1&0&-b\\
   0&0&1&-c\\
   0&0&0&1
   \end{pmatrix}
   $$

之前用到的 glm 库实现了构建观察矩阵的 LookAt 方法

```c++
// 返回观察矩阵
detail::tmat4x4<T> glm::gtc::matrix_transform::lookAt
(
    // 摄像机位置
    detail::tvec3< T > const &  eye,
    // 看向的目标点坐标
    detail::tvec3< T > const &  center,
    // 指向摄像机上方的向量
    detail::tvec3< T > const &  up
)
```

```c++
template<typename T, qualifier Q>
	GLM_FUNC_QUALIFIER mat<4, 4, T, Q> lookAtRH(vec<3, T, Q> const& eye, vec<3, T, Q> const& center, vec<3, T, Q> const& up)
	{
		vec<3, T, Q> const f(normalize(center - eye));
		vec<3, T, Q> const s(normalize(cross(f, up)));
        // 此处不进行标准化是因为 |u| = |f||s|sin(pi/2) = 1,不需要标准化
		vec<3, T, Q> const u(cross(s, f));

		mat<4, 4, T, Q> Result(1);
		Result[0][0] = s.x;
		Result[1][0] = s.y;
		Result[2][0] = s.z;
		Result[0][1] = u.x;
		Result[1][1] = u.y;
		Result[2][1] = u.z;
		Result[0][2] =-f.x;
		Result[1][2] =-f.y;
		Result[2][2] =-f.z;
		Result[3][0] =-dot(s, eye);
		Result[3][1] =-dot(u, eye);
		Result[3][2] = dot(f, eye);
		return Result;
	}
```

## 位置移动

位置移动很简单，只需要改变[上文](#link)观察矩阵的平移矩阵部分即可

直接用 glm 的 lookAt 函数如下。

```c++
glm::vec3 cameraPos   = glm::vec3(0.0f, 0.0f,  3.0f);
// 和上文的方向向量方向相反
glm::vec3 cameraFront = glm::vec3(0.0f, 0.0f, -1.0f);
glm::vec3 cameraUp    = glm::vec3(0.0f, 1.0f,  0.0f);

// 生成观察矩阵的方法
view = glm::lookAt(cameraPos, cameraPos + cameraFront, cameraUp);

// 改变位置的方法
void processInput(GLFWwindow *window)
{
    ...
    float cameraSpeed = 0.05f; // adjust accordingly
    if (glfwGetKey(window, GLFW_KEY_W) == GLFW_PRESS)
        // 沿方向向量
        cameraPos += cameraSpeed * cameraFront;
    if (glfwGetKey(window, GLFW_KEY_S) == GLFW_PRESS)
        // 沿方向向量
        cameraPos -= cameraSpeed * cameraFront;
    if (glfwGetKey(window, GLFW_KEY_A) == GLFW_PRESS)
        // 沿右向量
        cameraPos -= glm::normalize(glm::cross(cameraFront, cameraUp)) * cameraSpeed;
    if (glfwGetKey(window, GLFW_KEY_D) == GLFW_PRESS)
        // 沿右向量
        cameraPos += glm::normalize(glm::cross(cameraFront, cameraUp)) * cameraSpeed;
}
```

> 注意，我们对右向量进行了标准化。如果我们没对这个向量进行标准化，最后的叉乘结果会根据 cameraFront 变量返回大小不同的向量。如果我们不对向量进行标准化，我们就得根据摄像机的朝向不同加速或减速移动了，但如果进行了标准化移动就是匀速的。

### 移动速度

根据两次帧渲染之间的时间差改变每次移动时的距离大小，可以使得不同系统上移动速度体验相同，具体参见[此处](https://learnopengl-cn.github.io/01%20Getting%20started/09%20Camera/#_4)。

## 视角移动

### 欧拉角

> 欧拉角(Euler Angle)是可以表示 3D 空间中任何旋转的 3 个值，由莱昂哈德·欧拉(Leonhard Euler)在 18 世纪提出。一共有 3 种欧拉角：俯仰角(Pitch)、偏航角(Yaw)和滚转角(Roll)，下面的图片展示了它们的含义

<div style='text-align:center'>

<p><img src='../image/opengl/camera_pitch_yaw_roll.png' alt='图片加载失败'/></p>
<p><span style='font-size:12px'>俯仰角(pitch)、偏航角(yaw)、滚转角(roll)</span></p>
</div>

用欧拉角表示摄像机视角状态较为直观，不考虑滚转角情况下计算前轴（和上文提到的方向向量方向相反）方法如下（当然偏航角也可以定义为投影与 z 轴夹角，此时需要修改 front 向量的 x,z 计算方法）

<div style='text-align:center'>

<p><img src='../image/opengl/yaw_pitch.png' alt='图片加载失败'/></p>
<p><span style='font-size:12px'>欧拉角计算前向量</span></p>
</div>

```c++
front.x = cos(glm::radians(pitch)) * cos(glm::radians(yaw));
front.z = cos(glm::radians(pitch)) * sin(glm::radians(yaw));
front.y = sin(glm::radians(pitch));
```

## 摄像机类

有上述铺垫理解摄像机类示例就很简单

```c++
#ifndef CAMERA_H
#define CAMERA_H

#include <glad/glad.h>
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>

#include <vector>

// Defines several possible options for camera movement. Used as abstraction to stay away from window-system specific input methods
enum Camera_Movement {
    FORWARD,
    BACKWARD,
    LEFT,
    RIGHT
};

// Default camera values
// 初始font = (0,0,-1)与x轴夹角-pi/2
const float YAW         = -90.0f;
const float PITCH       =  0.0f;
const float SPEED       =  2.5f;
const float SENSITIVITY =  0.1f;
const float ZOOM        =  45.0f;


// An abstract camera class that processes input and calculates the corresponding Euler Angles, Vectors and Matrices for use in OpenGL
class Camera
{
public:
    // camera Attributes
    glm::vec3 Position;
    glm::vec3 Front;
    glm::vec3 Up;
    glm::vec3 Right;
    glm::vec3 WorldUp;
    // euler Angles
    float Yaw;
    float Pitch;
    // camera options
    float MovementSpeed;
    float MouseSensitivity;
    float Zoom;

    // constructor with vectors
    Camera(glm::vec3 position = glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3 up = glm::vec3(0.0f, 1.0f, 0.0f), float yaw = YAW, float pitch = PITCH) : Front(glm::vec3(0.0f, 0.0f, -1.0f)), MovementSpeed(SPEED), MouseSensitivity(SENSITIVITY), Zoom(ZOOM)
    {
        Position = position;
        WorldUp = up;
        Yaw = yaw;
        Pitch = pitch;
        updateCameraVectors();
    }
    // constructor with scalar values
    Camera(float posX, float posY, float posZ, float upX, float upY, float upZ, float yaw, float pitch) : Front(glm::vec3(0.0f, 0.0f, -1.0f)), MovementSpeed(SPEED), MouseSensitivity(SENSITIVITY), Zoom(ZOOM)
    {
        Position = glm::vec3(posX, posY, posZ);
        WorldUp = glm::vec3(upX, upY, upZ);
        Yaw = yaw;
        Pitch = pitch;
        updateCameraVectors();
    }

    // returns the view matrix calculated using Euler Angles and the LookAt Matrix
    glm::mat4 GetViewMatrix()
    {
        return glm::lookAt(Position, Position + Front, Up);
    }

    // processes input received from any keyboard-like input system. Accepts input parameter in the form of camera defined ENUM (to abstract it from windowing systems)
    void ProcessKeyboard(Camera_Movement direction, float deltaTime)
    {
        // 沿观察坐标系的坐标轴移动
        float velocity = MovementSpeed * deltaTime;
        if (direction == FORWARD)
            Position += Front * velocity;
        if (direction == BACKWARD)
            Position -= Front * velocity;
        if (direction == LEFT)
            Position -= Right * velocity;
        if (direction == RIGHT)
            Position += Right * velocity;
    }

    // processes input received from a mouse input system. Expects the offset value in both the x and y direction.
    void ProcessMouseMovement(float xoffset, float yoffset, GLboolean constrainPitch = true)
    {
        xoffset *= MouseSensitivity;
        yoffset *= MouseSensitivity;

        Yaw   += xoffset;
        Pitch += yoffset;

        // make sure that when pitch is out of bounds, screen doesn't get flipped
        // 限制俯仰角
        if (constrainPitch)
        {
            if (Pitch > 89.0f)
                Pitch = 89.0f;
            if (Pitch < -89.0f)
                Pitch = -89.0f;
        }

        // update Front, Right and Up Vectors using the updated Euler angles
        updateCameraVectors();
    }

    // processes input received from a mouse scroll-wheel event. Only requires input on the vertical wheel-axis
    // 滚轮改变视角大小来产生缩放效果
    void ProcessMouseScroll(float yoffset)
    {
        Zoom -= (float)yoffset;
        if (Zoom < 1.0f)
            Zoom = 1.0f;
        if (Zoom > 45.0f)
            Zoom = 45.0f;
    }

private:
    // calculates the front vector from the Camera's (updated) Euler Angles
    void updateCameraVectors()
    {
        // calculate the new Front vector
        // 每次欧拉角改变时都需要更新三个坐标轴方向上的向量
        glm::vec3 front;
        front.x = cos(glm::radians(Yaw)) * cos(glm::radians(Pitch));
        front.y = sin(glm::radians(Pitch));
        front.z = sin(glm::radians(Yaw)) * cos(glm::radians(Pitch));
        Front = glm::normalize(front);
        // also re-calculate the Right and Up vector
        Right = glm::normalize(glm::cross(Front, WorldUp));  // normalize the vectors, because their length gets closer to 0 the more you look up or down which results in slower movement.
        Up    = glm::normalize(glm::cross(Right, Front));
    }
};
```

## 疑问

不知道有没有人和我有过相同的的问题，直接保存一个观察矩阵，在每次摄像机位置移动和摄像机方向改变时用矩阵乘法更新观察矩阵不就可以实现摄像机类吗，何必像上面那么麻烦。

如下

```c++
class Camera2
{
public:
    float MovementSpeed;
    float MouseSensitivity;
    float Zoom;
    glm::mat4 matrix;
    glm::mat4 unit_matrix;
    glm::vec3 position;
    bool isMoved = false;
    float lastX, lastY;

    // constructor with vectors
    Camera2(glm::vec3 position = glm::vec3(0.0f, 0.0f, 3.0f)):MovementSpeed(SPEED), MouseSensitivity(SENSITIVITY), Zoom(ZOOM), matrix(1.0f), unit_matrix(1.0f)
    {
        this->position = position;
        isMoved = true;
    }

    // returns the view matrix calculated using Euler Angles and the LookAt Matrix
    glm::mat4 GetViewMatrix()
    {
        if (isMoved)
        {
            matrix = glm::translate(unit_matrix, -position) * matrix;
            position = glm::vec3(0.0f);
            isMoved = false;
        }
        return matrix;
    }

    // processes input received from any keyboard-like input system. Accepts input parameter in the form of camera defined ENUM (to abstract it from windowing systems)
    void ProcessKeyboard(Camera_Movement direction, float deltaTime)
    {
        // 直接改变position坐标
        float velocity = MovementSpeed * deltaTime;
        isMoved = true;
        switch (direction)
        {
        case FORWARD:
            position.z -= velocity;
            break;
        case BACKWARD:
            position.z += velocity;
            break;
        case LEFT:
            position.x -= velocity;
            break;
        case RIGHT:
            position.x += velocity;
            break;
        default:
            break;
        }
    }

    // 注意这里的参数是位置，不是偏移量
    void ProcessMouseMovement(float xpos, float ypos, GLboolean constrainPitch = true)
    {
        float xoffset = xpos - lastX;
        // 鼠标位移的y轴和绘图的y轴方向是反的
        float yoffset = lastY - ypos;
        lastX = xpos;
        lastY = ypos;
        xoffset *= MouseSensitivity;
        yoffset *= MouseSensitivity;
        if (isMoved)
        {
            matrix = glm::translate(unit_matrix, -position) * matrix;
            position = glm::vec3(0.0f);
            isMoved = false;
        }
        // 更新矩阵
        matrix = glm::rotate(unit_matrix, glm::radians(abs(xoffset / 2) + abs(yoffset / 2)), glm::vec3(-yoffset, xoffset, 0)) * matrix;
    }

    // processes input received from a mouse scroll-wheel event. Only requires input on the vertical wheel-axis
    void ProcessMouseScroll(float yoffset)
    {
        Zoom -= (float)yoffset;
        if (Zoom < 1.0f)
            Zoom = 1.0f;
        if (Zoom > 45.0f)
            Zoom = 45.0f;
    }
};
#endif
```

而且我还试了一下发现这种方法效率也更高

```c++
#include "Camera.hpp"
#include <GLFW/glfw3.h>
#include <iostream>
#include <time.h>

int main()
{
    Camera camera;
    glfwInit();
    std::cout << "time:" << glfwGetTime() << std::endl;
    for (int i = 0; i < 1000000; i++)
    {
        camera.ProcessKeyboard(FORWARD, 1.0f);
        camera.ProcessMouseMovement(0, 0);
        camera.GetViewMatrix();
    }
    std::cout << "time:" << glfwGetTime() << std::endl;
    Camera2 camera2;
    std::cout << "time2:" << glfwGetTime() << std::endl;
    for (int i = 0; i < 1000000; i++)
    {
        camera2.ProcessKeyboard(FORWARD, 1.0f);
        camera2.ProcessMouseMovement(0, 0);
        camera2.GetViewMatrix();
    }
    std::cout << "time2:" << glfwGetTime() << std::endl;
    glfwTerminate();
    return 0;
}
```

![图片无法显示](../image/opengl/效率比较.png)

# 计算

在得到，模型矩阵$M_{model}$，观察矩阵$M_{view}$，透视投影矩阵$M_{proj}$，以及代表顶点位置的向量$V_{pointer}$后，最终顶点着色器计算的顶点值应当是$M_{proj}*M_{view}*M_{model}*V_{pointer}$

# 总结

现在我们已经对创建 opengl 窗口并绘图的基础流程有了一定了解，下面对步骤进行总结。

1. `glfwInit()`初始化 glfw
2. 初始化窗口并且设置回调
   - 设置窗口参数`glfwWindowHint`
   - 创建窗口`glfwCreateWindow`
   - 设置当前上下文`glfwMakeContextCurrent`
   - 设置窗口事件回调`glfwSetFramebufferSizeCallback,glfwSetCursorPosCallback,glfwSetScrollCallback`
   - 设置窗口尺寸`glViewPort`(在初始化 glad 后才能调用)
3. 初始化 glad:`gladLoadGLLoader((GLADloadproc)glfwGetProcAddress)`
4. 准备程序
   - 编写顶点着色器，修饰符`in,out,uniform`等，类型`vec*,mat*,sampler2D`等
   - 编写片段着色器
   - 编译着色器`glCreateShader`,`glShaderSource`,`glCompileShader`,`glGetShaderiv`,`glGetShaderInfoLog`
   - 链接`glCreateProgram`,`glAttachShader`,`glLinkProgram`,`glGetProgramiv`,`glGetProgramInfoLog`
   - 使用程序`glUseProgram`
5. 准备数据
   - 模型顶点数据
   - 模型贴图位置数据
   - 模型矩阵$M_{model}$
   - 观察矩阵$M_{view}$
   - 透视投影矩阵$M_{proj}$
   - 最终顶点着色器计算的顶点值应当是$M_{proj}*M_{view}*M_{model}*V_{pointer}$
6. 填入数据
   - 生成顶点缓冲,或者索引缓冲`glGenBuffers`,生成顶点数组`glGenVertexArrays`,生成纹理`glGenTextures`
   - 绑定顶点数组对象`glBindVertexArray`,绑定缓冲`glBindBuffer`,绑定纹理`glBindTexture`,设置纹理属性`glTexParameter*`
   - 获取属性的指针`glGetUniformLocation`,`glGetAttribLocation`
   - 往顶点缓冲填入数据`glBufferData`,向纹理中填入图片`glTexImage2D`,生成多级渐远纹理`glGenerateMinmap`
   - 对`uniform`全局数据用`glUniform*`函数填入
   - 对`vertex attrib`用`glVertexAttribPointer`和`glEnableVertexAttribArray`启用
7. 主循环
   - 查询窗口是否应当关闭`glfwWindowShouldClose`
   - 查询键盘输入`getKeyInput`
   - 绘图
   - 交换双缓冲绘图的缓冲区`glfwSwapBuffers`
   - 轮询事件`glfwPollEvents`
8. 关闭、销毁和中止`glfwSetWindowShouldClose`,`glDeleteShader`,`glDeleteProgram`,`glDeleteBuffers`,`glDeleteVertexArrays`,`glfwTerminate`

# 参考资料

1. [线性代数的本质](https://www.youtube.com/divlaylist?list=PLZHQObOWTQDPD3MizzM2xVFitgF8hE_ab)
2. [learnopengl 中文文档译者四元数 pdf 教程](https://krasjet.github.io/quaternion/)
3. [learnopengl 中文文档译者万向节死锁 pdf 教程](https://krasjet.github.io/quaternion/bonus_gimbal_lock.pdf)
4. [数学科普 youtuber 3Blue1Brown b 站空间](https://space.bilibili.com/88461692#!/channel/detail?cid=9450)
5. [《绕任意轴旋转》](https://www.cnblogs.com/graphics/archive/2012/08/10/2627458.html) authored by [翰墨小生](https://home.cnblogs.com/u/graphics/)@[博客园](https://www.cnblogs.com/)
6. [《绕任意轴旋转的矩阵推导》](https://blog.csdn.net/aa20274270/article/details/50162213) authored by [aa20274270](https://blog.csdn.net/aa20274270)@[csdn](https://www.csdn.net/)
7. [《OpenGL Projection Matrix》](http://www.songho.ca/opengl/gl_projectionmatrix.html)
8. [《opengl – 了解 GLM :: LookAt()》](http://www.voidcn.com/article/p-xdiftyya-bsr.html)
9. [learn opengl:摄像机](https://learnopengl-cn.github.io/01%20Getting%20started/09%20Camera/)
10. [learn opengl:坐标系统](https://learnopengl-cn.github.io/01%20Getting%20started/09%20Camera/)
