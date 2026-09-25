---
title: WebGL 学习笔记(1)
tags:
- typescripts
- vite
- WebGL
date: 2023/05/14
---

# 工程框架

比较简易

vite构建 + typescript + glMatrix'

vite开箱支持typescript

## vscode上使用vite + typescript过程遇到的问题

1. 非ts及js扩展名的文件不被识别，报错

   目前不知道如何处理，只能在声明文件中声明`declare module "*.your_ext"`, 这样不会报错，但是不能自动导入，导入输入部分文件名时也不会自动提示，ctrl点击也不会跳转到对应的文件，而是跳转到module声明

2. 成员变量必须在构造函数初始化

   tsconfig.json中配置` compilerOptions.strict`为false

3. 多态

   typescript的类型转换写法是`ChildClass child = <ChildClass>baseClassInstance;`

4. 类型检查

   vite是不会对你的typescript代码进行检查的，如果需要检查，请安装typescript后用tsc命令

# glMatrix使用

[官方文档](https://glmatrix.net/docs/index.html)

