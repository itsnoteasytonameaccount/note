---
title: vite使用
date: 2023/04/24 23:39
tags:
- javascript
- vite
---

# 准备

> 首先环境需要 node 14.18+, 16+

# 创建工程

```bash
npm create vite@latest 你的工程名
```

上面方案创建的会让你选模板，不想选模板的话可以直接手动创建

创建工程目录后，进入目录，执行

```bash
npm init
npm i -D vite
```

再在工程目录下创建`index.html`和`vite.config.js`即可

vite会将你的`index.html`视作入口

只要将入口文件在index.html通过script标签引入，并且

# 使用问题

与webpack使用存在区别

1. 无法处理带jsx语法的js文件

https://github.com/vitejs/vite-plugin-react/discussions/72

2. 无法像webpack一样配置`resolve.roots`来使得某个目录下所有文件夹都可以被识别为路径开头

# 参考

1. [Vite 官方中文文档](https://cn.vitejs.dev/guide/)
