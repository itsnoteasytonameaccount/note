---
title: opengl(13):延迟着色法，环境光遮蔽
date: 2021/06/26
tags:
  - opengl
---

为学懂 webgl，从 opengl 入手。本文内容主要来自[LearnOpenGL CN](https://learnopengl-cn.github.io/)文档，加上个人理解，以及一些细节补充、经验记录。本人学习过程中的详细代码参见[此处](https://github.com/itsnoteasytonameaccount/OpenGL-learning)

要学习推荐看原文[LearnOpenGL CN](https://learnopengl-cn.github.io/)，有评论区，加载评论区需要翻墙

另可在[此处](https://www.khronos.org/opengl/wiki/Getting_Started)找到其他 opengl 官方推荐的资料、教程网站

<!-- more -->

# 延迟着色法（Deferred Shading）

在片段着色器中计算光照是开销比较大的一步操作（每个片段需要针对每个光源逐个地计算），我们之前计算光照的方法，叫正向渲染（forward rendering or forward shading），实际上进行了一些不必要的计算：对某些不会在屏幕上显示的片段（如不通过深度测试的片段）也计算了光照。

延迟着色法就是为解决上述问题而产生的方法，实现方式大致为：先得出将会显示在屏幕上的片段，将它们的位置、法线、纹理、镜面光强度等信息存在缓冲中，然后再根据这些信息一次并渲染计算光照。用来存放各种信息的缓冲被称作G缓冲，是一组纹理。

延迟着色法实现步骤大致如下：

1. 创建G缓冲（自定义帧缓冲，绑定多个颜色附件）
2. 第一次渲染，用多目标渲染，将各类信息分别渲染至G缓冲(顶点位置、法线、镜面光向量、纹理)
3. 